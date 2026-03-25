# ---------------------------------------------------------------------------
# Helper: each example is an executable that links against libea_runner.
# ---------------------------------------------------------------------------
function(add_ealib_example target_name source_file)
    add_executable(${target_name} ${source_file})
    target_include_directories(${target_name} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
    )
    target_link_libraries(${target_name} PRIVATE libea_runner ${ARGN})
endfunction()

# Core examples (libea only)
add_ealib_example(example-all-ones  all_ones.cpp)
add_ealib_example(example-logic9    logic9.cpp)
add_ealib_example(example-lod       lod.cpp)
add_ealib_example(example-mp        metapopulation.cpp)
add_ealib_example(example-nsga2     nsga2.cpp)
add_ealib_example(example-qhfc      qhfc.cpp)

# Needs libann (neural network headers)
add_ealib_example(example-pole      pole_balancing.cpp libann)

# Needs libmkv (Markov network headers)
add_ealib_example(example-mkv       markov_network.cpp libmkv)
