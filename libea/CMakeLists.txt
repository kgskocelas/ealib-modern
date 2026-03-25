# ---------------------------------------------------------------------------
# libea – header-only interface target
# ---------------------------------------------------------------------------
# The evolutionary algorithm core is entirely header-only.  Downstream
# targets that link against `libea` get the include path automatically.
add_library(libea INTERFACE)
target_include_directories(libea INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(libea INTERFACE
    Boost::headers
)

# ---------------------------------------------------------------------------
# libea_cmdline – compiled portion (command-line interface + expansion)
# ---------------------------------------------------------------------------
add_library(libea_cmdline STATIC
    src/cmdline_interface.cpp
    src/expansion.cpp
)
target_include_directories(libea_cmdline PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
target_link_libraries(libea_cmdline PUBLIC
    libea
    Boost::serialization
    Boost::iostreams
    Boost::regex
    Boost::system
    Boost::filesystem
    Boost::program_options
    Boost::timer
    Boost::chrono
    ZLIB::ZLIB
)

# ---------------------------------------------------------------------------
# libea_runner – thin wrapper that provides main()
# ---------------------------------------------------------------------------
add_library(libea_runner STATIC
    src/main.cpp
)
target_link_libraries(libea_runner PUBLIC
    libea_cmdline
)
