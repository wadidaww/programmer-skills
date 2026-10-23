# Skill: Feed Integrity and Book Synchronisation

How to turn an unreliable depth feed — stale messages, sequence gaps, bad data — into an order book that only ever publishes states it can prove. This is the correctness core: build it first, build it simple, and test every transition.

Rules enforced here: `rules/order-book-integrity.md` (OB-*). Data structures: `skills/order-book/reconstruction.md`. Tests: `skills/order-book/testing.md`.

---

## The Proof Obligation

A book `B` published at sequence `s` is **provably correct** when all of these hold:

1. `B` started from a snapshot `S₀` at sequence `s₀` that passed validation.
2. Every update in `(s₀, s]` was received, validated, and applied exactly once, in sequence order — no hole, no double application.
3. Each message was applied atomically: the whole message or none of it.
4. `B` satisfies the post-conditions: sorted, unique levels; positive quantities; not crossed or locked.
5. `B` agrees with every integrity signal available at `s`: checksum, or a snapshot taken at the same sequence.

Anything else is **unproven**, even if it happens to be right. The sequencer's job is to know, cheaply and at every message, whether (1)–(5) still hold. That knowledge is the state machine below.

Know the limit: a message that is corrupted yet well-formed, correctly sequenced and not covered by a checksum cannot be detected by anyone. Put that in the README's limitations instead of claiming more.

---

## Identify the Sequencing Model First

Profile the feed and read the checker (see `agents/market-data-engineer.md`, Phase 1) until you can name the model. Do not write the sequencer before this.

| Model | Feed carries | Next message is valid when | Typical sources |
|---|---|---|---|
| **Contiguous** | one `seq` per message | `seq == last + 1` | recorded/test feeds, ITCH-style, Coinbase |
| **Range** | `first..last` ids per message | `first == last + 1`; first after snapshot: `first ≤ snap + 1 ≤ last` | Binance spot (`U`/`u`) |
| **Linked** | the previous message's id | `prev == last` | Binance futures (`pu`), Deribit (`prev_change_id`) |
| **Checksum** | CRC of the top N levels | computed CRC == provided CRC | Kraken, OKX |
| **Snapshot-only** | periodic full books | each snapshot validates on its own | low-frequency feeds |

Feeds combine models (sequence + checksum). Each signal is another check in the proof, not an alternative to the others.

Also pin down, with real example lines:

- **Update semantics** — *absolute* (set level to qty; qty 0 deletes) or *incremental* (add/subtract). Absolute updates are idempotent; incremental ones are not (OB-10).
- **Granularity** — one level per message or many (many ⇒ atomic batch, OB-12, OB-16).
- **Snapshot delivery** — in-band and in order, or out-of-band (separate file, possibly older or newer than the first deltas).
- **Sequence scope** — per instrument or global; do non-book messages (trades, heartbeats) consume sequence numbers? If they do, they advance `last` after validation without touching the book — otherwise every heartbeat looks like a gap.
- **Output contract** — what the checker expects for every input line, including lines where the book is unsynced.

---

## Sync State Machine

```
                  valid snapshot, not older than newest validated seq
  ┌──────────────────┐ ─────────────────────────────────────────▶ ┌──────────┐
  │ AwaitingSnapshot │                                             │  Synced  │──▶ publish
  └──────────────────┘ ◀───────────────────────────────────────── └──────────┘    (gate)
       │      ▲          gap · lost update · broken post-condition     │    ▲
       └──────┘          · checksum mismatch                           └────┘
   delta: ignore, count                                     next valid delta: apply
                                                            stale delta: drop, count
```

Two states suffice for an in-order recorded feed. Add a `Buffering` state only if the spec requires bounded reorder recovery or out-of-band snapshots (OB-11).

### Transition table — write it before the code, test every row

| State | Event | Condition | Next | Action | Counter |
|---|---|---|---|---|---|
| Awaiting | snapshot | fails validation | Awaiting | reject | `snapshot_rejected` |
| Awaiting | snapshot | older than newest validated seq seen | Awaiting | reject | `snapshot_stale` |
| Awaiting | snapshot | valid | Synced | build book, `last = snap.seq`, publish | `snapshot_adopted` |
| Awaiting | delta | any | Awaiting | drop (or buffer if specified) | `ignored_unsynced` |
| Synced | delta | `seq ≤ last` | Synced | drop, book untouched | `stale` |
| Synced | delta | `seq > last + 1` | Awaiting | clear book | `gap` |
| Synced | delta | `seq == last + 1`, invalid | Awaiting | clear book | `lost_update[reason]` |
| Synced | delta | `seq == last + 1`, valid, post-condition fails | Awaiting | clear book | `post_condition_failed` |
| Synced | delta | `seq == last + 1`, valid, post-conditions hold | Synced | apply, `last = seq`, publish | `applied` |
| Synced | snapshot | fails validation | Synced | reject; chain still intact | `snapshot_rejected` |
| Synced | snapshot | `seq < last` | Synced | drop | `snapshot_stale` |
| Synced | snapshot | `seq == last` | Synced | compare; on mismatch adopt snapshot | `snapshot_mismatch` |
| Synced | snapshot | `seq > last` | Synced | replace book, `last = snap.seq` | `snapshot_adopted` |
| any | line | kind or sequence unidentifiable | unchanged | reject | `malformed` |

