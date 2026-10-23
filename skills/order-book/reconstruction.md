# Skill: Order Book Reconstruction

Data model, number representation and data structures for building price-level (L2) and order-level (L3) books from snapshots and incremental updates in C++17. *When* the book may be published is decided by `skills/order-book/feed-integrity.md`; this skill covers *what* the book is and how to mutate it safely and fast.

Rules: `rules/order-book-integrity.md` (OB-12 to OB-16 in particular).

---

## Vocabulary

| Term | Meaning |
|---|---|
| L1 / top of book | Best bid and best ask with their quantities |
| L2 / market-by-price (MBP) | Aggregated quantity per price level |
| L3 / market-by-order (MBO) | Individual orders with ids, queue position per level |
| Snapshot | Full book state as of a sequence number |
| Delta / incremental | Change since the previous sequence number |
| Absolute update | Sets a level's quantity; quantity 0 removes the level (idempotent) |
| Incremental update | Adds to or subtracts from a level's quantity (not idempotent) |
| Locked / crossed | Best bid == best ask / best bid > best ask |
| Top-N feed | Venue only reports the N best levels per side |

---

## Prices and Quantities: Fixed-Point Integers

- Represent prices and quantities as `std::int64_t` counts of `10^-scale` (or of the tick size). Never `double` (OB-15): `0.1 + 0.2 != 0.3`, so levels split, deletes miss, and equality checks lie.
- Derive `scale` from the spec, or from the maximum decimals observed while profiling the feed — and confirm it against the checker's output format.
- Parse text straight to an integer. A round-trip through `strtod` loses exactness and accepts `nan`, `inf`, hex floats and leading whitespace.
- Reject, never round (OB-13): a price with more decimals than the scale, or off the tick grid, is bad data.
- Wrap in strong types once the core works (`struct Price { std::int64_t ticks; }`) so `set(price, qty)` cannot be called with the arguments swapped.
- Notional (`price × qty`) can overflow `int64`; use `__int128` or avoid computing it.

Compiled and tested (GCC 13, Clang 18, `-Wconversion`, ASan + UBSan):

```cpp
// Parses an unsigned decimal ("123", "123.45") into integer units of 10^-scale.
// Rejects signs, exponents, whitespace, empty parts, excess precision and overflow.
std::optional<std::int64_t> parse_fixed(std::string_view s, int scale) {
    constexpr std::int64_t kMax = std::numeric_limits<std::int64_t>::max();
    std::int64_t value = 0;
    int int_digits = 0;
    int frac_digits = -1;                              // -1: no '.' seen yet
    for (char c : s) {
        if (c == '.') {
            if (frac_digits >= 0 || int_digits == 0) return std::nullopt;
            frac_digits = 0;
            continue;
        }
        if (c < '0' || c > '9') return std::nullopt;
        const int d = c - '0';
        if (frac_digits < 0) {
            ++int_digits;
        } else if (++frac_digits > scale) {
            if (d != 0) return std::nullopt;           // finer than the price scale
            continue;                                  // harmless trailing zero
        }
        if (value > (kMax - d) / 10) return std::nullopt;  // overflow
        value = value * 10 + d;
    }
    if (int_digits == 0 || frac_digits == 0) return std::nullopt;  // "", ".5", "12."
    for (int i = std::max(frac_digits, 0); i < scale; ++i) {
        if (value > kMax / 10) return std::nullopt;
        value *= 10;
    }
    return value;
}

// Writes v / 10^scale with exactly `scale` decimals: (1234500, 4) -> "123.4500".
// Precondition: v >= 0, 0 <= scale <= 18, out has room for 40 chars.
std::size_t format_fixed(std::int64_t v, int scale, char* out) {
    char digits[20];
    const auto n = static_cast<std::size_t>(std::to_chars(digits, digits + sizeof digits, v).ptr - digits);
    const auto s = static_cast<std::size_t>(scale);
    if (s == 0) { std::memcpy(out, digits, n); return n; }
    std::size_t w = 0;
    if (n <= s) {                                      // 0.00ddd
        out[w++] = '0';
        out[w++] = '.';
        for (std::size_t i = n; i < s; ++i) out[w++] = '0';
    } else {
        std::memcpy(out, digits, n - s);
        w = n - s;
        out[w++] = '.';
        std::memcpy(out + w, digits + n - s, s);
        return w + s;
    }
    std::memcpy(out + w, digits, n);
    return w + n;
}
```

**Match the checker's number format.** Read how the checker compares numbers before choosing the output format:

| Checker compares by | Consequence |
|---|---|
| Exact string | Trailing zeros, decimal count and `-0` matter — reproduce its format exactly |
| `Decimal(...)` | Formatting is free; values must be exact |
| `float(...)` with tolerance | Formatting is free; still compute in fixed point |

---

## Book-Side Containers

| Container | Update | Best level | Walk top N | Verdict |
|---|---|---|---|---|
| `std::map<int64_t, int64_t>` (`std::greater<>` for bids) | O(log n), one allocation per new level | O(1) | easy | Simplest correct choice; keep it forever as the **reference model** for differential tests |
| Sorted `std::vector`, best level at the back | O(log n) search + move of the levels *behind* the touched one | O(1) | contiguous | Usually fastest for depth feeds: activity clusters at the top, so inserts and erases move few elements |
| Dense array indexed by tick offset + occupancy bitmap | O(1) | bit scan | contiguous | Fastest when the price band is narrow and known; needs re-centring and bounds handling — complexity you must justify |
| B-tree / flat map library | O(log n) | O(1) | good | Adds a dependency (CPP-6) for a modest gain |

Start with `std::map`. Move to the sorted vector only with a benchmark showing the book phase matters and a differential test proving equivalence (`skills/order-book/testing.md`). Record the choice in an ADR.

