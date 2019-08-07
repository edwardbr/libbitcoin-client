# Copyright (c) 2017 The Bitcoin developers

cmake_minimum_required(VERSION 3.5)


set(CMAKE_VERBOSE_MAKEFILE ON)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)


include(HunterGate)

message(STATUS "[project] CMAKE_TOOLCHAIN_FILE: ${CMAKE_TOOLCHAIN_FILE}")

option(HUNTER_STATUS_DEBUG "Hunter debug info" ON)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.165.tar.gz"
    SHA1 "5a73f91df5f6109c0bb1104d0c0ee423f7bece79"
    LOCAL
)

set(HUNTER_KEEP_PACKAGE_SOURCES ON)
option(HUNTER_STATUS_DEBUG "Hunter debug info" ON)

project(libbitcoin-client VERSION 0.1)

include(CTest)

# Implement -Dwith-tests and declare with-tests.
#------------------------------------------------------------------------------
set( with-tests "yes" CACHE BOOL "Compile with unit tests." )

# Implement -Dwith-examples and declare with-examples.
#------------------------------------------------------------------------------
set( with-examples "yes" CACHE BOOL "Compile with examples." )

# Find secp256k1
#------------------------------------------------------------------------------

hunter_add_package(secp256k1)
find_package(secp256k1 CONFIG REQUIRED)

# Find boost
#------------------------------------------------------------------------------

hunter_add_package(Boost COMPONENTS 
    atomic
    chrono
    date_time
    filesystem
    iostreams
    log
    program_options
    regex
    system
    thread
    test)

find_package(Boost CONFIG REQUIRED 
    atomic
    chrono
    date_time
    filesystem
    iostreams
    log_setup
    log
    program_options
    regex
    system
    thread
    unit_test_framework)

hunter_add_package(libbitcoin-system)
find_package(libbitcoin-system CONFIG REQUIRED)

hunter_add_package(libbitcoin-protocol)
find_package(libbitcoin-protocol CONFIG REQUIRED)

hunter_add_package(ICU)
find_package(ICU CONFIG REQUIRED)

hunter_add_package(ZeroMQ)
find_package(ZeroMQ CONFIG REQUIRED)

add_library( ${PROJECT_NAME}
    "src/obelisk_client.cpp" )

# libsecp256k1 use a different set of flags.
target_compile_options(${PROJECT_NAME} PRIVATE
     "-Wall" 

    # Warn on extra stuff.
    "-Wextra" 

    # Be really annoying.
    "-Wpedantic" 

    # Disallow warning on style order of declarations.
    "-Wno-reorder" 

    # Suppress warning for incomplete field initialization.
    "-Wno-missing-field-initializers" 

    # Conform to style.
    "-Wno-missing-braces" 

    # Ignore comments within comments or commenting of backslash extended lines.
    "-Wno-comment" 

    # Clean up bx generated code.
    "-Wno-unused-parameter"
)


# Conflict in stdlib under clang.
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
     target_compile_options(${PROJECT_NAME} PRIVATE "-Wno-mismatched-tags") 
endif()

# Limit delays and warnings.
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
     target_compile_options(${PROJECT_NAME} PRIVATE "-fno-var-tracking-assignments" )
endif()

target_include_directories( ${PROJECT_NAME} PRIVATE include)

target_link_libraries( ${PROJECT_NAME}  
                            Boost::chrono
                            Boost::date_time
                            Boost::filesystem
                            Boost::iostreams
                            Boost::log_setup
                            Boost::log
                            Boost::program_options
                            Boost::regex
                            Boost::system
                            Boost::thread
                            Threads::Threads
                            secp256k1::secp256k1
                            libbitcoin-system::libbitcoin-system   
                            libbitcoin-protocol::libbitcoin-protocol                         
                            ICU::uc)


# Define libbitcoin-system-test project.
#------------------------------------------------------------------------------
if (with-tests)
    add_executable( ${PROJECT_NAME}-test
        "test/main.cpp"
        "test/obelisk_client.cpp" )

    add_test( NAME ${PROJECT_NAME}-test COMMAND ${PROJECT_NAME}-test
            --run_test=generated,obsolete,offline,config,stub
            --show_progress=no
            --detect_memory_leak=0
            --report_level=no
            --build_info=yes )

#     libbitcoin-system-test project specific include directories.
#------------------------------------------------------------------------------
    target_include_directories( ${PROJECT_NAME}-test PRIVATE
        "include" )

#     libbitcoin-system-test project specific libraries/linker flags.
#------------------------------------------------------------------------------
    target_link_libraries( ${PROJECT_NAME}-test
        ${PROJECT_NAME}
        Boost::unit_test_framework )
endif()

# Define libbitcoin-system-examples project.
#------------------------------------------------------------------------------
if (with-examples)
    add_executable( ${PROJECT_NAME}-examples
        "examples/console/client.cpp"
        "examples/console/client.hpp"
        "examples/console/main.cpp"
        "examples/console/read_line.cpp"
        "examples/console/read_line.hpp" )

    #     libbitcoin-system-examples project specific include directories.
    #------------------------------------------------------------------------------
    target_include_directories( ${PROJECT_NAME}-examples PRIVATE
        "include" )

    #     libbitcoin-system-examples project specific libraries/linker flags.
    #------------------------------------------------------------------------------
    target_link_libraries( ${PROJECT_NAME}-examples
        ${PROJECT_NAME} )

endif()

# Define console project.
#------------------------------------------------------------------------------
if (with-console)
    add_executable( console
        "examples/console/client.cpp"
        "examples/console/client.hpp"
        "examples/console/main.cpp"
        "examples/console/read_line.cpp"
        "examples/console/read_line.hpp" )

#     console project specific include directories.
#------------------------------------------------------------------------------
    target_include_directories( console PRIVATE
        "include" )

#     console project specific libraries/linker flags.
#------------------------------------------------------------------------------
    target_link_libraries( console ${PROJECT_NAME} )


# Define get_height project.
#------------------------------------------------------------------------------
    add_executable( get_height
        "examples/get_height/main.cpp" )

#     get_height project specific include directories.
#------------------------------------------------------------------------------
    target_include_directories( get_height PRIVATE "include" )

#     get_height project specific libraries/linker flags.
#------------------------------------------------------------------------------
    target_link_libraries( get_height
        ${PROJECT_NAME} )

endif()




set(pc_config "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc")
configure_file(${PROJECT_NAME}.pc.in ${pc_config} @ONLY)

#set(la_config "${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}.la")
#configure_file(${PROJECT_NAME}.la.in ${la_config} @ONLY)

# Installation (https://github.com/forexample/package-example) {

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    TARGETS ${PROJECT_NAME}
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

install(
    DIRECTORY include/
    DESTINATION "${include_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)

install(FILES ${pc_config} DESTINATION lib/pkgconfig)
install(FILES ${la_config} DESTINATION lib)
