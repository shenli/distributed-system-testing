# Fuzzing (input and concurrency)

**Last verified:** 2026-05-16

## When to reach for it
Use fuzzing on parsers, state machines, RPC servers, and anywhere untrusted or arbitrary bytes enter the system. Also apply randomized concurrency fuzzing to uncover scheduling and message-order bugs in lock-free code or async runtime layers.

## What it detects well
- Memory unsafety (segfaults, use-after-free, buffer overflow when running with ASan/UBSan/MSan).
- Parser crashes and panics on malformed input.
- State-machine invariant violations and integer overflow.
- Infinite loops triggered by malformed input.
- Message-order bugs in concurrent systems (via interleaving-space fuzzing).
- Regex denial-of-service amplifiers and pathological inputs.

## What it misses
- High-level correctness (fuzzing stops at "did not crash"; you need an oracle for right-answer-ness).
- Bugs requiring multi-input setup or stateful interaction patterns.
- Performance regressions and resource leaks detectable only under normal operation.

## Concrete tools
- `libFuzzer` — in-process coverage-guided fuzzer on LLVM — https://llvm.org/docs/LibFuzzer.html
- `AFL` / `AFL++` — coverage-guided binary fuzzer with excellent corpus feedback — https://github.com/AFLplusplus/AFLplusplus
- `cargo-fuzz` — libFuzzer wrapper for Rust projects — https://github.com/rust-fuzz/cargo-fuzz
- `go test -fuzz` — Go's native fuzzing support (Go 1.18+) — https://go.dev/security/fuzz/
- `honggfuzz` — feedback-driven fuzzer with hardware-assist support — https://github.com/google/honggfuzz
- `FlyMC` — research tool for scalable distributed-system interleaving fuzzing — https://dl.acm.org/doi/10.1145/3302424.3303986

## Papers to cite
- "FlyMC: Highly Scalable Testing of Complex Interleavings in Distributed Systems" (Lukman et al., EuroSys'19) — distributed-systems interleaving fuzzer for scheduling bugs — https://dl.acm.org/doi/pdf/10.1145/3302424.3303986
- "Combining AFL and QuickCheck for Directed Fuzzing" (Dan Luu) — practical fuzzing workflow — https://danluu.com/testing/

## Cost / wall-clock signal
CPU-hours to CPU-weeks depending on input complexity and coverage depth; well suited to continuous fuzzing farms and integration into CI.

## How a plan typically uses it
1. Identify each input boundary: external bytes, untrusted RPC messages, arbitrary call sequences.
2. Write a fuzz target that hits the smallest meaningful entry point (parser, state machine step, handler).
3. Run under sanitizers (ASan, UBSan, MSan)—fuzzing without them only catches crashes, missing memory safety.
4. Seed the corpus from real samples, existing tests, or protocol examples.
5. Set an exit criterion (coverage plateau or ops count) rather than unbounded wall-clock time.
