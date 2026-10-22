# Skill: Order Book Testing

How to *demonstrate* — not just assert — that an order book never publishes a state it cannot prove, and that it doesn't withhold states it could have proven. General testing practice is in `skills/testing/testing.md`; this skill is specific to feed handlers and books.

Rules: `rules/order-book-integrity.md`. Sync logic under test: `skills/order-book/feed-integrity.md`.

---

## Test Layers

| Layer | Proves | Size | Runtime |
|---|---|---|---|
| Unit | Number parsing, line framing, `BookSide` operations, validators, each transition-table row | hundreds | ms |
| Scenario | One tiny hand-written feed per rule and per anomaly | 35+ | ms |
| Golden | The provided small feed passes the provided checker | 1+ | s |
| Ground truth | Every published state equals the true book, over millions of random messages with injected faults | seeds × messages | s |
| Differential | Optimised implementation == reference implementation, message by message | seeds × messages | s |
| Fuzz | Arbitrary bytes never crash, never trigger UB, never publish a state violating post-conditions | minutes | min |
| Sanitizers | All of the above under ASan + UBSan | — | 2–3× |

All of it runs from one `ctest` (CPP-12). Fuzzing runs on demand, its crash reproducers become scenario tests.

**Test names cite rules**: `TEST_CASE("OB-7: a gap withholds the book until the next valid snapshot")`. A rule without a test is an unverified claim; an auditor will look for exactly this.

---

## Scenario Catalogue (minimum set)

Each scenario is a feed of a handful of lines plus the exact expected published sequence numbers *and* counters.

**Startup**
1. Deltas before the first snapshot → nothing published, `ignored_unsynced` counted.
2. Valid first snapshot → published exactly as given.
3. Invalid first snapshot (crossed, locked, duplicate price, qty 0, unsorted, bad number) → nothing published; the next valid snapshot syncs.
4. Snapshot with an empty side → behaviour per spec, documented, tested.

**Sequencing**
5. Contiguous deltas → each published.
6. Exact duplicate → second copy dropped, book unchanged.
7. Older message (`seq < last`) → dropped, book unchanged.
8. Gap of one → withheld from that message on.
9. Gap immediately after a snapshot (`snap 10`, then `12`).
10. Gap, then valid newer snapshot → resynced; the next contiguous delta is published.
11. Gap, then a snapshot older than the newest validated sequence → still withheld.
12. Gap, then a long run of contiguous deltas with no snapshot → all withheld (no resync from deltas).
13. Swapped pair (`102` before `101`) → per spec: invalidate, or bounded reorder.
14. Sequence edge values: `0`, maximum `uint64`, non-numeric, negative.
15. Synced, older snapshot arrives → ignored, book unchanged.
16. Synced, snapshot at `seq == last` — identical → no change; different → counted and adopted.
17. Synced, newer snapshot → book replaced.

**Bad data**
18. Garbled line between two contiguous messages → book still published (no sequence was lost).
19. Garbled line that carried `last + 1` (next valid is `last + 2`) → withheld.
20. Next-expected delta with negative quantity → withheld (`lost_update`).
21. … with zero or negative price.
22. … with each bad number form: `1e3`, `nan`, `inf`, `+1`, ` 1`, `1.`, `.5`, `1..2`, overflow, too many decimals.
23. … with unknown side; with unknown message type.
24. … with an off-grid price.
25. Stale message with garbage content → dropped, no invalidation.
26. Multi-level message with one bad level → no level applied, withheld.
27. Delete of a non-existent level → per policy (OB-14).
28. Delta that leaves the book crossed → withheld.
29. Delta that leaves the book locked → withheld unless the spec allows locks.
30. Message that crosses transiently but is uncrossed at its end → published (OB-16).
31. Checksum mismatch → withheld.
32. Duplicate price inside one message → per spec.

**Framing and output**
33. CRLF line endings, missing final newline, empty lines, UTF-8 BOM, trailing delimiter.
34. Every withheld state is emitted in the checker's invalid form (or omitted, if that's the contract).
35. End-of-run counters equal the expected values exactly, and are conserved (outcome counters sum to lines read).

## Scenario Tests Exercise the Real Pipeline

Build a `replay(lines)` helper that runs the production parser → sequencer → publisher into an in-memory sink. Scenario tests then cover the same code paths as the binary, not a test-only shortcut.

```cpp
// The line grammar below is illustrative — use the real one from the feed spec.
TEST_CASE("OB-8: a garbled next-expected update withholds the book") {
    const auto r = replay({
        "S,10,B,100.0,5,A,101.0,5",
        "D,11,B,100.0,-1",          // negative quantity: update 11 is lost
        "D,12,B,99.5,3",
    });
    CHECK(r.published_seqs == std::vector<std::uint64_t>{10});
    CHECK(r.stats.lost_update_total() == 1);
    CHECK(r.stats.ignored_unsynced == 1);
}
```

---

## Ground-Truth Generator — the Most Valuable Test

Hand-written scenarios check what you thought of. A generator that knows the true book checks what you didn't.

```
truth = random valid book around a mid price
emit snapshot(truth, seq 0)
for seq in 1..N:
    update = random absolute update that keeps truth sorted, positive, uncrossed
    truth.apply(update); remember digest(truth) at seq
    transmit(update, seq) through a fault layer:
        drop       (p_gap)   → nothing emitted                ⇒ consumer must invalidate
        duplicate  (p_dup)   → emitted twice                  ⇒ second copy is stale
        delay k    (p_late)  → emitted k messages later       ⇒ gap now, stale later
        corrupt    (p_bad)   → one field broken               ⇒ lost update
        garbage    (p_junk)  → extra junk line, no sequence   ⇒ harmless
    every K messages, and sometimes right after a fault: emit snapshot(truth, seq)
```

