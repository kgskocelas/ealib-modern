# Modernization Notes

## Goal

Make ealib-modern build cleanly with Boost 1.80+ (tested with Boost 1.87.0 from
Homebrew) on macOS Intel (Ventura 13.7.8, Apple Clang 15, Xcode 15.2).

---

## 1 — Build system: Jamfiles → CMake

### What changed

Replaced the Boost.Build (b2/bjam) build system with CMake.

| Old file | New file |
|----------|----------|
| `Jamroot` | `CMakeLists.txt` |
| `libea/Jamfile` | `libea/CMakeLists.txt` |
| `libmkv/Jamfile` | `libmkv/CMakeLists.txt` |
| `libann/Jamfile` | `libann/CMakeLists.txt` |
| *(none)* | `examples/CMakeLists.txt` |

### Why

Homebrew no longer ships the b2 build tool alongside the Boost libraries.
Installing b2 separately and configuring a site-config.jam for every developer
machine is fragile.  CMake has first-class support in Boost 1.80+ (the
`BoostConfig.cmake` / `find_package(Boost …)` config-mode package is included
in the Homebrew formula).

### Structure

```
CMakeLists.txt          # project root: Boost discovery, global defines
libea/CMakeLists.txt    # INTERFACE target "libea" (headers only)
                        # STATIC target  "libea_cmdline" (compiled)
                        # STATIC target  "libea_runner"  (compiled, provides main())
libmkv/CMakeLists.txt   # INTERFACE target "libmkv" (headers only)
libann/CMakeLists.txt   # INTERFACE target "libann" (headers only)
examples/CMakeLists.txt # eight example executables
```

### Build instructions

```bash
cmake .     # configure (in-source, or use -B build for an out-of-tree build)
make        # compile all targets
```

---

## 2 — Boost bind placeholders deprecation

### What changed

Added the compile definition `BOOST_BIND_GLOBAL_PLACEHOLDERS` in the root
`CMakeLists.txt`.

### Why

Since Boost 1.73, `boost/bind.hpp` emits a deprecation message when it imports
the argument placeholders (`_1`, `_2`, `_3`, …) into the global namespace.
Multiple headers in ealib use `boost::bind` with unqualified placeholders:

```
libea/include/ea/events.h
libea/include/ea/digital_evolution/events.h
libea/include/ea/datafiles/reactions.h
libea/include/ea/datafiles/evaluations.h
```

Defining `BOOST_BIND_GLOBAL_PLACEHOLDERS` before any `#include <boost/bind.hpp>`
restores the pre-1.73 behaviour without touching any source files.

### Decision

Adding the definition as a global CMake compile definition was preferred over
editing every affected header, keeping the source-code diff minimal.

---

## 3 — C++ standard: C++11 minimum raised to C++14

### What changed

The root `CMakeLists.txt` sets `CMAKE_CXX_STANDARD 14`.

### Why

Boost 1.87's `boost/math/tools/type_traits.hpp` uses C++14 type-alias
templates that are absent from C++11:

```
std::remove_cv_t, std::remove_const_t, std::remove_volatile_t,
std::decay_t, std::enable_if_t, std::make_signed_t, std::make_unsigned_t,
std::is_null_pointer, std::is_final, std::add_pointer_t, …
```

These appear in the include chain of `boost/math/constants/constants.hpp`,
which is used in `libea/include/ea/fitness_functions/pole_balancing.h` and
`libea/include/ea/fitness_functions/xor.h`.

C++14 is also consistent with what Apple Clang 15 uses as its default language
dialect (`-std=gnu++14`), so this change simply makes the previously-implicit
requirement explicit.

### Decision

C++14 was chosen as the minimum (rather than C++17) because all existing code
compiles cleanly at that level.  No application code was modified.

---

## 4 — Boost components: no header changes required

The following potentially deprecated Boost APIs were investigated and confirmed
still available in Boost 1.87:

| API | Header | Status in 1.87 |
|-----|--------|---------------|
| `boost::shared_ptr` | `boost/shared_ptr.hpp` | Available (compatibility header) |
| `boost::scoped_ptr` | `boost/scoped_ptr.hpp` | Available (compatibility header) |
| `boost::variate_generator` | `boost/random/variate_generator.hpp` | Available, included by `boost/random.hpp` |
| `boost::uniform_real` | `boost/random/uniform_real.hpp` | Available |
| `boost::uniform_int` | `boost/random/uniform_int.hpp` | Available |
| `boost::normal_distribution` | `boost/random/normal_distribution.hpp` | Available |
| `boost::mt19937` | `boost/random.hpp` | Available (typedef) |
| `boost/tuple/tuple.hpp` | — | Available |
| `boost/bind.hpp` | — | Available (see §2) |

No source files needed modifications for these APIs.

---

## 5 — Boost Chrono added as explicit link dependency

### What changed

`libea_cmdline` links against `Boost::chrono` explicitly.

### Why

The original `libea/Jamfile` listed `boost/chrono` in the library's
usage-requirements (it was added in a commit message as a fix for Avida/mt
builds).  Modern Boost's timer library may depend on Boost.Chrono at link
time on some platforms.  Listing it explicitly prevents `undefined symbol`
linker errors on platforms where `Boost::timer` does not carry the chrono
dependency automatically.

---

## System tested on

| Component | Version |
|-----------|---------|
| macOS | Ventura 13.7.8 (Intel x86_64) |
| Xcode | 15.2 (Apple Clang 15.0.0) |
| Homebrew Boost | 1.87.0 |
| CMake | 4.3.0 |
| zlib | 1.2.12 (Xcode SDK) |
