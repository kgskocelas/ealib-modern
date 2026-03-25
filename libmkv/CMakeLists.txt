# ---------------------------------------------------------------------------
# libmkv – header-only Markov network library
# ---------------------------------------------------------------------------
add_library(libmkv INTERFACE)
target_include_directories(libmkv INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(libmkv INTERFACE libea)
