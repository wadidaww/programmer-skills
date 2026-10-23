---
applyTo: "**/{CMakeLists.txt,*.cmake,*.cpp,*.cc,*.cxx,*.hpp,*.hh,*.h}"
---
# Rules: C++ Build and Toolchain

Rules for C++ deliverables that someone else must build, test and benchmark on Linux x86-64 by following the README. **MUST** rules are blocking; **SHOULD** rules need a written reason to break.

How-to: `skills/cpp/cpp.md`.

---

## Language and portability

- **CPP-1 (MUST) C++17 or later, declared in the build.** `CMAKE_CXX_STANDARD 17`, `CMAKE_CXX_STANDARD_REQUIRED ON`, `CMAKE_CXX_EXTENSIONS OFF`. No GNU extensions or compiler-specific behaviour without a guarded fallback.
- **CPP-2 (MUST) Mainstream toolchains.** Builds with a current GCC and Clang on Linux x86-64. The README names the exact compiler, CMake and OS versions the build was verified with.
- **CPP-3 (MUST) Portable default flags.** `-march=native`, `-ffast-math`, LTO and PGO are opt-in options, off by default. A binary built on the grader's machine must run there. `-ffast-math` is never used in code that validates numbers (it lets the compiler assume NaN and Inf do not exist).
- **CPP-4 (MUST) Release by default.** If `CMAKE_BUILD_TYPE` is unset, default to `Release`; a Debug build must never be benchmarked by accident.

## Reproducible build

- **CPP-5 (MUST) README commands work verbatim.** From a clean clone, in a fresh container, the documented commands configure, build and test without edits. Verify it before handing off (`skills/cpp/cpp.md` → Fresh-container verification).
- **CPP-6 (MUST) No network at build time by default.** Keep the core free of third-party dependencies; vendor single-header test frameworks with their licence. If anything is downloaded (`FetchContent`), pin it by commit hash and say in the README that the build needs network access.
- **CPP-7 (MUST) Warnings on, `-Werror` opt-in.** Always compile with `-Wall -Wextra -Wpedantic` (plus `-Wshadow -Wconversion` where practical) and keep the build warning-free. Enable `-Werror` only through an option (e.g. `-DOB_WERROR=ON`) used in CI, so a newer compiler on the grader's machine cannot break the build.
- **CPP-8 (MUST) Nothing generated in git.** No build directories, binaries, or large data files. Large feeds are regenerated or extracted by a documented command, and identified by size and checksum.

## Safety

- **CPP-9 (MUST) Sanitizer build.** A build option enables AddressSanitizer + UndefinedBehaviorSanitizer (`-fno-sanitize-recover=all`) and the full test suite passes under it. Add ThreadSanitizer if the code has more than one thread.
- **CPP-10 (MUST) No undefined behaviour.** No signed overflow, out-of-bounds access, uninitialised reads, dangling views or references, strict-aliasing violations (use `std::memcpy` to reinterpret bytes), or data races.
- **CPP-11 (MUST) Errors are values on the hot path.** Parsers and validators return a result with a reason (`std::optional`, or a small struct with a reject-reason enum). Exceptions are for startup failures only; `main` catches them, prints a useful message to `stderr`, and exits non-zero. Output write errors are checked and produce a non-zero exit.
- **CPP-12 (MUST) One command runs every test.** `ctest` runs unit tests, scenario tests, and the end-to-end check of the small feed with the provided checker.

## Design

- **CPP-13 (SHOULD) Core is a library.** Parsing, sequencing, book and publishing live in a library with no `main`; the replay tool, benchmark, generator, tests and fuzz targets all link the same library.
- **CPP-14 (SHOULD) No steady-state heap allocation per message.** Reserve buffers up front, parse into fixed-capacity structures, reuse output buffers. Verify with an allocation counter or `heaptrack`, not by inspection.
- **CPP-15 (SHOULD) Single-threaded core.** A replay of millions of messages does not need threads; threads add nondeterminism (OB-18) and race risk. Add them only with a measured reason and a TSan-clean suite.
- **CPP-16 (SHOULD) Formatted and linted.** Commit a `.clang-format`; run `clang-tidy` (`bugprone-*`, `performance-*`) and fix or justify each finding.
