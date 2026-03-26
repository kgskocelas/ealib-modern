# =============================================================================
# libea/CMakeLists.txt  –  Build script for the core EA library
# =============================================================================
#
# STRUCTURE OF LIBEA
#   libea is split into two parts that need to be handled differently:
#
#   1. HEADER-ONLY part (everything under include/ea/)
#      The evolutionary algorithm templates live entirely in .h files.
#      There is nothing to compile — the compiler processes these headers
#      only when another file #includes them.  We call this "header-only."
#
#   2. COMPILED part (src/cmdline_interface.cpp, src/expansion.cpp,
#                      src/main.cpp)
#      These .cpp files contain code that must be compiled into object files
#      (.o) and then bundled into static libraries (.a).
#
#   We create three CMake "targets" to model this:
#       libea         – the header-only part  (INTERFACE library)
#       libea_cmdline – the compiled part      (STATIC library)
#       libea_runner  – just main.cpp          (STATIC library)
#
# WHAT IS A "TARGET"?
#   In CMake, a target is anything that can be built: a library, an
#   executable, or even just a collection of settings.  Targets can depend
#   on other targets, and CMake figures out the right build order.
#
# Docs: https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html
# =============================================================================


# =============================================================================
# TARGET 1: libea  (INTERFACE – header-only)
# =============================================================================
#
# An INTERFACE library is a special CMake target that has no compiled code of
# its own.  Think of it as a "stamp" you can put on other targets that says
# "you need these include paths and these compile settings."  When another
# target links against libea, it automatically inherits everything marked
# INTERFACE here.
#
# This is exactly what we want for a header-only template library: the
# headers need to be findable, but there's nothing to compile.
#
# Docs: https://cmake.org/cmake/help/latest/command/add_library.html
#       (search for "INTERFACE")
add_library(libea INTERFACE)

