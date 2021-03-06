project(EntityX)
cmake_minimum_required(VERSION 2.8)

if(NOT DEFINED CMAKE_MACOSX_RPATH)
  set(CMAKE_MACOSX_RPATH 0)
endif()

set(ENTITYX_MAJOR_VERSION 1)
set(ENTITYX_MINOR_VERSION 0)
set(ENTITYX_PATCH_VERSION 1)
set(ENTITYX_VERSION ${ENTITYX_MAJOR_VERSION}.${ENTITYX_MINOR_VERSION}.${ENTITYX_PATCH_VERSION})


include_directories(${CMAKE_CURRENT_LIST_DIR})

set(ENTITYX_BUILD_TESTING false CACHE BOOL "Enable building of tests.")
set(ENTITYX_RUN_BENCHMARKS false CACHE BOOL "Run benchmarks (in conjunction with -DENTITYX_BUILD_TESTING=1).")
set(ENTITYX_MAX_COMPONENTS 64 CACHE STRING "Set the maximum number of components.")
set(ENTITYX_DT_TYPE double CACHE STRING "The type used for delta time in EntityX update methods.")
set(ENTITYX_BUILD_SHARED true CACHE BOOL "Build shared libraries?")

include(${CMAKE_ROOT}/Modules/CheckIncludeFile.cmake)
include(CheckCXXSourceCompiles)

# Default compiler args
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic -Werror -Wall -Wextra -Wno-unused-parameter -Wno-error=unused-variable -Wno-error=sign-compare -std=c++11")
	set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g")
	set(CMAKE_CXX_FLAGS_MINSIZEREL "-g -Os -DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELEASE "-g -O2 -DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g")
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
	# /Zi - Produces a program database (PDB) that contains type information and symbolic debugging information for use with the debugger.
	# /FS - Allows multiple cl.exe processes to write to the same .pdb file
	# /DEBUG - Enable debug during linking
	# /Od - Disables optimization
	set(CMAKE_CXX_FLAGS_DEBUG "/Zi /FS /DEBUG /Od /MDd")
	# /Ox - Full optimization
	set(CMAKE_CXX_FLAGS_RELEASE "/Ox -DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "/Ox /Zi /FS /DEBUG")
endif()

# if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
#     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Weverything -Wno-c++98-compat -Wno-shadow -Wno-padded -Wno-missing-noreturn -Wno-global-constructors")
# endif()

# Library installation directory
if(NOT DEFINED CMAKE_INSTALL_LIBDIR)
    set(CMAKE_INSTALL_LIBDIR lib)
endif(NOT DEFINED CMAKE_INSTALL_LIBDIR)
set(libdir ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})

# C++11 feature checks
include(CheckCXX11Features.cmake)

set(OLD_CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
endif()
check_cxx_source_compiles(
"
#include <memory>

int main() {
    std::shared_ptr<int>();
}
"
ENTITYX_HAVE_CXX11_STDLIB
)

if (NOT ENTITYX_HAVE_CXX11_STDLIB)
    message("-- Not using -stdlib=libc++ (test failed to build)")
    set(CMAKE_CXX_FLAGS "${OLD_CMAKE_CXX_FLAGS}")
else ()
    message("-- Using -stdlib=libc++")
endif ()


# Misc features
check_include_file("stdint.h" HAVE_STDINT_H)

macro(require FEATURE_NAME MESSAGE_STRING)
    if (NOT ${${FEATURE_NAME}})
        message(FATAL_ERROR "${MESSAGE_STRING} required -- ${${FEATURE_NAME}}")
    else()
        message("--   ${MESSAGE_STRING} found")
    endif()
endmacro(require)

macro(create_test TARGET_NAME SOURCE)
    add_executable(${TARGET_NAME} ${SOURCE})
    target_link_libraries(
        ${TARGET_NAME}
        entityx
        ${ARGN}
        )
    add_test(${TARGET_NAME} ${TARGET_NAME})
endmacro()

if (NOT CMAKE_BUILD_TYPE)
    message("-- Defaulting to release build (use -DCMAKE_BUILD_TYPE:STRING=Debug for debug build)")
    set(CMAKE_BUILD_TYPE "Release")
endif()

message("-- Checking C++ features")
require(HAS_CXX11_AUTO "C++11 auto support")
require(HAS_CXX11_NULLPTR "C++11 nullptr support")
require(HAS_CXX11_RVALUE_REFERENCES "C++11 rvalue reference support")
#require(HAS_CXX11_CSTDINT_H "C++11 stdint support")
require(HAS_CXX11_VARIADIC_TEMPLATES "C++11 variadic templates")
require(HAS_CXX11_RVALUE_REFERENCES "C++11 rvalue references")
require(HAS_CXX11_LONG_LONG "C++11 long long")
require(HAS_CXX11_LONG_LONG "C++11 lambdas")

message("-- Checking misc features")
require(HAVE_STDINT_H "stdint.h")

# Things to install
set(install_libs entityx)

set(sources entityx/System.cc entityx/Event.cc entityx/Entity.cc entityx/help/Timer.cc entityx/help/Pool.cc)
add_library(entityx STATIC ${sources})

if (ENTITYX_BUILD_SHARED)
    message("-- Building shared libraries (-DENTITYX_BUILD_SHARED=0 to only build static librarires)")
    add_library(entityx_shared SHARED ${sources})
    target_link_libraries(
        entityx_shared
        )
    set_target_properties(entityx_shared PROPERTIES
        OUTPUT_NAME entityx
        VERSION ${ENTITYX_VERSION}
        SOVERSION ${ENTITYX_MAJOR_VERSION})
    list(APPEND install_libs entityx_shared)
endif (ENTITYX_BUILD_SHARED)

if (ENTITYX_BUILD_TESTING)
    enable_testing()
    create_test(pool_test entityx/help/Pool_test.cc)
    create_test(entity_test entityx/Entity_test.cc)
    create_test(event_test entityx/Event_test.cc)
    create_test(system_test entityx/System_test.cc)
    create_test(tags_component_test entityx/tags/TagsComponent_test.cc)
    create_test(dependencies_test entityx/deps/Dependencies_test.cc)
    if (ENTITYX_RUN_BENCHMARKS)
        message("-- Running benchmarks")
        create_test(benchmarks_test entityx/Benchmarks_test.cc)
    else ()
        message("-- Not running benchmarks (use -DENTITYX_RUN_BENCHMARKS=1 to enable)")
    endif ()
endif (ENTITYX_BUILD_TESTING)


configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/entityx/config.h.in
    ${CMAKE_CURRENT_SOURCE_DIR}/entityx/config.h
)


if (NOT WINDOWS OR CYGWIN)
    set(entityx_libs -lentityx)

    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/entityx.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/entityx.pc
        )

    install(
        FILES ${CMAKE_CURRENT_BINARY_DIR}/entityx.pc
        DESTINATION "${libdir}/pkgconfig"
        )
endif()

install(
    DIRECTORY "entityx"
    DESTINATION "include"
    FILES_MATCHING PATTERN "*.h"
    )

install(
    TARGETS ${install_libs}
    LIBRARY DESTINATION "${libdir}"
    ARCHIVE DESTINATION "${libdir}"
    )
