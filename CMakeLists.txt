# =============================================================================
# CMakeLists.txt  –  Root build script for ealib-modern
# =============================================================================
#
# WHAT IS THIS FILE?
#   CMake is a "build system generator."  You give it a CMakeLists.txt that
#   describes your project, and it figures out how to compile it on whatever
#   platform you're on (macOS, Linux, Windows, etc.).  When you run
#
#       cmake .
#
#   CMake reads this file and generates a regular Makefile (on Unix/macOS) or
#   a Visual Studio project (on Windows).  Then you run `make` to actually
#   compile the code.
#
#   This file replaces the old Boost.Build system (Jamroot + Jamfiles), which
#   required the `b2` tool that Homebrew no longer ships alongside Boost.
#
# DOCS: https://cmake.org/cmake/help/latest/manual/cmake.1.html
# =============================================================================


# -----------------------------------------------------------------------------
# Minimum CMake version
# -----------------------------------------------------------------------------
# This says "this project requires at least CMake 3.14."  Version 3.14
# introduced the ability to use imported targets (like Boost::serialization)
# cleanly, which we rely on throughout this file.
#
# Docs: https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html
cmake_minimum_required(VERSION 3.14)


# -----------------------------------------------------------------------------
# Project declaration
# -----------------------------------------------------------------------------
# Names the project "ealib" and tells CMake it's a C++ project.
# The "CXX" part means CMake will only look for a C++ compiler (not C or
# Fortran), which makes configuration a bit faster.
#
# Docs: https://cmake.org/cmake/help/latest/command/project.html
project(ealib CXX)


# =============================================================================
# C++ LANGUAGE STANDARD
# =============================================================================
#
# WHY THIS IS HERE (the problem we're solving):
#   C++ is a language that has been updated over the years.  Each major
#   revision gets a year-based name:
#       C++98  – the original standard
#       C++11  – added lambdas, smart pointers, range-for, and much more
#       C++14  – small improvements; added "template alias" shortcuts like
#                std::decay_t, std::enable_if_t, std::remove_cv_t, etc.
#       C++17  – more improvements; removed some old deprecated things
#       C++20  – concepts, ranges, coroutines, and more
#
#   The ealib code was originally written around 2014 and targeted C++11.
#   When we upgraded Boost to version 1.87, its math header
#
#       boost/math/tools/type_traits.hpp
#
#   started using C++14 features (those `_t` shortcut names above).  When
#   the compiler tried to build pole_balancing.h or xor.h, which pull in that
#   Boost math header, it failed with errors like:
#
#       error: no member named 'decay_t' in namespace 'std'
#       error: no member named 'enable_if_t' in namespace 'std'
#
#   Those names were only added to std:: in C++14, so we must tell the
#   compiler to use at least C++14.
#
# WHY NOT C++17 OR LATER?
#   C++17 removed std::random_shuffle, which ealib still uses in rng.h.
#   Staying at C++14 keeps everything compiling without changing any of the
#   original source code.
#
# REFERENCES:
#   C++14 standard overview: https://en.cppreference.com/w/cpp/14
#   List of C++14 type-trait aliases: https://en.cppreference.com/w/cpp/types
#   Boost math requires C++14: https://www.boost.org/doc/libs/release/libs/math/

set(CMAKE_CXX_STANDARD 14)

# If the compiler doesn't support C++14, stop immediately with a clear error
# instead of silently compiling broken code.
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Use the *standard* C++14, not the compiler's extended dialect.
# Compilers like GCC and Clang have their own extras (e.g., GNU extensions)
# on top of the standard.  Turning extensions off keeps the build portable.
# Docs: https://cmake.org/cmake/help/latest/prop_tgt/CXX_EXTENSIONS.html
set(CMAKE_CXX_EXTENSIONS OFF)


# =============================================================================
# FINDING BOOST
# =============================================================================
#
# Boost is a large collection of C++ libraries covering things like random
# number generation, file I/O, serialization, string processing, and more.
# ealib uses many of them.
#
# On macOS with Homebrew, `brew install boost` puts Boost's files under
# /usr/local (on Intel Macs) or /opt/homebrew (on Apple Silicon Macs).
# CMake doesn't automatically look there, so we tell it to by appending
# /usr/local to the list of places CMake searches.
#
# This line is harmless on systems where Boost is already in a standard
# location — it just adds one more place to look.
#
# Homebrew prefix docs: https://docs.brew.sh/FAQ
list(APPEND CMAKE_PREFIX_PATH /usr/local)

# Ask CMake to find Boost on this machine.
#
# "1.80"     – the minimum version we need (we have 1.87 installed)
# "REQUIRED" – stop with an error if Boost isn't found; don't continue
# "COMPONENTS ..." – we only use these specific Boost sub-libraries.
#   Boost has 160+ libraries; listing only the ones we need speeds up the
#   search and makes our dependencies explicit.
#
# Brief description of each component:
#   serialization   – saves/loads C++ objects to/from files (checkpointing)
#                     https://www.boost.org/doc/libs/release/libs/serialization/
#   iostreams       – layered I/O streams (e.g., gzip-compressed checkpoints)
#                     https://www.boost.org/doc/libs/release/libs/iostreams/
#   regex           – regular expressions (pattern matching in strings)
#                     https://www.boost.org/doc/libs/release/libs/regex/
#   system          – OS error codes; many other Boost libs depend on it
#                     https://www.boost.org/doc/libs/release/libs/system/
#   filesystem      – reading/writing files and directories portably
#                     https://www.boost.org/doc/libs/release/libs/filesystem/
#   program_options – parses command-line arguments and config files
#                     https://www.boost.org/doc/libs/release/libs/program_options/
#   timer           – high-resolution wall-clock and CPU timers
#                     https://www.boost.org/doc/libs/release/libs/timer/
#   chrono          – time durations and clocks; Boost.Timer depends on it
#                     https://www.boost.org/doc/libs/release/libs/chrono/
#
# After this call succeeds, CMake gives us imported targets like
# Boost::serialization, Boost::filesystem, etc. that we can link against.
#
# Docs: https://cmake.org/cmake/help/latest/module/FindBoost.html
# CMP0167 NEW: use Boost's own BoostConfig.cmake instead of CMake's
# deprecated FindBoost module (removed in CMake 3.30+).
cmake_policy(SET CMP0167 NEW)
find_package(Boost 1.80 REQUIRED COMPONENTS
    serialization
    iostreams
    regex
    filesystem
    program_options
    timer
    chrono
)