Properties to assert on every run:

- **Safety (must be exact)** — every state the consumer publishes at sequence `s` has `digest == truth_digest[s]`. Compare the full book, not only the top level; bugs hide in deeper levels.
- **Liveness** — the consumer is synced exactly when the generator's model of the policy says the chain is provable. This catches over-withholding, which a safety check alone rewards.
- **Conservation** — counters match the faults the generator injected (gaps, duplicates, corrupt lines) and sum to lines read.
- **Determinism** — two runs give byte-identical output.

Use a fully specified PRNG. `std::mt19937_64` is portable, but `std::uniform_int_distribution` and friends are implementation-defined — libstdc++ and libc++ produce different feeds from the same seed.

```cpp
// Portable PRNG: same sequence on every compiler and standard library.
struct SplitMix64 {
    std::uint64_t state;
    std::uint64_t next() {
        std::uint64_t z = (state += 0x9E3779B97F4A7C15ULL);
        z = (z ^ (z >> 30)) * 0xBF58476D1CE4E5B9ULL;
        z = (z ^ (z >> 27)) * 0x94D049BB133111EBULL;
        return z ^ (z >> 31);
    }
    std::uint64_t below(std::uint64_t n) { return next() % n; }                 // bias is irrelevant for test data
    bool chance(double p) { return static_cast<double>(next() >> 11) * 0x1.0p-53 < p; }
};
```

Run small (200 seeds × 20 000 messages) in `ctest`; run the ~4M-message benchmark feed through the same checks once per release (`skills/order-book/benchmarking.md`). The same generator produces the benchmark feed, so the benchmark is also a correctness run (DEL-6).

## Differential Testing

Keep the simple implementation (`std::map` book, straightforward sequencer) as the reference. For every generated feed, feed both and compare after every message: outcome, synced flag, and full-book digest. Every optimisation lands only with this test green.

## Golden Test with the Provided Checker

Wire the provided checker into `ctest` (tested with CMake 3.28):

```cmake
find_package(Python3 COMPONENTS Interpreter)
if(Python3_FOUND)
  add_test(NAME checker_small
           COMMAND ${CMAKE_SOURCE_DIR}/tests/run_checker.sh
                   $<TARGET_FILE:ob_replay> ${Python3_EXECUTABLE}
                   ${CMAKE_SOURCE_DIR}/tools/check.py ${CMAKE_SOURCE_DIR}/data/small.feed)
endif()
```

```bash
#!/usr/bin/env bash
# Usage: run_checker.sh <replay-binary> <python> <checker.py> <feed>
# Replays the feed, then lets the provided checker judge the output.
set -euo pipefail
replay=$1 python=$2 checker=$3 feed=$4
out=$(mktemp)
trap 'rm -f "$out"' EXIT
"$replay" "$feed" > "$out"
"$python" -I "$checker" "$out"      # adapt the arguments to the checker's real CLI
```

Make sure the checker's exit code reflects failure; if it only prints a verdict, grep for it in the script.

## Fuzzing

```cpp
// fuzz/fuzz_pipeline.cpp — whole pipeline on arbitrary bytes.
extern "C" int LLVMFuzzerTestOneInput(const std::uint8_t* data, std::size_t size) {
    const std::string_view input(reinterpret_cast<const char*>(data), size);
    Pipeline pipeline;                    // parser + sequencer + publisher into a null sink
    pipeline.feed(input);
    pipeline.assert_invariants();         // abort() if a published book breaks a post-condition
    return 0;
}
```

```bash
clang++ -std=c++17 -g -O1 -fsanitize=fuzzer,address,undefined -Iinclude \
    fuzz/fuzz_pipeline.cpp src/*.cpp -o fuzz_pipeline
./fuzz_pipeline -max_total_time=300 fuzz/corpus/     # seed corpus: small feed + scenario feeds
```

Add a dictionary of feed tokens (message kinds, sides, separators) so the fuzzer reaches deep states. Without Clang's libFuzzer, a seeded mutation loop in a test (flip, insert, delete bytes of real lines) gives most of the value.

## Mutation Sanity Check

Before trusting the suite, break the code on purpose. Each mutation must make at least one test fail; if none does, a test is missing.

| Mutation | Caught by scenario |
|---|---|
| Delete the stale check | 6, 7 |
| Don't invalidate on a gap | 8–12 |
| Skip an invalid next-expected message and stay synced | 20–24, 26 |
| Skip the post-apply cross check | 28, 29 |
| Check crossing after every level | 30 |
| Publish the last book while unsynced | 8, 34 |
| Adopt snapshots without validation | 3 |
| Parse prices through `double` | 22, generator safety property |

## Sanitizers and Coverage

- The whole suite passes in the sanitizer build (CPP-9).
- Branch coverage of the sequencer and validators (`--coverage` + `gcovr`, or `llvm-cov`) reaches every transition-table row; 100 % is achievable there and expected.

---

## Checklist

- [ ] Every OB rule and every anomaly-catalogue row has a named scenario test.
- [ ] Scenario tests assert published sequences *and* counters.
- [ ] Ground-truth generator checks safety, liveness, conservation and determinism.
- [ ] Differential test guards every optimisation.
- [ ] Provided checker runs in `ctest` and fails the suite on a bad result.
- [ ] Fuzzer ran clean for at least five minutes; reproducers committed as tests.
- [ ] Every mutation in the table is caught.
- [ ] Suite is clean under ASan + UBSan.
