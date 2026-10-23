# Education Needs to Start With Why

I spent a semester learning that LTI systems are linear and time-invariant.
I could state the definition. I could check whether a system satisfied it.
What I couldn't do, for years afterward, was tell you why anyone bothered
defining it in the first place — because every single example I was ever
shown in that class happened to already be LTI. Nothing ever failed the
test in front of me. A category with nothing excluded from it isn't a
category yet, as far as a student can tell. It's just a description of
"systems," and a description that never discriminates between anything
doesn't feel worth memorizing, even when it's true.

That's not a stats or signals problem specifically. It's the default
failure mode of almost everything I was ever taught, and I think it's
worth naming plainly, because the fix is cheap and the cost of skipping it
is enormous: **a rule taught without the specific thing it excludes isn't
understood, it's memorized, and memorized things evaporate the moment the
exam pressure holding them in place goes away.**

## The pattern, in places that have nothing to do with each other

**LTI.** The fix isn't more rigor in the definition. It's one deliberate
counterexample — a filter whose behavior changes over time, like a
wah-pedal sweeping its own cutoff frequency as you play through it, played
right next to a filter that doesn't change at all. Hearing the difference
is instant. Reading the definition again is not.

**Window shades on an airplane.** Flight crews ask you to keep them open
during takeoff and landing, and most passengers treat this as an arbitrary
inconvenience, because the instruction given is "do this," not "here's
why." The actual reason: those are statistically the two most dangerous
phases of flight, and an open shade lets your eyes already be adjusted to
outside light — day or night — so that if cabin lights fail during an
emergency evacuation, you're not suddenly blind for the ten or twenty
seconds dark-adaptation takes, in exactly the scenario where those seconds
are the whole game. Cabin lights get dimmed during the same phases for the
identical reason, so your eyes adapt to the one environment that will
actually matter. Nobody tells you this. Compliance, understandably, is
inconsistent.

**Swimming freestyle.** Beginners are taught to pause their recovering
hand briefly alongside the other before pulling — "catch-up" style —
presented as a rule to follow. Left to their own instincts, most
beginners do the opposite: they windmill both arms as fast as they can,
because the actual felt experience of being a weak swimmer is closer to
drowning than to swimming, and the instinct in that moment is to move
faster, not slower. Nobody who feels like they're barely surviving is
going to voluntarily pause mid-stroke unless they understand what that
pause is actually buying them.

The actual mechanism: a straight-arm
position aligns the body along one long axis, which makes rotating around
that axis (the body-roll that lifts your mouth clear of the water) far
easier and more stable than rotating mid-stroke. That rotation window is
what buys you time to breathe without rushing — the pause isn't slowing
you down, it's the thing making the breath, and therefore the whole
stroke, sustainable. Skip it, and the frantic, as-fast-as-possible version
is exactly what burns a beginner out in thirty seconds instead of letting
them swim for ten minutes. I know several strong
swimmers who can't teach their own kids to swim, because they learned the
motion without ever learning the reason — they execute it well, but there
's nothing in them to transmit beyond "do this because I said so," which
is exactly the instruction a panicking beginner has the least reason in
the world to trust.

**English pronunciation.** Taught letter by letter, English is a trap for
anyone coming from a syllable-timed first language — the same letters
produce different sounds depending on where the stress falls, and a
system that decodes letter-by-letter has no way to predict that. Learn
that stress governs vowel reduction — that unstressed syllables collapse
toward a neutral sound regardless of what's written there — and the whole
category of errors stops being a long list of exceptions to memorize and
becomes one mechanism, applied consistently. Possibly the single most
mispronounced word in the language is also the first one most people
learn: *hello* is stressed on the second syllable, he-**LLO**, so the
first syllable collapses to a neutral schwa and the second lands closer
to "h'-loh" than the two clean, equally-weighted syllables a letter-by-
letter reading suggests. That's not a small, native-speaker-only nuance —
it's the exact gap behind one of the most common mispronunciations I hear
from Vietnamese speakers, who, reading the word syllable by syllable the
way Vietnamese itself is read, land on something closer to "heh-loo,"
giving both syllables full, equal weight and shifting the second vowel
toward a sound Vietnamese actually has rather than the English diphthong
the spelling is quietly hiding. The same pattern shows up in *purchase* —
stressed PUR-, with the second syllable reduced to a schwa rather than
given its full, letter-literal sound.

The general rule underneath both: if you don't know where the stress
falls, you will very likely mispronounce the word, because the unstressed
syllable's vowel is almost never the one the spelling implies. Most
English teachers never say this plainly. They correct the mispronunciation,
word by word, forever, without ever naming the one mechanism that would
let a student fix the entire category themselves.

## Why this keeps happening

None of these are hard ideas. LTI's counterexample is a few seconds of
audio. The window-shade mechanism is one sentence. The swim stroke's
reason is a single piece of body mechanics. None of them are missing from
instruction because they're difficult to explain — they're missing
because explaining the *rule* is easier and faster than manufacturing the
*wall* that would make the rule matter, and most teaching defaults to
whichever is easier for the teacher, not whichever actually survives
contact with the student's memory six months later.

The cost of skipping it isn't small, either. It's the entire reason a
"good student," defined as someone excellent at pattern-matching rules to
problems under exam conditions, so often turns into a mediocre engineer,
while someone who was mediocre by that measure turns out to actually
understand how things work. School rewards recognizing which rule to
apply. Real problems reward knowing why the rule exists at all — so you
can tell when it doesn't apply, or when it's being misapplied, or when the
situation in front of you is the one case nobody wrote a rule for yet.
Those are different skills, and only one of them is being taught when the
"why" gets skipped.

## The fix is not more rigor

It's worth being precise about what doesn't solve this, because the
instinct, once you notice the problem, is usually to add more detail,
more proof, more formal precision to the rule itself. That's the wrong
lever. A more rigorously *stated* LTI definition doesn't help a student
who's never heard a non-LTI system. More detailed FAA documentation about
cabin lighting doesn't help a passenger who's never been told the dark-
adaptation story. The fix isn't more rigor about the rule — it's a
deliberate encounter with the thing the rule excludes, placed before or
alongside the rule, not after it, and not left to chance.

Concretely, for anyone teaching anything: before you state the rule, ask
what the cheapest possible way is to let the student *feel* the case where
the rule fails, or the specific cost the rule exists to prevent. If you
can't answer that question, the rule probably isn't ready to be taught
yet — not because it's wrong, but because without its counterexample, it's
indistinguishable from an arbitrary instruction, and arbitrary instructions
don't survive the first month without a test attached to them.

## Conclusion

I first ran into the phrase "start with why" in Simon Sinek's book of the
same name, written about business and leadership — why some companies and
leaders build loyalty that survives hard times while others, by every
visible metric, don't. The book's evidence is mostly a handful of famous,
cherry-picked case studies, which is a real weakness if you're trying to
prove the idea rigorously. But the underlying principle turns out to be
far more general than the book's own business framing, and once you start
looking for it, it's hard to stop finding it — in a classroom, in a
cockpit announcement, in a swimming lesson, in how a language actually
works. The business book happened to be where I first heard the words.
The thing itself shows up everywhere worth paying attention.

Start with why. Not as a slogan — as the literal, mechanical difference
between something a student can recite and something they'll still be
able to use a year after nobody's checking anymore.