Compiled and tested:

```cpp
struct Level { std::int64_t price; std::int64_t qty; };
enum class SetResult { Inserted, Updated, Deleted, DeleteMissing };

// One side of an L2 book. Storage order is worst -> best, so the best level is
// levels_.back() and the frequent top-of-book inserts/erases move few elements.
template <bool IsBid>
class BookSide {
public:
    // Absolute-quantity update; qty == 0 removes the level.
    SetResult set(std::int64_t price, std::int64_t qty) {
        auto it = find_slot(price);
        const bool found = it != levels_.end() && it->price == price;
        if (qty == 0) {
            if (!found) return SetResult::DeleteMissing;
            levels_.erase(it);
            return SetResult::Deleted;
        }
        if (found) { it->qty = qty; return SetResult::Updated; }
        levels_.insert(it, Level{price, qty});
        return SetResult::Inserted;
    }
    bool contains(std::int64_t price) const {
        auto it = std::lower_bound(levels_.begin(), levels_.end(), price, &worse_than);
        return it != levels_.end() && it->price == price;
    }
    bool empty() const { return levels_.empty(); }
    std::size_t depth() const { return levels_.size(); }
    const Level& best() const { return levels_.back(); }                       // pre: !empty()
    const Level& nth_best(std::size_t i) const { return levels_[levels_.size() - 1 - i]; }
    void clear() { levels_.clear(); }

private:
    static bool worse_than(const Level& l, std::int64_t p) { return IsBid ? l.price < p : l.price > p; }
    std::vector<Level>::iterator find_slot(std::int64_t price) {
        return std::lower_bound(levels_.begin(), levels_.end(), price, &worse_than);
    }
    std::vector<Level> levels_;
};

struct Book {
    BookSide<true> bids;
    BookSide<false> asks;
    bool crossed_or_locked() const {
        return !bids.empty() && !asks.empty() && bids.best().price >= asks.best().price;
    }
    void clear() { bids.clear(); asks.clear(); }
};
```

Tuning notes, each to be proven by measurement: a linear scan from the back often beats binary search when most updates land in the top ~10 levels; `reserve()` the vectors from snapshot depth; keep `Level` at 16 bytes.

---

## Applying a Message Atomically

OB-12 requires validate-then-mutate:

1. **Parse** the whole message into a fixed-capacity staging array of `LevelUpdate` (no heap allocation per message).
2. **Validate** every level: price > 0, qty ≥ 0, on grid, side known, no duplicate price per side in the message, and book-dependent checks (delete of a missing level) as read-only lookups.
3. **Apply** all levels.
4. **Check** post-conditions once, after the whole message (OB-16).

Incremental (+/−) feeds add one rule: a decrement below zero is a desync, not a clamp-to-zero.

## Snapshots

Build into a fresh `Book`, validate, then adopt by move/swap — a rejected snapshot must not damage the current book. Sorted-input snapshots can be loaded with `push_back` in worst→best order (O(n)) instead of n binary-search inserts; still check ordering and duplicates while loading.

## L3 (Order-Level) Books

Only if the feed carries order ids:

- `std::unordered_map<OrderId, OrderHandle>` for lookup, plus per-level FIFO queues (intrusive list or index-linked vector) and aggregated level quantities.
- Add / modify / cancel / execute each validate against the order: unknown id, side mismatch, or a fill larger than the remaining quantity is desync evidence (OB-14).
- A price change is cancel + add (loses queue priority) unless the venue says otherwise.
- Aggregated L2 levels derived from L3 must equal the sum of their orders — assert it in debug builds and tests.

## Top-N (Limited-Depth) Feeds

If the venue only reports changes inside its top N levels, a level that drops out of the window is no longer maintained, and when a better level is removed the level sliding in from N+1 may be unknown. Decide — and document — the depth you can prove, and never publish deeper than that.

## Checksums

- Venue-defined text over the top N levels (e.g. top 10 for Kraken, top 25 interleaved bid/ask for OKX), then CRC32. Signed versus unsigned 32-bit comparison is a classic bug.
- The text must be the venue's exact representation of price and quantity; keep the original token or a formatter proven to reproduce it.
- Verify after the whole message and before publishing (OB-5).

## Output

- Format into a reusable buffer with `std::to_chars` / `format_fixed`, write in large chunks, never `std::endl` (`skills/cpp/cpp.md`).
- Emit exactly what the checker expects for unsynced states — explicit marker, empty line, or nothing.
- Formatting output often costs more than updating the book; benchmark it as its own phase.

---

## Gotchas

- **Inverted comparator** — best bid is the *highest* price, best ask the *lowest*. A flipped comparator yields a plausible-looking but wrong book; test ordering on both sides explicitly.
- **`std::map` bids without `std::greater<>`** — `begin()` is then the *worst* bid.
- **qty 0** means delete in an absolute delta, but is invalid inside a snapshot.
- **Same price on both sides** in one message — a lock/cross candidate, evaluated after the whole message.
- **Dangling views** — a `const Level&` or `string_view` held across a `set()` or buffer refill is a use-after-free waiting to happen; views never outlive the message.
- **Depth assumptions** — measure the real maximum depth from the data before choosing a dense array.

---

## Checklist

- [ ] Prices and quantities are fixed-point integers parsed directly from text; overflow and excess precision rejected.
- [ ] Output number format matches how the checker compares.
- [ ] `std::map` reference book exists and is used by differential tests.
- [ ] Any optimised container is justified by a benchmark and an ADR.
- [ ] Messages are staged, validated, applied, then post-checked — in that order.
- [ ] Snapshots are built off to the side and adopted only after validation.
- [ ] Publishable depth is defined and enforced for limited-depth feeds.
