![system_rebuild](https://github.com/user-attachments/assets/3edfbb33-2e79-4052-b0c2-2dd733a87d9e)

# In-Memory Query System Rebuild

## Overview

I served as co-tech lead on a year-long project to modernize and rebuild the in-memory query system. The project aimed to accelerate future development velocity by replacing legacy C++ components with idiomatic Go, improving tooling, debugging capabilities, and horizontal scalability. Three engineers from my sub-team supported the effort throughout the project.

## Decision to Rebuild

### The Problem

The original system was built in C++ with a custom WebSocket communication layer. While the performance-critical storage and computation components justified C++, the query processing and routing components did not. These components were single-threaded, making horizontal scaling difficult, and the custom WebSocket layer created significant friction for development and debugging. As the system grew to serve more customers at greater scale, the architectural limitations became a meaningful bottleneck on development velocity.

Specifically:

- Adding new query types required navigating a codebase where tooling support was limited and debugging distributed timing issues was painful
- The event-driven single-threaded C++ model is notoriously difficult for engineers without prior experience in that paradigm, limiting who on the team could contribute effectively and making onboarding new engineers onto the codebase slow
- The single-threaded model meant scaling required adding more nodes rather than utilizing available CPU capacity on existing nodes
- The custom WebSocket layer lacked the ecosystem support that a standard protocol would provide, making interoperability with new clients harder over time
- Caching behavior was inconsistent across query types: normal aggregate count queries were rewritten for intelligent caching while structurally identical histogram queries were not, resulting in unnecessary backend load for a common query type

### Alternatives Considered

**Incremental refactoring of the existing C++ codebase.** This would have avoided the risk of a large rewrite but would not have addressed the threading model or the WebSocket layer. The tooling and debugging limitations would have persisted. Incremental improvements to a fundamentally constrained architecture tend to produce local optima rather than resolving the underlying constraints.

**Wrapping existing components.** Introducing a thin wrapper around the C++ components would have preserved the existing behavior while exposing a cleaner interface. However, this would not have unlocked multi-threading, would not have improved the development experience inside the components, and would have added an abstraction layer without removing the underlying complexity.

### Why a Rebuild Was the Right Decision

The query processing and routing components — frontend, aggregator, and translation layer — were not the performance-critical parts of the system. The storage and computation happened in the backends. Rewriting the non-performance-critical components in Go therefore carried low performance risk while delivering meaningful gains in developer experience, tooling, and scalability.

gRPC provided a well-supported, language-agnostic communication protocol with built-in support for streaming, deadlines, and code generation. Replacing the custom WebSocket layer with gRPC reduced the surface area of custom infrastructure the team needed to maintain and made adding new clients straightforward. A concrete example: implementing TLS on the custom WebSocket layer was prohibitively difficult given how deeply custom it was. With gRPC, TLS support came for free.

Go's native goroutine model meant the rewritten components could handle concurrent requests without the complexity of manual thread management, unlocking horizontal scaling within a node in addition to across nodes.

The investment was justified by the expected return: a system that could support the team's growing customer base and query volume without requiring proportional increases in engineering effort to add new capabilities.

### Risk Mitigation

A year-long rewrite of production infrastructure carries real risk. We mitigated this through:

- **Shadow testing**: Running 100% of production queries against the rebuilt system in parallel before cutover, comparing outputs using a similarity score to account for non-determinism
- **Extensive integration testing**: Building thorough test coverage for complex distributed timing scenarios before rollout, not after
- **Gradual rollout**: Validating against the largest internal cluster before exposing to external customers

The outcome validated the approach: only two crashes across the entire production rollout, both with clear root causes and straightforward fixes.

## System Architecture

The rebuilt system handles point queries (retrieve a specific trace by trace ID), list queries, aggregate queries, and time series aggregate queries. The diagram below shows the overall component topology.

I led the rewrite of the following components:

### Frontend (Query Distributor)

The frontend is the central query processing layer. It receives incoming queries, determines routing based on the sharding scheme, and handles result aggregation before returning responses to clients. Key capabilities include:

- **Query rewriting**: Transforms queries for optimal backend execution
- **Time series query caching**: Reduces redundant computation for repeated time series queries
- **Query deduplication**: Coalesces identical in-flight queries into a single backend request
- **Query approximation**: Routes a query to a representative subset of backends and interpolates the final result, reducing backend load for approximate use cases
- **Distributed query throttling**: Coordinates rate limiting across multiple parallel frontend instances that may have different views of backend load
- **Query batching**: Groups queries sharing the same filter for efficient batch execution
- **Timeout handling**: Accounts for delayed responses from overloaded backends
- **Complex aggregation**: Supports top-k, histogram, unique counts, and other advanced aggregation types

### Aggregator

As the system scales to thousands of backends, the fan-in from a single frontend node becomes prohibitively high. Without an intermediate layer, frontends would be overwhelmed by responses from thousands of concurrent backend connections. The aggregator tier sits between frontends and backends, merging results from a subset of backends before forwarding consolidated responses upstream. This architecture enables the system to scale to clusters of 6,000+ backend nodes while keeping frontend fan-in manageable.

### REST to gRPC Translation Layer

To ensure a seamless transition for clients that had not yet adopted the new gRPC interface, a translation layer was introduced to handle all legacy REST queries. This allowed existing integrations to continue operating without modification while the ecosystem migrated to gRPC at its own pace.

## Correctness and Performance Validation

### Correctness

Correctness validation relied on two complementary mechanisms. First, an extensive integration test suite covered functional correctness as well as complex interaction patterns specific to distributed systems. Second, shadow queries were run against the largest internal cluster to compare outputs between the rebuilt system and the original. Because query outputs are not fully deterministic, comparisons were made using a similarity score rather than exact equality.

### Performance

For realistic performance validation, 100% of queries from the largest production cluster were shadowed to the rebuilt system. This ensured that performance characteristics were evaluated under real traffic patterns rather than synthetic benchmarks.

## Reliability in Production

The rebuild delivered measurably against its original thesis. Median query latency improved by 20%, driven in part by the consistent caching now applied across all query types including histogram aggregations. What I am most proud of, however, is not the performance improvement alone but the exceptional reliability of the rollout itself. Across the entire production deployment, there were only two crashes:

- A nil callback pointer that was not passed through correctly in the complex timeout handling logic
- A contract violation between frontends and backends: the frontend assumed all items in a list query would have keys of the same type to enable sorting, but a backend returned a list with both string and double keys

This low defect rate was the direct result of heavy upfront investment in testing. Writing correct distributed systems code is difficult; writing it in a way that enables thorough unit and integration tests covering complex timing scenarios is significantly harder. That investment paid off materially during validation and rollout.

## Natural Extension: On-Disk Storage

A natural extension of this architecture is to replace the in-memory storage nodes with on-disk storage. ClickHouse is a strong candidate: by introducing a ClickHouse adapter that translates gRPC requests into ClickHouse query language, the system could function as a fully capable analytics and monitoring platform for logs and traces at a fraction of the storage cost.

One architectural consideration with this approach is the tight coupling between storage and compute that ClickHouse introduces. This makes the system less efficient under spiky traffic since the configuration is largely static. For teams requiring true compute-storage separation at scale, a high-performance control layer capable of handling large volumes of metadata reads and writes would be needed to manage dynamic routing between compute and storage tiers.
