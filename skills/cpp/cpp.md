# Skill: Modern C++ Projects (C++17)

Engineering practices for small-to-medium C++17 projects that someone else must build, test and benchmark from a README on Linux x86-64: layout, CMake, warnings, error handling, fast and safe text I/O, undefined-behaviour avoidance, and tooling. Rules: `rules/cpp-build.md`.

---

## Project Layout

```
<project>/
├── CMakeLists.txt
├── README.md
├── .clang-format
├── include/<proj>/     # public headers of the core library
├── src/                # core library: parser, sequencer, book, publisher (no main)
├── apps/               # thin mains: replay.cpp, bench.cpp, gen_feed.cpp
├── tests/              # unit + scenario tests, helpers, run_checker.sh, fixtures/
├── fuzz/               # libFuzzer targets and seed corpus (optional build)
├── third_party/        # vendored single-header deps (e.g. doctest.h) + their LICENSE
├── tools/              # scripts: feed profiling, plotting
└── docs/               # knowledge/ (feed spec, ADRs) and ai-usage-log.md
```

The core is a library (CPP-13): tests, benchmark and fuzz targets link exactly the code the replay tool runs.

---

## CMake Template

Tested with CMake 3.28, GCC 13 and Clang 18; requires CMake ≥ 3.16.

```cmake
cmake_minimum_required(VERSION 3.16)
project(orderbook VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)   # benchmarks need it
endif()

option(OB_WERROR   "Treat warnings as errors (CI only)"        OFF)
option(OB_SANITIZE "Build with AddressSanitizer + UBSan"        OFF)
option(OB_NATIVE   "Tune for this CPU (-march=native)"         OFF)

add_compile_options(-Wall -Wextra -Wpedantic -Wshadow -Wconversion)
if(OB_WERROR)
  add_compile_options(-Werror)
endif()
if(OB_SANITIZE)
  add_compile_options(-fsanitize=address,undefined -fno-sanitize-recover=all -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address,undefined)
endif()
if(OB_NATIVE)
  add_compile_options(-march=native)
endif()

add_library(ob_core src/parser.cpp src/book.cpp src/sequencer.cpp src/publisher.cpp)
target_include_directories(ob_core PUBLIC include)

add_executable(ob_replay apps/replay.cpp)
target_link_libraries(ob_replay PRIVATE ob_core)
add_executable(ob_bench apps/bench.cpp)
target_link_libraries(ob_bench PRIVATE ob_core)

enable_testing()
add_executable(ob_tests tests/main.cpp tests/test_parser.cpp tests/test_book.cpp tests/test_scenarios.cpp)
target_include_directories(ob_tests PRIVATE third_party)
target_link_libraries(ob_tests PRIVATE ob_core)
add_test(NAME unit COMMAND ob_tests)
# + checker_small, see skills/order-book/testing.md
```

## README-Ready Commands

```bash
# Build and test (Release)
cmake -S . -B build
cmake --build build -j"$(nproc)"
(cd build && ctest --output-on-failure)

# Sanitizer build
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DOB_SANITIZE=ON
cmake --build build-asan -j"$(nproc)"
(cd build-asan && ctest --output-on-failure)
```

### Fresh-container verification (CPP-5)

```bash
docker run --rm -v "$PWD":/src:ro gcc:13 bash -c '
  apt-get update -qq && apt-get install -y -qq cmake python3 >/dev/null &&
  git config --global --add safe.directory "*" &&
  git clone -q /src /work && cd /work &&
  cmake -S . -B build && cmake --build build -j"$(nproc)" &&
  cd build && ctest --output-on-failure'
```

Cloning (rather than copying) builds only what is committed, which proves the build doesn't depend on untracked files or a stale build directory in your working tree. Run the same with a Clang toolchain if you claim Clang support. Without Docker, the same `git clone` into an empty temporary directory catches most of these mistakes.

---

## Warnings

