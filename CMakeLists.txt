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
		KlyLogger/KlyLogger.cpp
		KlyLogger/KlyLogger.hpp
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

add_subdirectory(KlyLogger)
find_package(Threads REQUIRED)

# Mirror replxx's upstream source list, but compile it into each Scanner archive.
# Linking another STATIC target would not embed its objects in the .lib/.a.
file(GLOB KLY_SCANNER_REPLXX_SOURCES CONFIGURE_DEPENDS
	"${CMAKE_CURRENT_SOURCE_DIR}/replxx/src/*.cpp"
	"${CMAKE_CURRENT_SOURCE_DIR}/replxx/src/*.cxx")

function(kly_scanner_add_variant target_name runtime_library logger_target)
	add_library(${target_name} STATIC
		KlyScanner.cpp
		brigadier-cpp/brigadier.cpp
		${KLY_SCANNER_REPLXX_SOURCES})
	target_compile_features(${target_name} PUBLIC cxx_std_20)
	target_include_directories(${target_name} PUBLIC
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/brigadier-cpp>
		$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/replxx/include>
		$<INSTALL_INTERFACE:include>
		PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/replxx/src")
	target_compile_definitions(${target_name} PUBLIC REPLXX_STATIC)
	set_target_properties(${target_name} PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
	if(MSVC)
		# All objects in this archive must use the same CRT, including dependencies.
		set_property(TARGET ${target_name} PROPERTY MSVC_RUNTIME_LIBRARY ${runtime_library})
		target_compile_options(${target_name} PUBLIC /utf-8)
		target_compile_options(${target_name} PRIVATE /O2 /Ob3)
		target_compile_definitions(${target_name} PRIVATE _CRT_SECURE_NO_WARNINGS)
		# brigadier.hpp requests brigadier.lib, whose objects are already embedded.
		target_link_options(${target_name} INTERFACE /NODEFAULTLIB:brigadier.lib)
	endif()
	target_link_libraries(${target_name} PUBLIC ${logger_target} Threads::Threads)
endfunction()

if(MSVC)
	kly_scanner_add_variant(KlyScanner_MD MultiThreadedDLL KlyLogger::KlyLogger_MD)
	kly_scanner_add_variant(KlyScanner_MT MultiThreaded KlyLogger::KlyLogger_MT)
	add_library(KlyScanner::KlyScanner_MD ALIAS KlyScanner_MD)
	add_library(KlyScanner::KlyScanner_MT ALIAS KlyScanner_MT)
	set(KLY_SCANNER_TARGETS KlyScanner_MD KlyScanner_MT)

	# Keep the old generic alias configurable; both archives are always built.
	set(KLY_SCANNER_MSVC_RUNTIME MD CACHE STRING "CRT for KlyScanner::KlyScanner alias (MD or MT)")
	set_property(CACHE KLY_SCANNER_MSVC_RUNTIME PROPERTY STRINGS MD MT)
	if(NOT KLY_SCANNER_MSVC_RUNTIME MATCHES "^(MD|MT)$")
		message(FATAL_ERROR "KLY_SCANNER_MSVC_RUNTIME must be MD or MT")
	endif()
	add_library(KlyScanner::KlyScanner ALIAS KlyScanner_${KLY_SCANNER_MSVC_RUNTIME})
	add_custom_target(KlyScanner DEPENDS ${KLY_SCANNER_TARGETS})
else()
	kly_scanner_add_variant(KlyScanner "" KlyLogger::KlyLogger)
	add_library(KlyScanner::KlyScanner ALIAS KlyScanner)
	set(KLY_SCANNER_TARGETS KlyScanner)
endif()

include(GNUInstallDirs)
install(TARGETS ${KLY_SCANNER_TARGETS} ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES KlyScanner.hpp brigadier-cpp/brigadier.hpp replxx/include/replxx.hxx replxx/include/replxx.h
	DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES LICENSE DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME KlyScanner-LICENSE)
install(FILES brigadier-cpp/LICENSE DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME brigadier-cpp-LICENSE)
install(FILES replxx/LICENSE.md DESTINATION ${CMAKE_INSTALL_DATADIR}/licenses/KlyScanner RENAME replxx-LICENSE.md)
