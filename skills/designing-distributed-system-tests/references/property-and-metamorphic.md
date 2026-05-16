# Property-Based and Metamorphic Testing

**Last verified:** 2026-05-16

## When to reach for it
Use property-based testing when you can state an invariant or relation that should hold across many inputs, even if computing the right answer for any one input is hard. Metamorphic testing is especially powerful for query engines, serializers, encoders, schema systems, and configuration systems where round-trip or equivalence properties exist.

## What it detects well
- Algebraic-law violations: round-trip (encode-decode), commutativity, idempotency, associativity.
- Behavior divergence between two implementations or versions.
- Edge cases hidden by example-based testing: boundary integers, empty/huge inputs, unicode, null values.
- Configuration-space typos and type mismatches.

## What it misses
- Bugs that do not violate the stated invariant.
- Requires good shrinking to be debuggable—if you don't have it, failures are hard to reproduce.
- Complex multipart interactions not captured by simple properties.

## Concrete tools
- `Hypothesis` — Python property-based testing with excellent shrinking — https://hypothesis.works
- `QuickCheck` — Haskell original; the reference implementation — https://hackage.haskell.org/package/QuickCheck
- `PropEr` — Erlang property-based testing, QuickCheck-compatible — https://propertesting.com
- `proptest` — Rust property testing with powerful shrinking — https://github.com/proptest-rs/proptest
- `fast-check` — TypeScript/JavaScript property testing — https://github.com/dubzzz/fast-check
- `ScalaCheck` — Scala QuickCheck port — https://scalacheck.org

## Papers to cite
- "Metamorphic Testing: A Review of Challenges and Opportunities" (Chen et al., 2017) — comprehensive survey of metamorphic-relation patterns — https://www.cs.hku.hk/data/techreps/document/TR-2017-04.pdf
- "Metamorphic Testing" (Hillel Wayne) — pragmatic introduction with concrete examples — https://www.hillelwayne.com/post/metamorphic-testing/

## Cost / wall-clock signal
Per-property execution is seconds to minutes; design cost is mostly in stating correct and effective properties that actually find bugs.

## How a plan typically uses it
1. List the algebraic laws and metamorphic relations the system-under-test must obey.
2. Colocate the properties next to the code they test for maintainability.
3. Keep generators tight: boundary integers, empty/huge collections, unicode, null—narrow the input space to high-leverage regions.
4. Require shrinking on failure so that failures are reproducible and minimal.
5. For configuration systems, treat the config space itself as the input domain; generate configs as aggressively as inputs.
