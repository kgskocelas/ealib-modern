# ---------------------------------------------------------------------------
# libann – header-only artificial neural network library
# ---------------------------------------------------------------------------
add_library(libann INTERFACE)
target_include_directories(libann INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(libann INTERFACE libea)
