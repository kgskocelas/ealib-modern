cmake_minimum_required(VERSION 3.14)
project(ealib CXX)

# Boost 1.80+ headers (boost/math/tools/type_traits.hpp and others) use
# C++14 features such as std::remove_cv_t, std::decay_t, std::enable_if_t,
# std::is_final, and std::is_null_pointer.  C++14 is therefore the minimum
# standard required to compile against modern Boost.
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ---------------------------------------------------------------------------
# Boost discovery
# ---------------------------------------------------------------------------
# Tell CMake where Homebrew installed Boost (Intel Mac default prefix).
# This is a no-op when Boost is already on CMAKE_PREFIX_PATH or when
# building on a system where Boost is in the default search path.
list(APPEND CMAKE_PREFIX_PATH /usr/local)

find_package(Boost 1.80 REQUIRED COMPONENTS
    serialization
    iostreams
    regex
    system
    filesystem
    program_options
    timer
    chrono
)

find_package(ZLIB REQUIRED)

# ---------------------------------------------------------------------------
# Global compile definitions (mirror the original Jamroot settings)
# ---------------------------------------------------------------------------
# BOOST_BIND_GLOBAL_PLACEHOLDERS: In Boost ≥1.73, boost/bind.hpp emits a
#   deprecation warning when _1/_2/… are pulled into the global namespace.
#   Defining this macro restores the old behaviour without changing any
#   source files.
# BOOST_PARAMETER_MAX_ARITY=7 / BOOST_GEOMETRY_EMPTY_INPUT_NO_THROW: from
#   the original Jamroot.
add_compile_definitions(
    BOOST_BIND_GLOBAL_PLACEHOLDERS
    BOOST_PARAMETER_MAX_ARITY=7
    BOOST_GEOMETRY_EMPTY_INPUT_NO_THROW
)

add_subdirectory(libea)
add_subdirectory(libmkv)
add_subdirectory(libann)
add_subdirectory(examples)
