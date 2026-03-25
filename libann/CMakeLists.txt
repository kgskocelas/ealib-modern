# =============================================================================
# libann/CMakeLists.txt  –  Build script for the neural network library
# =============================================================================
#
# WHAT IS LIBANN?
#   libann provides artificial neural network (ANN) implementations used
#   as phenotypes (the "body" or "brain") in neuroevolution experiments —
#   where evolution discovers the weights and topology of a neural network
#   that solves some task.  The pole-balancing example uses libann to
#   evolve a neural network controller.
#
#   The ann/ headers live under libann/include/ann/ and are #included by
#   examples like pole_balancing.cpp via:
#       #include <ann/basic_neural_network.h>
#
# HEADER-ONLY: no .cpp files, nothing to compile
#   libann is entirely template-based header code — there are no .cpp files
#   to compile.  We therefore declare it as an INTERFACE library, exactly
#   as we do for libmkv and the core libea target.
#
#   See libea/CMakeLists.txt for a longer explanation of INTERFACE libraries.
#
# Docs: https://cmake.org/cmake/help/latest/command/add_library.html
# =============================================================================

add_library(libann INTERFACE)

# Expose the ann/ headers to any target that links against libann.
target_include_directories(libann INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# libann headers use ealib types, so anything that uses libann also needs
# libea's headers (and Boost::headers transitively from libea).
target_link_libraries(libann INTERFACE libea)