`-Wall -Wextra -Wpedantic -Wshadow -Wconversion` catches most real bugs in parsing code (sign and width conversions in particular). Fix warnings rather than silencing them; when a cast is needed, make it explicit with `static_cast` and a reason. `-Werror` stays opt-in (CPP-7).

---

## Errors as Values

C++17 has no `std::expected`. Two lightweight patterns:

```cpp
// Reject reasons double as counter indices (OB-17).
enum class Reject : std::uint8_t {
    None, Malformed, BadNumber, BadSide, UnknownType, NonPositivePrice,
    NegativeQty, OffGrid, DuplicatePrice, DeleteMissing, Count
};

template <class T>
struct Parsed {
    T value{};
    Reject reject = Reject::Malformed;
    bool ok() const { return reject == Reject::None; }
};
```

- `std::optional<T>` when the reason doesn't matter (number parsing); `Parsed<T>` when it is counted or reported.
- Exceptions only at startup (arguments, opening files). `main` wraps everything in `try`/`catch`, prints to `stderr`, returns non-zero (CPP-11).

---

## Fast, Safe Text Input

Load the whole file, then walk it with views. Compiled and tested (GCC 13, Clang 18, ASan + UBSan):

```cpp
// Whole-file load: simple, fast enough for hundreds of MB, no partial-read surprises.
std::string read_file(const char* path) {
    std::FILE* f = std::fopen(path, "rb");
    if (!f) throw std::runtime_error(std::string("cannot open ") + path);
    std::string data;
    char chunk[1 << 16];
    std::size_t n;
    while ((n = std::fread(chunk, 1, sizeof chunk, f)) > 0) data.append(chunk, n);
    const bool failed = std::ferror(f) != 0;
    std::fclose(f);
    if (failed) throw std::runtime_error(std::string("read error on ") + path);
    return data;
}

// Calls fn(line) for each line without its '\n' or '\r\n'; handles a missing final newline.
template <class Fn>
void for_each_line(std::string_view data, Fn&& fn) {
    while (!data.empty()) {
        const std::size_t nl = data.find('\n');
        std::string_view line = data.substr(0, nl);
        if (!line.empty() && line.back() == '\r') line.remove_suffix(1);
        fn(line);
        if (nl == std::string_view::npos) break;
        data.remove_prefix(nl + 1);
    }
}

// Splits into at most N fields. Returns the field count, or N + 1 when there are too many.
template <std::size_t N>
std::size_t split(std::string_view line, char delim, std::array<std::string_view, N>& out) {
    std::size_t count = 0;
    for (;;) {
        const std::size_t pos = line.find(delim);
        if (count == N) return N + 1;                  // too many fields: caller rejects
        out[count++] = line.substr(0, pos);
        if (pos == std::string_view::npos) return count;
        line.remove_prefix(pos + 1);
    }
}

// Strict unsigned integer: rejects empty, signs, whitespace, trailing junk and overflow.
std::optional<std::uint64_t> parse_u64(std::string_view s) {
    std::uint64_t v = 0;
    const char* end = s.data() + s.size();
    const auto [ptr, ec] = std::from_chars(s.data(), end, v);
    if (s.empty() || ec != std::errc{} || ptr != end) return std::nullopt;
    return v;
}
```

Decimal prices and quantities: `parse_fixed` in `skills/order-book/reconstruction.md`.

| Avoid | Why |
|---|---|
| `std::getline` + `std::stringstream` | Allocation and locale machinery per line; typically the slowest part of a naive replay |
| `std::stod`, `atof`, `strtod` | Locale-dependent; accept `nan`, `inf`, hex floats, leading spaces; lose exactness |
| `atoi`, `sscanf` | No overflow detection; silent partial parses |
| `std::regex` | Very slow, and easy to make accidentally permissive |
| Floating-point `std::from_chars` | Needs GCC 11+; you should not need floating point for prices anyway |

For JSON feeds: a hand-written parser for the fixed schema (fast, but it must reject everything else) or vendored, version-pinned simdjson (mind its input padding requirement).

`mmap` (with `MADV_SEQUENTIAL`) is an option for very large files; measure it against `read_file` before adding the complexity.