A `snapshot_mismatch` means a state you published may have been wrong. The snapshot is the stronger evidence, so adopt it — and treat any non-zero mismatch count on a test feed as a bug to root-cause.

### Per-message decision procedure

```
for each line:
  1. Frame     kind + sequence unidentifiable?            → malformed: reject, no state change
  2. Sequence  seq ≤ last?                                → stale: drop, no state change
               seq > last + 1?                            → gap: invalidate
  3. Validate  next expected, but content invalid?        → lost update: invalidate
  4. Apply     the whole message, atomically
  5. Check     sorted · unique · qty > 0 · uncrossed · CRC → otherwise invalidate
  6. Publish   through the gate only
```

Why this order:

- Classifying the sequence before validating content means a corrupted *stale* message is dropped harmlessly, while a corrupted *next* message is recognised as a lost update instead of being silently skipped.
- A line whose sequence can't be read changes nothing. If it really carried update `last + 1`, the following message arrives as `last + 2` and step 2 invalidates. The sequence numbers do the proving — your only job is never to advance `last` past something you did not apply.
- A corrupted sequence number fails safe: too high ⇒ spurious gap (availability loss only); too low ⇒ dropped as stale, and the next message exposes the hole.
- An unknown message type that carries the next sequence is a lost update unless the spec lists it as a non-book message.

---

## Anomaly Catalogue

| Anomaly | Example | Detected by | Action | Rule |
|---|---|---|---|---|
| Duplicate | seq 101 twice | `seq ≤ last` | drop | OB-6 |
| Late / stale | seq 95 after 100 | `seq ≤ last` | drop | OB-6 |
| Gap | 100 → 103 | `seq > last + 1` | invalidate | OB-7 |
| Swapped pair | 102 before 101 | 102 is a gap, 101 then stale | invalidate (unless bounded reorder is specified) | OB-7, OB-11 |
| Truncated / garbled line | missing fields, binary junk | framing | reject; next sequence decides | OB-8 |
| Bad number | `1O0.5`, `1e3`, `nan`, `inf`, `+1`, ` 1`, overflow | strict parser | lost update if next expected | OB-13 |
| Unknown side or type | side `X` | enum parse | lost update if next expected | OB-8 |
| Non-positive price | `0`, `-5` | validation | lost update | OB-13 |
| Negative quantity | `-1` | validation | lost update | OB-13 |
| Off-grid price | `100.003` on a 0.01 grid | `ticks % tick != 0` | lost update | OB-13 |
| Same price twice in one message | two bid updates at 100.5 | validation | reject message unless spec says last-wins | OB-2 |
| Delete of a missing level | qty 0 at an unknown price | lookup | invalidate unless spec calls it normal | OB-14 |
| Crossed or locked after a message | best bid ≥ best ask | post-condition | invalidate | OB-5, OB-16 |
| Checksum mismatch | CRC differs | post-condition | invalidate | OB-5 |
| Invalid snapshot | unsorted, duplicate, crossed, qty ≤ 0 | snapshot validation | reject; stay as you were | OB-9 |
| Stale snapshot | older than newest validated seq | sequence compare | drop | OB-6, OB-9 |
| Deltas before first snapshot | — | state | ignore and count | OB-4 |
| Other instrument | different symbol | field check | ignore if multi-instrument, else malformed | OB-2 |
| Timestamp regression | time goes back, seq goes forward | — | count only; time is not a sequencing signal unless the spec says so | OB-2 |

Every row becomes at least one scenario test (`skills/order-book/testing.md`).

---

## Snapshots

- **Validate before adopting.** Build the candidate into a *separate* book; adopt by swap only if it passes (sorted or sortable without duplicates, every qty > 0, prices on grid, not crossed or locked, sequence parsed). A rejected snapshot must leave the current state untouched.
- **Reject stale snapshots even while unsynced.** Track the newest sequence seen on any message that passed validation. Adopting a snapshot older than that would publish a provably outdated book as current.
- **Bridge correctly.** Contiguous: next delta must be `snap + 1`. Range: drop deltas with `last ≤ snap`; the first applied must satisfy `first ≤ snap + 1 ≤ last` (absolute updates only, OB-10). Linked: first delta's `prev` must equal the snapshot id.
- **Out-of-band snapshots.** When the snapshot can arrive after deltas it does not cover, buffer deltas while unsynced in a bounded buffer; after adopting, replay buffered deltas newer than the snapshot through the normal sequence checks; on overflow, discard the buffer and keep waiting.
- **Mid-stream snapshots are free audits.** A snapshot at `seq == last` must equal your book. In tests, assert zero mismatches; in production, count them.