# Find zlib, the compression library.
#
# WHY: Boost.Iostreams can compress/decompress data on-the-fly (e.g., to
# write gzip-compressed checkpoint files).  It links against zlib under the
# hood, so we need to find it here and pass it to the linker.  On macOS,
# zlib comes bundled with the Xcode SDK, so this should always succeed.
#
# Docs: https://cmake.org/cmake/help/latest/module/FindZLIB.html
find_package(ZLIB REQUIRED)


# =============================================================================
# GLOBAL COMPILE DEFINITIONS
# =============================================================================
#
# A "compile definition" is a #define macro that gets passed to the compiler
# for every file in the project.  It's equivalent to writing
#
#     #define SOME_MACRO
#
# at the top of every .cpp and .h file, but without actually editing them.
# This lets us tune library behaviour from the build system.
#
# Docs: https://cmake.org/cmake/help/latest/command/add_compile_definitions.html

add_compile_definitions(

    # -----------------------------------------------------------------
    # BOOST_BIND_GLOBAL_PLACEHOLDERS
    # -----------------------------------------------------------------
    # WHY THIS IS NEEDED:
    #   boost::bind is a function that lets you "partially apply" another
    #   function — for example, if you have add(x, y) you can create a new
    #   function add5 = bind(add, 5, _1) that always passes 5 as the first
    #   argument.  The _1, _2, _3 symbols are called "placeholders" and
    #   stand for "whatever argument gets passed later."
    #
    #   Originally these placeholders lived in the global namespace, meaning
    #   you could just write _1 anywhere.  Starting in Boost 1.73, the Boost
    #   developers moved them into boost::placeholders::_1 to avoid
    #   accidentally clashing with other code that might define a variable
    #   called _1 (a common name in some codebases).
    #
    #   The old boost/bind.hpp still works, but it now prints a deprecation
    #   warning telling you to switch.  Defining BOOST_BIND_GLOBAL_PLACEHOLDERS
    #   tells Boost "yes, I know what I'm doing, please keep the old behavior"
    #   and silences the warning.
    #
    #   The ealib headers that use _1/_2/_3 (and would trigger the warning):
    #       libea/include/ea/events.h
    #       libea/include/ea/digital_evolution/events.h
    #       libea/include/ea/datafiles/reactions.h
    #       libea/include/ea/datafiles/evaluations.h
    #
    #   Rather than editing all those files, we define the macro here once
    #   for the whole project.
    #
    #   Boost.Bind docs:      https://www.boost.org/doc/libs/release/libs/bind/
    #   Boost 1.73 changelog: https://www.boost.org/users/history/version_1_73_0.html
    BOOST_BIND_GLOBAL_PLACEHOLDERS

    # -----------------------------------------------------------------
    # BOOST_PARAMETER_MAX_ARITY=7
    # -----------------------------------------------------------------
    # Boost.Parameter is a library that lets functions accept arguments by
    # name (like Python's keyword arguments).  The "arity" is the maximum
    # number of named parameters a single function can accept.  The default
    # is 5; ealib raises it to 7 because some of its EA template types have
    # more than 5 named configuration parameters.
    #
    # This was already in the original Jamroot, so we carry it forward.
    #
    # Boost.Parameter docs: https://www.boost.org/doc/libs/release/libs/parameter/
    BOOST_PARAMETER_MAX_ARITY=7

    # -----------------------------------------------------------------
    # BOOST_GEOMETRY_EMPTY_INPUT_NO_THROW
    # -----------------------------------------------------------------
    # Boost.Geometry is a library for working with shapes and spatial data
    # (points, lines, polygons, etc.).  ealib uses it in the l-system /
    # spatial graph code.
    #
    # By default, some Boost.Geometry algorithms throw an exception if you
    # pass them an empty shape (a polygon with no vertices, for example).
    # This flag tells Boost.Geometry to silently handle empty input instead
    # of throwing, which matches the behavior ealib expects.
    #
    # This was already in the original Jamroot, so we carry it forward.
    #
    # Boost.Geometry docs: https://www.boost.org/doc/libs/release/libs/geometry/
    BOOST_GEOMETRY_EMPTY_INPUT_NO_THROW
)


# =============================================================================
# SUBDIRECTORIES
# =============================================================================
#
# Each subdirectory has its own CMakeLists.txt that defines the targets
# (libraries, executables) in that part of the project.  CMake processes
# them in order, so targets defined earlier can be used by later ones.
# (libea must come before examples, because examples depend on libea.)
#
# Docs: https://cmake.org/cmake/help/latest/command/add_subdirectory.html
add_subdirectory(libea)     # core EA library (must be first)
add_subdirectory(libmkv)    # Markov network headers (depends on libea)
add_subdirectory(libann)    # neural network headers  (depends on libea)
add_subdirectory(examples)  # example programs        (depends on all three)
