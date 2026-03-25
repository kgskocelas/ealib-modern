# =============================================================================
# libmkv/CMakeLists.txt  –  Build script for the Markov network library
# =============================================================================
#
# WHAT IS LIBMKV?
#   libmkv provides data structures and algorithms for Markov networks —
#   a type of probabilistic graph model where each node has a set of inputs
#   and a table of transition probabilities.  In the context of ealib, Markov
#   networks are often evolved as controllers for simulated agents.
#
#   The libmkv headers live under libmkv/include/mkv/ and are #included by
#   libea/include/ea/mkv/markov_network_evolution.h.
#
# HEADER-ONLY: no .cpp files, nothing to compile
#   Like the core libea target, libmkv is entirely template-based and lives
#   in header files.  We therefore declare it as an INTERFACE library — a
#   CMake target with no compiled output, used only to carry include paths
#   and dependencies to other targets.
#
#   See libea/CMakeLists.txt for a longer explanation of INTERFACE libraries.
#
# Docs: https://cmake.org/cmake/help/latest/command/add_library.html
# =============================================================================

add_library(libmkv INTERFACE)

# Expose the mkv/ headers to any target that links against libmkv.
# The $<BUILD_INTERFACE:...> form gives the path while building inside this
# source tree; $<INSTALL_INTERFACE:...> gives the path after installation.
target_include_directories(libmkv INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# libmkv headers use ealib types, so anything that uses libmkv also needs
# libea's headers (and Boost::headers transitively from libea).
target_link_libraries(libmkv INTERFACE libea)
