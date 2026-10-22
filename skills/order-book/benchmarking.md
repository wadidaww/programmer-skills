# Skill: Order Book Benchmarking

How to benchmark an order-book replay on a feed of about 4 million messages credibly: what to time, how to stop the compiler and the machine from lying to you, and how to report it so a reviewer can reproduce it. General profiling is in `skills/performance/performance.md`; deep tuning in `agents/low-latency-engineer.md`.

Rules: `rules/deliverables.md` (DEL-4 to DEL-6), `rules/cpp-build.md` (CPP-3, CPP-4).

---

## Get the Data

- **Provided large feed** — use it as is. Record line count, byte size, `sha256sum`, and the anomaly mix (profile it with the script in `skills/order-book/feed-integrity.md`).
- **No large feed provided** — generate one with the ground-truth generator (`skills/order-book/testing.md`) at a fixed seed and realistic fault rates, e.g. `ob_gen --messages 4000000 --seed 1 > data/large.feed`. Commit the generator and the command, not the file (CPP-8); print the file's `sha256` so runs can be compared.
- **The mix matters.** A feed with no anomalies, no snapshots, or a flat price benchmarks a different program. Report the fraction of snapshots, stale messages, gaps and bad lines, and the typical depth.

---

## Measure the Phases Separately

| Phase | What is timed | Why it matters |
|---|---|---|
| Load | File → memory | Disk and page cache, not your code |
| Parse only | All lines → message structs, discarded | Parser cost — often the largest |
| Book only | Pre-parsed messages → sequencer + book + gate | The core you are being judged on |
| End-to-end | Load + parse + book + formatted output to `/dev/null` | What a user of the tool experiences |
| Output | End-to-end to a real file versus to `/dev/null` | Formatting and writes often dominate |
| Per-message latency | Distribution of book-phase time per message | Tails: deep inserts, snapshot rebuilds, resyncs |
| Memory | Peak RSS (`/usr/bin/time -v`) | Whole-file loading versus streaming trade-off |

**Throughput** (messages/s, ns/message) comes from timing the whole phase with no per-message timers. **Latency** comes from a separate run that timestamps every message, because each clock read costs about as much as a book update and would distort throughput.

---

## Harness Rules

1. **Release build** with exactly the README flags. Never benchmark Debug or sanitizer builds (CPP-4).
2. **Warm up, then repeat** — one discarded run, then at least 5 measured runs; report median with min and max.
3. **Pre-load** the file into memory for the parse and book phases; state whether the page cache was warm for end-to-end.
4. **Fresh state per run** — a new book and sequencer each run, so later runs don't replay into a populated book.
5. **Defeat dead-code elimination** — fold every outcome and every published top of book into a 64-bit digest and print it.
6. **Prove the answer** — the digest and counters must equal the correctness run on the same feed (DEL-6).
7. **Measure the timer** — report the median cost of a back-to-back `steady_clock::now()` pair before quoting per-message latencies.
8. **Quiet machine** — pin with `taskset -c <core>`; use the `performance` governor if you control the host; close other work. Report what you could not control.
9. **No allocation in the measured loop** — reserve up front; verify with `heaptrack` or a counting allocator.

---

## Harness Building Blocks

Compiled and tested (GCC 13, Clang 18, ASan + UBSan):

```cpp
using Clock = std::chrono::steady_clock;

// Order-sensitive digest of everything the pipeline produced; print it so the
// optimiser cannot delete the work, and compare it with the correctness run.
inline std::uint64_t mix(std::uint64_t h, std::uint64_t v) {
    h ^= v + 0x9E3779B97F4A7C15ULL + (h << 6) + (h >> 2);
    return h;
}

struct Percentiles { std::uint64_t p50, p99, p999, max; };

// Exact percentiles from per-message samples (4M samples x 8 bytes = 32 MB).
Percentiles percentiles(std::vector<std::uint64_t> ns) {
    auto at = [&ns](double q) {
        const auto k = static_cast<std::size_t>(q * static_cast<double>(ns.size() - 1));
        std::nth_element(ns.begin(), ns.begin() + static_cast<std::ptrdiff_t>(k), ns.end());
        return ns[k];
    };
    const std::uint64_t mx = *std::max_element(ns.begin(), ns.end());
    return {at(0.50), at(0.99), at(0.999), mx};
}

double median(std::vector<double> v) {
    std::sort(v.begin(), v.end());
    return v[v.size() / 2];
}

// Cost of a pair of clock reads on this machine; report it next to latencies.
std::uint64_t timer_overhead_ns() {
    std::vector<std::uint64_t> s(100000);
    for (auto& x : s) {
        const auto a = Clock::now();
        const auto b = Clock::now();
        x = static_cast<std::uint64_t>(std::chrono::duration_cast<std::chrono::nanoseconds>(b - a).count());
    }
    return percentiles(s).p50;
}
```