# Tell CMake where the headers live.
#
# The $<BUILD_INTERFACE:...> and $<INSTALL_INTERFACE:...> are "generator
# expressions" — they evaluate to different paths depending on whether you're
# building the project locally or installing it to your system.
#
#   $<BUILD_INTERFACE:...> is used when compiling within this source tree.
#   $<INSTALL_INTERFACE:...> is used after `cmake --install` copies
#   headers to /usr/local/include or similar.
#
# The "INTERFACE" keyword here means: propagate this include path to any
# target that links against libea.  A target that does
#     target_link_libraries(my_program PRIVATE libea)
# will automatically get -I<libea/include> added to its compiler flags.
#
# Docs: https://cmake.org/cmake/help/latest/command/target_include_directories.html
target_include_directories(libea INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# libea uses Boost headers extensively (boost/random, boost/serialization,
# boost/accumulators, etc.).  Linking against Boost::headers gives any target
# that uses libea the Boost include path automatically.
#
# Boost::headers is a special CMake target provided by Boost's own CMake
# config files.  It represents the header-only part of Boost (no linking
# against compiled .a/.dylib files needed for this target).
#
# Docs: https://cmake.org/cmake/help/latest/module/FindBoost.html
target_link_libraries(libea INTERFACE
    Boost::headers
)


# =============================================================================
# TARGET 2: libea_cmdline  (STATIC – compiled library)
# =============================================================================
#
# This is the compiled portion of libea.  STATIC means the compiler builds
# these .cpp files into object files (.o) and then the archiver (ar) bundles
# them into a single static library file (liblibea_cmdline.a).
#
# A static library is essentially a zip file of compiled code.  When an
# executable links against it, the linker copies the needed pieces directly
# into the final binary.  This is different from a dynamic/shared library
# (.so or .dylib) where the code stays in a separate file and is loaded at
# runtime.
#
# What lives here:
#   cmdline_interface.cpp – implements the command-line / config-file parser
#                           (uses Boost.ProgramOptions and Boost.Serialization)
#   expansion.cpp         – expands environment variables in file paths
#                           (uses POSIX wordexp())
#
# Docs: https://cmake.org/cmake/help/latest/command/add_library.html
add_library(libea_cmdline STATIC
    src/cmdline_interface.cpp
    src/expansion.cpp
)

# Make libea's headers available to this library's own .cpp files AND to
# anything that links against libea_cmdline.
#
# "PUBLIC" means: use this both when compiling libea_cmdline itself AND
# propagate it to any target that links against libea_cmdline.
# (Compare with PRIVATE, which would keep it internal to libea_cmdline.)
#
# Docs: https://cmake.org/cmake/help/latest/command/target_include_directories.html
target_include_directories(libea_cmdline PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Link against all required Boost compiled libraries and zlib.
#
# "PUBLIC" means: these libraries are needed both to build libea_cmdline
# AND are needed by any executable that links against libea_cmdline.  In
# other words, the dependency is "infectious" — users of libea_cmdline
# automatically get all these Boost libs linked in too.  This is correct
# here because the headers in libea/include reference Boost types, so any
# code that #includes them also needs the libraries at link time.
#
# Why each library is needed:
#
#   libea          – the header-only part above; gives us the include path
#
#   Boost::serialization
#       Saves and restores C++ objects (populations, RNG state, etc.) to
#       disk as XML or binary.  Used for checkpointing long EA runs.
#       https://www.boost.org/doc/libs/release/libs/serialization/
#
#   Boost::iostreams
#       Provides layered stream filters.  ealib uses it to transparently
#       read and write gzip-compressed checkpoint files.
#       Needs zlib (see ZLIB::ZLIB below) to do the compression.
#       https://www.boost.org/doc/libs/release/libs/iostreams/
#
#   Boost::regex
#       Regular expression matching.  Used in the command-line interface
#       and checkpoint loader to parse file names and config strings.
#       https://www.boost.org/doc/libs/release/libs/regex/
#
#   Boost::system
#       Provides OS-level error codes (like errno but portable).  Many
#       other Boost libraries depend on it internally.
#       https://www.boost.org/doc/libs/release/libs/system/
#
#   Boost::filesystem
#       Portable file and directory operations (list files, check if a
#       path exists, etc.).  Used when locating checkpoint files.
#       https://www.boost.org/doc/libs/release/libs/filesystem/
#
#   Boost::program_options
#       Parses command-line flags (--population_size 100) and config
#       files.  This is the backbone of ealib's configuration system.
#       https://www.boost.org/doc/libs/release/libs/program_options/
#
#   Boost::timer
#       High-resolution wall-clock timer.  Used in datafiles/runtime.h
#       to record how long each EA update takes.
#       https://www.boost.org/doc/libs/release/libs/timer/
#
#   Boost::chrono
#       Time duration types (seconds, milliseconds, etc.).  Boost.Timer
#       depends on Boost.Chrono at link time.  Without it, the linker
#       would fail with "undefined symbol" errors for the timer library.
#       This dependency was also noted in the original Jamfile.
#       https://www.boost.org/doc/libs/release/libs/chrono/
#
#   ZLIB::ZLIB
#       The zlib compression library.  Boost.Iostreams calls into zlib
#       when compressing/decompressing gzip streams.  On macOS, zlib
#       ships with the Xcode SDK.
#       https://zlib.net
#
# Docs: https://cmake.org/cmake/help/latest/command/target_link_libraries.html
target_link_libraries(libea_cmdline PUBLIC
    libea
    Boost::serialization
    Boost::iostreams
    Boost::regex
    Boost::filesystem
    Boost::program_options
    Boost::timer
    Boost::chrono
    ZLIB::ZLIB
)


# =============================================================================
# TARGET 3: libea_runner  (STATIC – compiled library providing main())
# =============================================================================
#
# src/main.cpp contains the main() entry point shared by all example
# programs.  It calls into the registrar (set up by cmdline_interface) and
# handles top-level exceptions from Boost.ProgramOptions.
#
# Separating main() into its own static library is a classic C++ pattern.
# It lets every example program get a main() for free by linking against
# libea_runner, without each example needing to write its own main().
# It also keeps each example's source file focused purely on defining the
# EA and its configuration.
#
# Linking chain for a typical example:
#
#   example-all-ones
#       → libea_runner   (provides main)
#           → libea_cmdline  (provides command-line parsing, checkpointing)
#               → libea          (provides EA headers)
#               → Boost::*       (all compiled Boost libs)
#               → ZLIB::ZLIB
add_library(libea_runner STATIC
    src/main.cpp
)

# PUBLIC: executables that link against libea_runner also automatically pull
# in libea_cmdline (and therefore all of its Boost dependencies).
target_link_libraries(libea_runner PUBLIC
    libea_cmdline
)
