# Compute Once, Fan Out: The Universal Optimization Hiding in Every System

LLMs have been tremendously useful to me since I discovered them. As an engineer, making things efficient is one of my greatest passions. LLMs are generally used for unstructured data, making it inefficient to generate structured data. We can exploit the predictability of structured data in many cases to improve performance — not only in the LLM space but also across other areas of computer science.

SGLang's RadixAttention achieves up to 5x higher throughput for LLM serving by doing something conceptually simple: when multiple requests share a common prompt prefix, compute the KV cache for that prefix once and reuse it across all of them. The paper presents this as a novel contribution to LLM inference. And it is. But the underlying principle is not new. The same optimization appears in CPU microarchitecture, database query execution, and observability systems. Once you see the pattern, you start finding it everywhere.

## The Pattern

The general form is this: a system receives multiple concurrent requests. Each request is expensive to serve independently. But the requests share a common subproblem — a prefix, a filter condition, a branch condition — that accounts for most of the work. The naive approach executes the subproblem once per request. The optimized approach detects the overlap, executes the subproblem once, and fans out the results.

The conditions that make this worthwhile are specific:

- The shared subproblem must be expensive relative to the per-request work that follows
- Detecting overlap must be cheap relative to the cost of redundant execution
- The overlap rate in practice must be high enough to justify the detection overhead
- The fan-out step — applying per-request projections to the shared result — must be cheap

When all four conditions hold, the optimization is almost always worth implementing. In practice they hold more often than engineers expect.

## LLM Serving: RadixAttention

In LLM inference, generating a response requires computing a key-value (KV) cache for the input tokens. This computation scales with the number of tokens — a 500-token system prompt takes meaningful GPU time to process. If ten thousand requests per second all begin with the same system prompt, the naive approach computes that KV cache ten thousand times per second.

RadixAttention stores KV caches in a prefix tree. When a new request arrives, the runtime walks the tree to find the longest matching cached prefix, computes only the remaining tokens, and extends the tree. Multiple requests sharing the same system prompt pay the KV cache computation cost once, not once per request.

The overlap rate is high in practice because real deployments have structure: every request to a given chatbot shares the same system prompt; every RAG request over the same retrieved document shares the document context; every few-shot classification request shares the same examples. The system prompt alone can be hundreds of tokens. Caching it gives back most of the computation cost on the shared prefix.

The eviction policy is where the interesting engineering lives. The prefix tree grows indefinitely under load. Long unique conversation histories consume GPU memory without meaningful reuse probability. Short shared system prompts have very high reuse probability. A naive LRU policy treats these identically — recently used long unique context gets retained over older but frequently reused short shared prefix. The right eviction policy weights by sharing factor and expected reuse, not just recency.

## CPU Architecture: Speculative Execution

CPU branch prediction is the same optimization applied to instruction execution. When the processor encounters a conditional branch, it predicts which path will be taken and begins executing speculatively down that path. If the prediction is correct, the work is already done when the branch resolves — no pipeline stall. If wrong, the speculative work is discarded and the correct path executes from scratch.

The shared subproblem here is instruction fetch and decode. The expensive work is execution. The detection mechanism is the branch predictor — a hardware structure that learns from past branch behavior. Modern branch predictors achieve 95%+ accuracy on typical workloads, meaning the pipeline stall cost is rare enough that speculative execution wins decisively.