The book-phase loop (sketch — adapt names to your pipeline):

```cpp
std::vector<double> seconds;
std::uint64_t digest = 0;
for (int run = 0; run <= runs; ++run) {                 // run 0 is the warm-up
    Sequencer seq;                                      // fresh state per run
    std::uint64_t d = 0;
    const auto t0 = Clock::now();
    for (const Message& m : messages) {                 // pre-parsed, in memory
        d = mix(d, static_cast<std::uint64_t>(seq.on_message(m)));
        if (const auto v = seq.verified()) d = mix(d, top_of_book_hash(*v));
    }
    const auto t1 = Clock::now();
    if (run > 0) seconds.push_back(std::chrono::duration<double>(t1 - t0).count());
    digest = d;
}
// print: median(seconds), min, max, messages / median, digest, stats
```

---

## Profile, Then Optimise One Thing at a Time

```bash
perf stat -e task-clock,cycles,instructions,cache-misses,branch-misses ./build/ob_bench data/large.feed --runs 5
perf record -g ./build/ob_bench data/large.feed --runs 1 && perf report       # or a flame graph
heaptrack ./build/ob_bench data/large.feed --runs 1                           # allocations
```

Where the time usually goes in a replay, roughly in order — confirm with the profile before acting:

1. **Parsing** — `std::getline` + `std::stringstream`, `std::stod`, `std::regex` and per-field `std::string` allocations are typically the largest cost. Replace with whole-file buffers, `std::string_view` and `std::from_chars` / `parse_fixed` (`skills/cpp/cpp.md`).
2. **Output** — `std::endl`, `printf` per field, unbuffered streams. Format with `std::to_chars` into a large buffer.
3. **Book container** — `std::map` node allocations and pointer chasing; a sorted vector with the best level at the back usually wins (`skills/order-book/reconstruction.md`).
4. **Allocation** — per-message vectors in message structs; stage into fixed-capacity arrays.
5. **Branches and layout** — only after the above; the anomaly paths are rare, so keep them out of line.

After every change: re-run the differential and ground-truth tests, re-measure, and keep the change only if both pass and the number moved beyond run-to-run noise. Record before/after in the commit message (DEL-9).

`-march=native` and LTO may be reported as an *additional* row, never as the headline number of a default build (CPP-3).

---

## Reporting Template (README)

```markdown
## Benchmark

Feed: data/large.feed — <exact count> messages, <size> MB, sha256 <hash>  (provided | generated: `./build/ob_gen --messages 4000000 --seed 1`)
Mix: <x>% snapshots · <x>% stale · <n> gaps · <x>% bad lines · median depth <n> levels/side
Machine: <CPU model>, 1 core pinned (taskset -c 2), <RAM>, <OS + kernel>; governor: <performance | not controllable>
Build: <compiler + version>, CMake Release (-O2), no -march=native
Command: ./build/ob_bench data/large.feed --runs 10

| Phase                          | Median  | Min | Max | Throughput     |
|--------------------------------|---------|-----|-----|----------------|
| Parse only                     | … ms    | …   | …   | … M msg/s      |
| Book only                      | … ms    | …   | …   | … M msg/s      |
| End-to-end, output → /dev/null | … ms    | …   | …   | … M msg/s      |
| End-to-end, output → file      | … ms    | …   | …   | … M msg/s      |

Book-phase latency per message (ns): p50 … · p99 … · p99.9 … · max …  (clock pair overhead … ns, not subtracted)
Digest 0x… and counters identical to the correctness run. Peak RSS … MB.
```

---

## Pitfalls

- **Debug or sanitizer binary** benchmarked by accident — the default build type must be Release.
- **Benchmarking the generator** — generate the feed to a file first; time only the replay.
- **Cold versus warm page cache** — the first run after generating a file differs from the next; say which you report.
- **Per-message timers inside the throughput loop** — the observer becomes the workload.
- **Averages for latency** — report percentiles and the maximum; one snapshot rebuild hides inside a mean.
- **Silent correctness regressions** — a faster run with a different digest is a bug, not a speed-up.
- **Unreproducible numbers** — no seed, no checksum, no machine description.

---

## Checklist

- [ ] Feed of ~4M messages identified by count, size, checksum and anomaly mix.
- [ ] Phases measured separately; throughput and latency from separate runs.
- [ ] Release build, warm-up, ≥ 5 runs, median with spread.
- [ ] Digest and counters printed and equal to the correctness run.
- [ ] Timer overhead reported next to latency numbers.
- [ ] Profile captured; each optimisation kept only with a measured gain and green differential tests.
- [ ] README table filled from a real run at the final commit, with the environment described.
