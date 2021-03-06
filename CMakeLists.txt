cmake_minimum_required(VERSION 3.9)

project(helpers)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build configuration")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS Debug Release)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_BUILD_TYPE}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_BUILD_TYPE}/lib)

# Create an interface library to link against for compiler warnings
add_library(project_warnings INTERFACE)
include(cmake/CompilerWarnings.cmake)
set_project_warnings(project_warnings)

# ccache to speed up builds
option(ENABLE_CCACHE "Enable ccache to speed up builds" ON)
if(ENABLE_CCACHE)
  find_program(CCACHE ccache)
  if(CCACHE)
    message(STATUS "Found ccache: ${CCACHE}")
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CMAKE_COMMAND} -E env CCACHE_BASEDIR=${PROJECT_SOURCE_DIR} ${CCACHE})
    set(CMAKE_C_COMPILER_LAUNCHER ${CMAKE_COMMAND} -E env CCACHE_BASEDIR=${PROJECT_SOURCE_DIR} ${CCACHE})
  else()
    message(WARNING "ccache not found")
  endif()
endif()

# link time optimization/interprocedural optimization
option(ENABLE_LINK_TIME_OPTIMIZATION "Optimize during link time" ON)
if(ENABLE_LINK_TIME_OPTIMIZATION)
  cmake_policy(SET CMP0069 NEW)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT IsIpoSupported)
  if(IsIpoSupported)
      set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  else()
      message(WARNING "IPO Not Supported")
  endif()
endif()

include(cmake/CodeCoverage.cmake)
add_library(code_coverage INTERFACE)
enable_code_coverage(code_coverage)

include(cmake/Sanitizers.cmake)
add_library(code_sanitizers INTERFACE)
enable_compiler_sanitizers(code_sanitizers)

# Use GTest::Main and GTest::GTest when linking libraries
find_package(GTest)
if(GTEST_FOUND)
    include(cmake/CompileTests.cmake)
    include (CTest)
else()
    message(WARNING "GTest not found in system path. Not building unit tests.")
endif()

# use benchmark::benchmark when linking libraries
find_package(benchmark)
if(benchmark_FOUND)
    set(BUILD_BENCHMARKS ON)
else()
    message(WARNING "benchmark not found in system path. Not building benchmarks.")
endif()

if(BUILD_TESTING)
    add_subdirectory(test)
endif()

if(BUILD_BENCHMARKS)
    add_subdirectory(benchmark)
endif()