## Fast Output

```cpp
// Accumulates output and writes it in large chunks. Check ok() before exiting 0.
class OutBuffer {
public:
    explicit OutBuffer(std::FILE* f) : f_(f) { buf_.reserve(kFlushAt + 256); }
    ~OutBuffer() { flush(); }
    OutBuffer(const OutBuffer&) = delete;
    OutBuffer& operator=(const OutBuffer&) = delete;

    void put(std::string_view s) { buf_.append(s.data(), s.size()); maybe_flush(); }
    void put(char c) { buf_.push_back(c); }
    void put_u64(std::uint64_t v) {
        char tmp[20];
        const auto r = std::to_chars(tmp, tmp + sizeof tmp, v);
        buf_.append(tmp, static_cast<std::size_t>(r.ptr - tmp));
    }
    void flush() {
        if (!buf_.empty() && std::fwrite(buf_.data(), 1, buf_.size(), f_) != buf_.size()) ok_ = false;
        buf_.clear();
        if (std::fflush(f_) != 0) ok_ = false;
    }
    bool ok() const { return ok_; }

private:
    static constexpr std::size_t kFlushAt = 1 << 20;
    void maybe_flush() { if (buf_.size() >= kFlushAt) flush(); }
    std::FILE* f_;
    std::string buf_;
    bool ok_ = true;
};
```

Never `std::endl` in a loop (it flushes). If you must use iostreams, call `std::ios::sync_with_stdio(false)`.

---

## Undefined-Behaviour Checklist

- **Signed overflow** — check before multiplying or adding parsed values (`parse_fixed` does), or use `__builtin_mul_overflow` / `__builtin_add_overflow`.
- **Out-of-bounds** — `string_view::substr` throws past the end but `operator[]` does not; prefer `find`/`remove_prefix` patterns; `assert` indices in debug.
- **Dangling views** — a `string_view` into a buffer that was reallocated, or a reference into a `std::vector` across `insert`/`erase`.
- **Uninitialised members** — give every member a default initialiser.
- **Type punning** — `std::memcpy`, not `reinterpret_cast` between unrelated types.
- **Shifts** by ≥ the bit width, or of negative values.
- **`std::abs(INT64_MIN)`**, and negating the minimum value in general.
- **Data races** — keep the core single-threaded (CPP-15); otherwise TSan.

---

## Testing Frameworks

| Option | Notes |
|---|---|
| doctest | Single header, fast compile; vendor in `third_party/` — recommended |
| Catch2 v3 | Compiled library; needs to be vendored or fetched |
| GoogleTest | Usually via `FetchContent` ⇒ network at build time (CPP-6) |
| Minimal in-house `CHECK` macro | Zero dependencies; fine for a small suite |

## Tooling

| Need | Tool |
|---|---|
| Memory and UB errors | `-fsanitize=address,undefined` (ThreadSanitizer if threaded) |
| Fuzzing | `clang++ -fsanitize=fuzzer,address,undefined` |
| Static analysis | `clang-tidy` (`bugprone-*`, `performance-*`, `modernize-*`), `cppcheck` |
| Formatting | `clang-format` with a committed `.clang-format` |
| Coverage | `--coverage` + `gcovr`, or `llvm-cov` |
| Profiling | `perf stat` / `perf record`, `heaptrack`, `valgrind --tool=callgrind` |

---

## Checklist

- [ ] C++17 declared; Release default; portable flags; options for `-Werror`, sanitizers, native tuning.
- [ ] Core is a library; apps, tests, bench and fuzz targets link it.
- [ ] No network needed to build; vendored deps carry their licence.
- [ ] Parsers are strict, allocation-free, and return errors as values.
- [ ] Output buffered; write errors detected and reflected in the exit code.
- [ ] Warning-free with `-Wall -Wextra -Wpedantic -Wshadow -Wconversion`.
- [ ] `ctest` green in Release and sanitizer builds.
- [ ] README commands verified in a fresh container.
