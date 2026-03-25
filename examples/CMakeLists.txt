# =============================================================================
# examples/CMakeLists.txt  –  Build script for the ealib example programs
# =============================================================================
#
# This file builds the eight example evolutionary algorithm programs that
# ship with ealib.  Each example follows the same pattern:
#
#   1. One .cpp source file that defines the EA type and its configuration.
#   2. Link against libea_runner, which provides the main() function and
#      all the Boost-based command-line / checkpointing infrastructure.
#   3. Most examples only need libea.  Two need extra libraries:
#        pole_balancing.cpp  → also needs libann (neural network headers)
#        markov_network.cpp  → also needs libmkv (Markov network headers)
#
# =============================================================================


# -----------------------------------------------------------------------------
# Helper function: add_ealib_example
# -----------------------------------------------------------------------------
#
# WHAT IS A CMAKE FUNCTION?
#   A function in CMake works just like a function in any programming
#   language: you give it a name, declare what arguments it takes, and write
#   a block of commands that run whenever you call it.  This avoids
#   copy-pasting the same add_executable / target_link_libraries commands
#   eight times.
#
# ARGUMENTS:
#   target_name  – the name of the executable CMake will build
#                  (e.g., "example-all-ones")
#   source_file  – the .cpp file that contains the EA definition
#                  (e.g., "all_ones.cpp")
#   ${ARGN}      – any extra arguments passed after source_file.
#                  ${ARGN} is a special CMake variable that captures all
#                  "extra" arguments beyond the named ones.  We use it to
#                  pass optional extra libraries (libann, libmkv).
#
# Docs: https://cmake.org/cmake/help/latest/command/function.html
#       https://cmake.org/cmake/help/latest/command/add_executable.html
function(add_ealib_example target_name source_file)

    # Create an executable target from the given source file.
    add_executable(${target_name} ${source_file})

    # Add the examples/ directory itself to the include path so that
    # logic9.cpp can find logic9.h (which lives in the same folder).
    # PRIVATE means this include path is only for this target, not
    # propagated to anything that links against it.
    target_include_directories(${target_name} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
    )

    # Link against libea_runner (which transitively pulls in libea_cmdline,
    # libea, and all required Boost libraries), plus any extra libraries
    # passed as additional arguments (${ARGN}).
    #
    # PRIVATE: the linkage is only needed internally for this executable.
    # Unlike libraries, executables don't expose dependencies to other
    # targets (there's nothing downstream of an executable).
    target_link_libraries(${target_name} PRIVATE libea_runner ${ARGN})

endfunction()


# -----------------------------------------------------------------------------
# Core examples  (only need libea + libea_runner)
# -----------------------------------------------------------------------------

# Evolves a bitstring to be all 1s — the "Hello World" of evolutionary algorithms.
add_ealib_example(example-all-ones  all_ones.cpp)

# Evolves logic gates that compute a 9-input logic function.
add_ealib_example(example-logic9    logic9.cpp)

# Runs all-ones evolution and records the "line of descent" — the sequence
# of ancestor individuals leading to the best final solution.
add_ealib_example(example-lod       lod.cpp)

# Runs a metapopulation EA — a population of populations, where sub-populations
# evolve semi-independently and occasionally exchange individuals.
add_ealib_example(example-mp        metapopulation.cpp)

# Demonstrates NSGA-II, a classic multi-objective optimization algorithm that
# simultaneously optimizes for multiple competing fitness criteria.
# Reference: Deb et al. (2002), https://doi.org/10.1109/4235.996017
add_ealib_example(example-nsga2     nsga2.cpp)

# Demonstrates QHFC (Quasi-Hierarchical Fair Competition), an algorithm that
# maintains multiple fitness tiers to preserve diversity.
add_ealib_example(example-qhfc      qhfc.cpp)


# -----------------------------------------------------------------------------
# Examples that need extra libraries
# -----------------------------------------------------------------------------

# Evolves a neural network (libann) that balances a pole on a cart.
# This is the classic "cartpole" or "inverted pendulum" control benchmark.
# Requires libann for the neural network phenotype headers.
# Reference: Anderson (1989), https://doi.org/10.1109/72.80205
add_ealib_example(example-pole      pole_balancing.cpp  libann)

# Evolves a Markov network (libmkv) as a discrete-state probabilistic controller.
# Requires libmkv for the Markov network data structure headers.
add_ealib_example(example-mkv       markov_network.cpp  libmkv)