---

## Make Unsafe Publishing Fail to Compile

Give publishers a type that only the sequencer can create, and only while synced. Then "publish an unproven book" (OB-3, OB-4) is not a code-review question — it does not compile.

```cpp
// A read-only view that only the Sequencer can construct, and only while synced.
// Publishers accept nothing else. Do not keep a view across messages.
class VerifiedView {
public:
    const Book& book() const { return *book_; }
    std::uint64_t seq() const { return seq_; }
private:
    friend class Sequencer;
    VerifiedView(const Book& b, std::uint64_t s) : book_(&b), seq_(s) {}
    const Book* book_;
    std::uint64_t seq_;
};

void publish(const VerifiedView& view, OutBuffer& out);   // the only function that prints levels
```

---

## Reference Sequencer (contiguous model)

Compiled and tested with GCC 13 and Clang 18 (`-std=c++17 -Wall -Wextra -Wconversion`, ASan + UBSan). `Book`, `SetResult` and `BookSide` are in `skills/order-book/reconstruction.md`. Adapt the message structs to the real feed; keep the order of checks.

```cpp
enum class SyncState { AwaitingSnapshot, Synced };
enum class Outcome { Applied, Stale, IgnoredUnsynced, Gap, LostUpdate, PostConditionFailed,
                     SnapshotAdopted, SnapshotStale, SnapshotRejected };

struct LevelUpdate { bool is_bid; std::int64_t price; std::int64_t qty; };   // fixed-point ticks
struct Delta    { std::uint64_t seq; std::vector<LevelUpdate> updates; };
struct Snapshot { std::uint64_t seq; std::vector<LevelUpdate> levels; };

class Sequencer {
public:
    Outcome on_delta(const Delta& d) {
        if (state_ != SyncState::Synced) { ++stats_.ignored_unsynced; return Outcome::IgnoredUnsynced; }
        if (d.seq <= last_seq_)          { ++stats_.stale; return Outcome::Stale; }        // OB-6
        if (d.seq != last_seq_ + 1)      return invalidate(Outcome::Gap);                  // OB-7
        if (!is_valid(d))                return invalidate(Outcome::LostUpdate);           // OB-8
        for (const auto& u : d.updates) {                                                  // OB-12
            if (u.is_bid) book_.bids.set(u.price, u.qty);
            else          book_.asks.set(u.price, u.qty);
        }
        last_seq_ = d.seq;
        if (book_.crossed_or_locked())   return invalidate(Outcome::PostConditionFailed);  // OB-5, OB-16
        ++stats_.applied;
        return Outcome::Applied;
    }

    Outcome on_snapshot(const Snapshot& s) {
        Book candidate;                                     // never damage the current book
        if (!build_valid(s, candidate)) { ++stats_.snapshots_rejected; return Outcome::SnapshotRejected; }
        if (state_ == SyncState::Synced && s.seq <= last_seq_) {
            ++stats_.snapshots_stale;                       // == last: compare here (audit)
            return Outcome::SnapshotStale;
        }
        book_ = std::move(candidate);
        last_seq_ = s.seq;
        state_ = SyncState::Synced;
        ++stats_.snapshots_adopted;
        return Outcome::SnapshotAdopted;
    }

    std::optional<VerifiedView> verified() const {          // OB-3, OB-4
        if (state_ != SyncState::Synced) return std::nullopt;
        return VerifiedView(book_, last_seq_);
    }

private:
    // Read-only checks; nothing is mutated until all levels pass (OB-12, OB-13, OB-14).
    bool is_valid(const Delta& d) const {
        if (d.updates.empty()) return false;
        for (const auto& u : d.updates) {
            if (u.price <= 0 || u.qty < 0) return false;
            const bool exists = u.is_bid ? book_.bids.contains(u.price) : book_.asks.contains(u.price);
            if (u.qty == 0 && !exists) return false;        // delete of a missing level
        }
        return true;   // also: duplicate prices within the message, tick grid, side/type enums
    }
    static bool build_valid(const Snapshot& s, Book& out) {
        for (const auto& l : s.levels) {
            if (l.price <= 0 || l.qty <= 0) return false;
            const auto r = l.is_bid ? out.bids.set(l.price, l.qty) : out.asks.set(l.price, l.qty);
            if (r != SetResult::Inserted) return false;     // duplicate price
        }
        return !out.crossed_or_locked();
    }
    Outcome invalidate(Outcome why) {
        state_ = SyncState::AwaitingSnapshot;
        book_.clear();                                      // an unproven book is worthless
        ++stats_.invalidations;
        return why;
    }

    SyncState state_ = SyncState::AwaitingSnapshot;
    std::uint64_t last_seq_ = 0;
    Book book_;
    Stats stats_;
};
```

