---
applyTo: "**/*.{cpp,cc,cxx,hpp,hh,h,py}"
---
# Rules: Order Book Integrity

Non-negotiable rules for any code that reconstructs an order book from a market-data depth feed. Every rule serves one guarantee:

> **The book never publishes a state it cannot prove is correct.**

*Provably correct* means the published state was built from a validated snapshot plus an unbroken, fully validated, exactly-once, in-order chain of updates, and it agrees with every integrity signal the feed carries (sequence numbers, checksums, snapshots). When in doubt, withhold: a withheld state costs availability, a wrong state costs money.

All rules are **MUST**. A violation is a blocking defect regardless of test results or benchmark gains. Cite rule IDs in test names and review findings (`agents/order-book-auditor.md`).

How-to: `skills/order-book/feed-integrity.md` (sync and proof), `skills/order-book/reconstruction.md` (data structures).

---

## Specification

- **OB-1 The checker is the spec.** Read the provided checker script and any format notes completely before writing code. Input grammar, output format, how an unknown/invalid book is expressed, numeric comparison (exact string, decimal, or float tolerance) and output cadence (per message, per snapshot, per change) all come from it. Where the checker is silent, choose the conservative behaviour.
- **OB-2 Document every interpretation.** Each ambiguity — what counts as stale, how a gap is detected, which lines are bad data, what is emitted while invalid, whether locked books are allowed — gets one line in the README: decision + rationale.

## Publishing

- **OB-3 One publish gate.** Every output that exposes book state goes through a single path that requires the synced state. No debug print, log line, or benchmark shortcut emits book contents around it.
- **OB-4 Publish only when synced.** Before the first valid snapshot, after any gap, after any lost update, and after any failed post-condition the book is *unsynced*: emit nothing, or the checker's explicit invalid marker — never the last known book ("hold last value" is forbidden).
- **OB-5 Post-conditions before publish.** After a complete message is applied and before anything is published: levels sorted and unique per side; every quantity > 0; best bid < best ask when both sides are non-empty (locked or crossed ⇒ unsynced unless the spec explicitly allows it); feed checksum matches when one is provided.

## Sequencing

- **OB-6 Stale is a no-op.** A message at or below the last applied sequence is dropped and counted. It never mutates the book, never re-applies, never resynchronises, never invalidates.
- **OB-7 A gap invalidates immediately.** A sequence discontinuity moves the book to unsynced before anything derived from post-gap data can be published. No interpolation, no "probably fine", no resync from deltas alone.
- **OB-8 A lost update is a gap.** If the next expected message (correct sequence) fails validation, its update is lost: invalidate. A line whose sequence cannot be identified is rejected without changing state; the sequence check on the next valid message decides whether anything was lost. Never advance the last-applied sequence past a message you did not apply.
- **OB-9 Recover only from proof.** Leave unsynced only by adopting a snapshot that passed validation (sorted, unique, positive, not crossed, sequence parsed, not older than the newest validated sequence already seen), then continue under the feed's documented bridging rule. Updates at or below the snapshot sequence are discarded.
- **OB-10 Overlap only with idempotent updates.** Re-applying an update that overlaps the snapshot (range-sequenced feeds) is allowed only when updates set absolute level quantities. Incremental (+/−) updates are never applied twice.
- **OB-11 Reorder only if specified.** Do not buffer or reorder out-of-order messages unless the spec requires it. If it does, the window is bounded and documented, nothing past a hole is applied, and an unfilled hole at window expiry invalidates.

## Validation and mutation

- **OB-12 Validate, then mutate.** A message is fully parsed and validated before the first book mutation. A multi-level message is applied all-or-nothing; a partially applied message never exists in a synced book.
- **OB-13 Reject, never repair.** Do not clamp negative quantities, round off-grid prices, swap crossed sides, drop "bad" levels and keep the rest, or default missing fields. Invalid input is rejected and counted; its effect on sync follows OB-8.
- **OB-14 Unknown references are desync evidence.** Deleting a price level (L2) or cancelling/modifying an order id (L3) that is not in the book means the book or the message is wrong: invalidate — unless the spec documents it as normal venue behaviour (OB-2).
- **OB-15 Exact arithmetic.** Prices and quantities are fixed-point integers (or exact decimals). Never use `float`/`double` as a map key, never compare prices with floating-point `==`, never round-trip prices through `double`. Overflow is detected and rejected, never wrapped.
- **OB-16 Check per message, not per level.** Post-conditions are evaluated after the whole message is applied. A transient cross inside one atomic message is not a violation; a cross that survives the message is.

## Observability and determinism

- **OB-17 Count every anomaly.** Keep counters for at least: lines read, applied, stale, gaps, lost updates (by reject reason), post-condition failures, snapshots adopted / rejected / stale / mismatched, malformed lines, messages published, messages withheld. Print them at the end of every run, including benchmark runs.
- **OB-18 Deterministic output.** Same input ⇒ byte-identical output and counters, in Debug, Release and sanitizer builds. No dependence on wall-clock time, unordered-container iteration order, thread scheduling, or uninitialised memory.
- **OB-19 Hostile input never causes undefined behaviour.** Every byte of the feed is untrusted: bounds-checked parsing, checked integer conversion, no exceptions escaping the hot path, clean under ASan + UBSan on the full test suite and a fuzzing run.
