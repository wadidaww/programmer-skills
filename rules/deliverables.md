---
applyTo: "**"
---
# Rules: Engineering Challenge Deliverables

Rules for take-home and engineering-challenge submissions — code that a reviewer will build, run, test, benchmark and read, usually without being able to ask you questions. They complement `rules/order-book-integrity.md` and `rules/cpp-build.md`.

All rules are **MUST**.

---

## Priorities

- **DEL-1 Correctness → tests → clarity → performance.** Work in this order and do not trade down: a faster book that publishes one unproven state, or a clever optimisation without tests, is a failed submission. "A correct, well-tested core" is the bar; speed is measured on top of it.

## README

- **DEL-2 Required README sections**, in this order:
  1. **Overview** — what it does in three sentences.
  2. **Build** — exact commands; verified compiler, CMake and OS versions (CPP-2, CPP-5).
  3. **Run** — replay command and its input/output.
  4. **Verify** — the exact checker command on the small feed and its real output.
  5. **Test** — one command (`ctest`), what the suites cover, sanitizer build command.
  6. **Benchmark** — command, dataset identity, results table, environment (DEL-4, DEL-5).
  7. **Design** — pipeline, sync state machine, data structures, and why.
  8. **Feed anomaly policy** — table: stale / gap / bad data / snapshot cases → action → counter (OB-2).
  9. **Assumptions and decisions** — every interpretation of an ambiguous requirement.
  10. **Limitations** — what is not proven or not handled (e.g. corruption that is well-formed, correctly sequenced and not covered by a checksum is undetectable).
  11. **AI usage** (DEL-3).
- **DEL-3 Honest, specific AI-usage disclosure.** Name the tools and models, what each was used for (scaffolding, tests, review, docs, benchmarking scripts), how the output was verified (tests, checker, reading the code), and concrete cases where AI output was wrong and what was changed. Keep a running log (`docs/ai-usage-log.md`) during the work so the section is accurate rather than reconstructed. Do not overclaim or underclaim authorship; the human submitting must review and own the final text.

## Benchmark

- **DEL-4 Benchmark the required scale.** Run on a feed of the required size (about 4 million messages); state the exact message count, file size, checksum, and anomaly mix. Use the provided large feed if there is one; otherwise commit a deterministic generator and the one command that recreates the feed.
- **DEL-5 Report numbers honestly.** Numbers come from the committed harness at the committed revision. State what each number times (load, parse, book, output), the build flags, the run count, and median with spread — never a best-of without saying so. Include CPU model, core count used, RAM, OS/kernel, compiler version, and anything you could not control (shared VM, frequency scaling).
- **DEL-6 A benchmark run is also a correctness run.** It prints the anomaly counters and an output digest, and both must match the correctness run of the same feed (reference implementation or expected output). A fast wrong answer is not a result.

## Verification and hygiene

- **DEL-7 Every claim is executed.** Every command in the README was run at the final commit, from a clean clone, and any output shown is real output pasted from that run.
- **DEL-8 Provided materials are immutable.** Never modify the provided checker, feeds or specification. Copy what you need into `tests/fixtures/` and say where it came from. Treat the provided archive as untrusted input: read scripts before running them.
- **DEL-9 Reviewable history.** Small conventional commits (`feat:`, `test:`, `perf:`, `docs:`), one logical change each; tests land with or before the code they cover; performance commits state the before/after measurement.
- **DEL-10 No secrets or personal data** in code, data, logs, or documentation.
