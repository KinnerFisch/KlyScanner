cmake_minimum_required(VERSION 3.20)
project(KlyScanner VERSION 1.0.0 LANGUAGES C CXX)

# Match KlyLogger's supported toolchains and optimized static-library builds.
if(WIN32 AND NOT MSVC)
    message(FATAL_ERROR "KlyScanner on Windows requires MSVC or clang-cl")
endif()
if(CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES Release CACHE STRING "Available build configurations" FORCE)
else()
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

foreach(required_file
        KlyLogger/CMakeLists.txt
        KlyLogger/fmt/include/fmt/format.h
        brigadier-cpp/brigadier.hpp
        replxx/CMakeLists.txt)
    if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${required_file}")
        message(FATAL_ERROR "Missing ${required_file}; run git submodule update --init --recursive")
    endif()
endforeach()

include(CheckIPOSupported)
check_ipo_supported(RESULT KLY_SCANNER_IPO_SUPPORTED OUTPUT KLY_SCANNER_IPO_ERROR LANGUAGES CXX)
if(NOT KLY_SCANNER_IPO_SUPPORTED)
    message(FATAL_ERROR "KlyScanner requires LTO: ${KLY_SCANNER_IPO_ERROR}")
endif()

set(BUILD_SHARED_LIBS OFF)
set(REPLXX_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(REPLXX_BUILD_PACKAGE OFF CACHE BOOL "" FORCE)
add_subdirectory(KlyLogger)
add_subdirectory(replxx)

# brigadier-cpp's own CMakeLists builds a standalone executable, not a library.
add_library(kly_scanner_brigadier STATIC brigadier-cpp/brigadier.cpp)
target_compile_features(kly_scanner_brigadier PUBLIC cxx_std_20)
target_include_directories(kly_scanner_brigadier PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/brigadier-cpp>
    $<INSTALL_INTERFACE:include>)
set_target_properties(kly_scanner_brigadier PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)

add_library(KlyScanner STATIC KlyScanner.cpp)
add_library(KlyScanner::KlyScanner ALIAS KlyScanner)
target_compile_features(KlyScanner PUBLIC cxx_std_20)
target_include_directories(KlyScanner PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>)
set_target_properties(KlyScanner PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)

if(MSVC)
    # KlyLogger provides both CRT variants; select the same runtime for all libraries.
    set(KLY_SCANNER_MSVC_RUNTIME MD CACHE STRING "MSVC runtime variant (MD or MT)")
    set_property(CACHE KLY_SCANNER_MSVC_RUNTIME PROPERTY STRINGS MD MT)
    if(NOT KLY_SCANNER_MSVC_RUNTIME MATCHES "^(MD|MT)$")
        message(FATAL_ERROR "KLY_SCANNER_MSVC_RUNTIME must be MD or MT")
    endif()
    if(KLY_SCANNER_MSVC_RUNTIME STREQUAL "MT")
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")
        set_property(TARGET KlyScanner kly_scanner_brigadier replxx PROPERTY MSVC_RUNTIME_LIBRARY MultiThreaded)
    else()
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDLL")
        set_property(TARGET KlyScanner kly_scanner_brigadier replxx PROPERTY MSVC_RUNTIME_LIBRARY MultiThreadedDLL)
    endif()
    target_compile_options(KlyScanner PUBLIC /utf-8)
    target_link_libraries(KlyScanner PUBLIC KlyLogger::KlyLogger_${KLY_SCANNER_MSVC_RUNTIME})
else()
    target_link_libraries(KlyScanner PUBLIC KlyLogger::KlyLogger)
endif()
target_link_libraries(KlyScanner PUBLIC kly_scanner_brigadier replxx::replxx)

include(GNUInstallDirs)
install(TARGETS KlyScanner kly_scanner_brigadier ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES KlyScanner.hpp brigadier-cpp/brigadier.hpp DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES LICENSE DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME KlyScanner-LICENSE)
install(FILES brigadier-cpp/LICENSE DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME brigadier-cpp-LICENSE)
install(FILES replxx/LICENSE.md DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME replxx-LICENSE.md)