This sketch omits three things a full implementation needs: rejecting stale snapshots while unsynced (track the newest validated sequence), the `seq == last` snapshot comparison, and per-reason counters (`is_valid` should return a reject reason, and `invalidate` should count gaps, lost updates and post-condition failures separately).

---

## Counters

```cpp
struct Stats {
    std::uint64_t lines = 0, malformed = 0;
    std::uint64_t applied = 0, stale = 0, ignored_unsynced = 0;
    std::uint64_t gaps = 0, post_condition_failed = 0, invalidations = 0;
    std::uint64_t snapshots_adopted = 0, snapshots_rejected = 0, snapshots_stale = 0, snapshot_mismatch = 0;
    std::array<std::uint64_t, kRejectReasons> lost_update{};   // indexed by reject reason
    std::uint64_t published = 0, withheld = 0;
};
```

Conservation check for tests: every line increments exactly one *outcome* counter (`malformed`, `applied`, `stale`, `ignored_unsynced`, `gaps`, each `lost_update` reason, `post_condition_failed`, and the three snapshot outcomes), so those counters sum to `lines`. `invalidations`, `snapshot_mismatch`, `published` and `withheld` are cross-cutting and checked separately. A sum that doesn't add up is a code path you didn't think about.

---

## Profile the Feed Before Designing

A throwaway script answers "which anomalies actually occur, and where?" in minutes. Run it on the provided small feed (`python3 -I`, the archive is untrusted) and turn the first line number of each class into a test fixture.

```python
"""Profile a depth feed before designing anything. Adapt parse() to the real grammar.

Usage: python3 -I feed_profile.py path/to/feed
"""
import collections
import sys


def parse(line):
    """Return (kind, seq) or None when the line is malformed. ADAPT to the feed."""
    parts = line.rstrip("\r\n").split(",")
    if len(parts) < 2:
        return None
    try:
        return parts[0], int(parts[1])
    except ValueError:
        return None


def main(path):
    kinds = collections.Counter()
    order = collections.Counter()     # position relative to the highest sequence seen so far
    first_example = {}                # class -> first line number, to turn into a fixture
    highest = None
    with open(path, encoding="utf-8", errors="replace") as feed:
        for lineno, line in enumerate(feed, 1):
            msg = parse(line)
            if msg is None:
                cls = "malformed"
            else:
                kind, seq = msg
                kinds[kind] += 1
                if highest is None:
                    cls = "first"
                elif seq <= highest:
                    cls = "stale-or-duplicate"
                elif seq == highest + 1:
                    cls = "contiguous"
                else:
                    cls = "gap"
                highest = seq if highest is None else max(highest, seq)
            order[cls] += 1
            first_example.setdefault(cls, lineno)
    print("message kinds:", dict(kinds))
    for cls, count in order.most_common():
        print(f"{cls:>20}: {count:>10}  first at line {first_example[cls]}")


if __name__ == "__main__":
    main(sys.argv[1])
```

Extend it with: decimals per price/qty field (fixes the fixed-point scale), snapshot sizes, levels per message, crossed states after naive application, and the value ranges of every numeric field.

---

## Anti-Patterns

- **Skip and carry on** — dropping a bad next-expected message and continuing leaves a silent hole in the book.
- **Advancing `last` on a rejected message** — erases the evidence the next sequence check needs.
- **Hold last value** — publishing the previous good book while unsynced.
- **Resync from deltas** — "the gap was small, the next updates overwrite those levels anyway". You cannot know that.
- **Repairing** — removing the level that caused a cross, clamping a negative quantity, rounding an off-grid price.
- **Per-level cross checks** — rejecting a valid message because it crosses transiently mid-application.
- **`try { apply(); } catch (...) {}`** — exceptions that swallow half-applied messages.
- **`double` prices** — `100.1 + 0.2 != 100.3`; price levels silently split or fail to delete.
- **Debug dump of the book** — a convenience print that bypasses the gate is still a publish.

---

## Checklist

- [ ] Sequencing model, update semantics, snapshot delivery and output contract written down with real example lines.
- [ ] Transition table written; every row has a test.
- [ ] Order of checks is frame → sequence → validate → apply → post-check → publish.
- [ ] `last` advances only on applied messages (and validated non-book messages that consume sequence numbers).
- [ ] Snapshots are built off to the side and adopted by swap; stale snapshots rejected even while unsynced.
- [ ] Publishers can only receive a `VerifiedView` (or equivalent gate).
- [ ] Counters printed at end of run and conserved.
- [ ] Limitations of the proof stated in the README.