The same principle extends to memory prefetching (predict which cache lines will be needed and load them before they're requested), out-of-order execution (identify independent instructions that can execute in parallel rather than sequentially), and return address prediction (predict where a function will return before the function executes).

In each case: cheap prediction, expensive shared work, cheap fan-out to the actual result. The pattern is identical.

## HPC and Linear Algebra: Communication-Avoiding Algorithms

The same principle has deep roots in high-performance computing and numerical linear algebra — predating GPU computing by decades. The canonical insight, well-established in the parallel computing community by the time I was in graduate school, is that FLOPs are cheap and communication is the bottleneck.

In distributed matrix computations across a cluster, every inter-node message carries overhead orders of magnitude larger than the arithmetic it enables. The naive approach — compute a partial result, communicate it, compute the next partial result — pays this communication cost repeatedly. Communication-avoiding algorithms restructure the computation to minimize data movement, performing more local arithmetic rather than communicating intermediate results.

James Demmel's group at UC Berkeley proved theoretical lower bounds on the communication required for matrix operations — the minimum number of memory reads/writes or messages any algorithm must perform — and then designed algorithms that achieve those bounds. For dense linear algebra operations, they showed that the naive implementations were far from optimal on communication, and that restructuring the computation to exploit memory hierarchy could bring dramatic improvements.

The mechanism is tiling: partition the computation into blocks that fit in fast memory (cache or on-chip SRAM), operate entirely within fast memory on each block, and write back only the final result. The expensive operation — moving data between memory levels — executes once per tile rather than once per arithmetic operation.

FlashAttention is this principle applied to GPU memory hierarchy for attention computation. The theoretical insight — that attention is memory-bandwidth bound rather than compute-bound, and that tiling can reduce HBM reads/writes from O(N²) to O(N) — follows directly from the framework Demmel's group developed for matrix multiplication. The attention matrix is never materialized in HBM. Computation stays in fast on-chip SRAM. Data movement is minimized, not arithmetic.

The connection goes further: FlashAttention has been shown to achieve the IO complexity lower bound for attention computation — a result that would have been unsurprising to numerical linear algebra researchers familiar with Demmel's lower bound proofs. The surprise was not the result but how long it took for this framework to reach the attention computation community, which arrived at GPU programming from a machine learning background rather than an HPC background.

The same tiling principle appears in systolic array design — processor arrays where data flows through in a regular pattern, each element reusing values from neighbors rather than fetching from external memory. Systolic arrays minimize off-chip communication by maximizing data reuse within the array. The hardware and the algorithm are co-designed around the same constraint: communication is expensive, compute is cheap, so recompute rather than communicate.

This lineage matters because it reveals that FlashAttention and RadixAttention are not isolated insights. They are applications of a principle that the HPC community developed rigorously over decades: identify the communication bottleneck, restructure computation to minimize data movement, accept additional arithmetic as the price of reduced communication. The principle applies at every level of the memory hierarchy — registers, L1/L2/L3 cache, DRAM, NVMe, network — and at every scale from a single CPU to a thousand-node cluster.

## Databases: Common Subexpression Elimination

Database query optimizers have applied this principle for decades under the name common subexpression elimination (CSE). When a query contains the same subexpression evaluated multiple times — or when multiple concurrent queries share a common scan — a good optimizer executes the shared work once.

A simple example: a dashboard that fires five queries simultaneously, all with the same WHERE condition but different SELECT projections:

```sql
SELECT avg(duration) FROM spans WHERE service = 'checkout' AND env = 'prod'
SELECT count(*)      FROM spans WHERE service = 'checkout' AND env = 'prod'
SELECT p99(duration) FROM spans WHERE service = 'checkout' AND env = 'prod'
SELECT max(duration) FROM spans WHERE service = 'checkout' AND env = 'prod'
SELECT min(duration) FROM spans WHERE service = 'checkout' AND env = 'prod'
```

The naive approach scans the spans table five times, applying the same filter condition each time. The optimized approach scans once, materializes the matching rows, and computes all five aggregations over the same result set. The filter traversal — which is the expensive part — executes once.

Multi-Query Optimization (MQO) generalizes this further: given a batch of queries, identify shared scans, joins, and filter conditions, and build a combined execution plan that minimizes redundant work across the entire batch. The challenge is detection — recognizing that two queries with structurally equivalent but syntactically different filter conditions share the same subproblem requires normalization and equivalence checking before execution.

## Observability Systems: Dashboard Filter Coalescing

The database pattern appears directly in observability systems. Modern observability UIs load dashboards by firing multiple queries simultaneously — one per widget. A typical dashboard might have ten widgets: an error rate chart, a latency histogram, a throughput graph, a top-k endpoints list, a log stream, a trace list. If all widgets are scoped to the same service and environment, all ten queries share the same filter condition.

In an in-memory observability backend — the kind that serves real-time log and trace queries with subsecond latency — the filter traversal is the expensive operation. The backend maintains in-memory data structures over recently ingested events. Evaluating a filter condition requires traversing those structures to find matching events. Ten queries with the same filter means ten traversals of the same data structure over the same data.

The optimization is the same as database MQO: detect that multiple in-flight queries share a filter condition, execute the traversal once, apply each query's distinct projection (aggregation, sampling, sorting) to the shared result. The filter condition is normalized for equivalence comparison. Queries arriving within a short batching window — milliseconds — are grouped before execution.

The impact is largest during incidents, which is precisely when it matters most. When a service degrades, multiple engineers simultaneously open dashboards scoped to the same service. Without coalescing, each dashboard load multiplies the query load on the backend — exactly when the backend is already under stress and engineers need fast responses most urgently. With coalescing, ten engineers loading the same dashboard generates roughly the same backend load as one.

The conditions for effectiveness hold strongly in this context: filter traversal is expensive (in-memory scan over millions of recent events), overlap rate is high (engineers debugging the same incident scope to the same service), and projection is cheap (aggregation over the already-filtered result set is fast).

## The Unifying Principle

Across CPU speculation, communication-avoiding algorithms, database CSE, RadixAttention, and dashboard filter coalescing, the structure is identical:

```
Shared subproblem:  expensive, high overlap rate in practice
Detection:          cheap relative to redundant execution cost
Fan-out:            cheap per-request projection over shared result
```

The optimization wins when detection cost < (overlap_rate × subproblem_cost). In systems with high structure — LLM deployments with system prompts, observability dashboards with shared filter scope, CPU workloads with predictable branch behavior, distributed matrix computations with regular tiling patterns — the overlap rate is high enough that the optimization is almost always justified.

The failure mode is detection cost exceeding savings. Comparing arbitrary query filters for semantic equivalence is expensive if done naively — a general SQL equivalence checker approaches the complexity of a full query optimizer. The practical solution is normalization to a canonical form (sort filter predicates, resolve aliases, expand shortcuts) followed by string or hash comparison. This makes detection O(filter_length) rather than O(exponential), cheap enough to apply to every incoming query.

The deeper pattern across all these domains is the same constraint: moving data is expensive, computing is cheap. Accept additional arithmetic to reduce data movement. This is true whether "data movement" means inter-node messages in an MPI cluster, reads from GPU HBM, cache misses on a CPU, or redundant filter traversals in an observability backend. The specific numbers differ by orders of magnitude — network latency vs memory bandwidth vs cache access time — but the optimization strategy is identical: identify the data movement bottleneck, restructure computation to minimize it, accept more arithmetic as the price.

## An Extension: Probabilistic Speculation

The FSM approach in SGLang for structured output generation is a deterministic version of this pattern: at positions where only one token is valid given the grammar, skip the model forward pass and fill the token directly. The FSM determines the next token with certainty.

A natural extension is probabilistic speculation — use a learned Markov chain over grammar state transitions as a draft model for speculative decoding. At structural positions where one token is overwhelmingly likely (the colon after a JSON key, the closing quote after a string value), the Markov chain predicts with near-certainty. The LLM verifies in its forward pass — already running regardless — and accepts at high rate. For positions where the Markov chain is uncertain, fall back to the LLM's distribution normally.

This is the same three-tier pattern as CPU microarchitecture: deterministic cases handled by the FSM (branch with known outcome), high-confidence cases handled by the Markov chain (branch predictor), uncertain cases handled by the full model (actual execution). Each tier handles what it can cheaply, passing uncertainty to the next tier.

The Markov chain is trivially trained from any corpus of valid structured outputs — pure frequency counting, no GPU training required. It works for any grammar: JSON, SQL, Python, function call schemas. The acceptance rate at structural positions approaches the FSM's 100% while covering the probabilistic cases that a pure FSM cannot handle.

## Where to Look Next

The pattern appears wherever concurrent requests have structure. A few places worth examining:

**Vector search with metadata filters.** Multiple similarity queries over the same vector index with the same metadata filter — execute the filter once over the index, then run each similarity computation over the filtered candidate set. The filter traversal is the expensive part; the ANN computation over a filtered subset is cheaper than over the full index.

**Inference serving with shared context.** Agent workflows where multiple tool calls or generation steps share the same system context and tool definitions. Compute the KV cache for the shared context once; reuse across all steps.

**Streaming aggregations.** Multiple subscribers to the same event stream with different aggregation windows but overlapping filter conditions. Evaluate the filter once per event; fan out to each subscriber's aggregation state.

In each case, the question is the same: what subproblem do concurrent requests share, how expensive is it, how cheap is detection, and how high is the overlap rate in practice? When the math works — and it does more often than expected — the optimization is almost always worth building.

The systems that win at scale are the ones that avoid redundant work. Not through clever algorithms invented from scratch, but through the straightforward discipline of noticing when the same expensive thing is being done twice and doing it once instead. This principle has been formalized in the HPC community for decades — Demmel's communication lower bounds, systolic array design, cache-oblivious algorithms. It keeps reappearing in new domains because the constraint is physical: moving data costs more than computing. That won't change. The optimization always applies.
