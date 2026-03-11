cmake_minimum_required(VERSION 3.15)
project(luci_interpreter)
enable_testing()

set(CMAKE_CXX_STANDARD 14)

# Set the project source directory for nnas
set(NNAS_PROJECT_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/.." CACHE INTERNAL "Where to find nnas top-level source directory")

# Set the module path to include the main project's cmake directories
set(CMAKE_MODULE_PATH "${NNAS_PROJECT_SOURCE_DIR}/infra/onert-micro/cmake/buildtool/config" ${CMAKE_MODULE_PATH})

# Include utility functions
include(${NNAS_PROJECT_SOURCE_DIR}/infra/onert-micro/utils.cmake)

# Find Google Test package for testing support
nnas_find_package(GTest QUIET)

# Option to enable testing with Google Test
option(ENABLE_TEST "Build Tests using Google Test" ${GTest_FOUND})

if(${ENABLE_TEST} AND NOT ${GTest_FOUND})
    message(FATAL_ERROR "Google Test is required to enable test")
endif(${ENABLE_TEST} AND NOT ${GTest_FOUND})

if(${ENABLE_TEST})
    include(CTest)
endif(${ENABLE_TEST})

# Create interface library for coverage analysis
add_library(onert_micro_coverage INTERFACE)

# Find FlatBuffers
nnas_find_package(FlatBuffers EXACT 23.5.26 QUIET)

if (NOT FlatBuffers_FOUND)
    message(STATUS "Build luci-interpreter failed: FlatBuffers 23.5.26 NOT FOUND")
    return()
endif ()

message(STATUS "FOUND FlatBuffers")

# Configure Circle schema 0.4
set(SCHEMA_FILE "${NNAS_PROJECT_SOURCE_DIR}/res/CircleSchema/0.4/circle_schema.fbs")

# Copy circle_schema.fbs as schema.fbs to generate "schema_generated.fbs" instead of "circle_schema_generated.fbs"
add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/schema.fbs"
    COMMAND ${CMAKE_COMMAND} -E copy "${SCHEMA_FILE}" schema.fbs
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
    DEPENDS "${SCHEMA_FILE}"
)

# Generate FlatBuffers target for Circle schema
set(CIRCLE_SCHEMA luci_micro_circle_schema)
FlatBuffers_Target(${CIRCLE_SCHEMA}
    OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen/circle-generated/circle"
    INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen"
    SCHEMA_DIR "${CMAKE_CURRENT_BINARY_DIR}"
    SCHEMA_FILES "schema.fbs"
)

set(GENERATED_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen")
set(LUCI_MICRO_CIRCLE_SCHEMA_TARGET luci_micro_circle_schema)

set(LUCI_INTERPRETER_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(LUCI_INTERPRETER_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(LUCI_INTERPRETER_PAL_COMMON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/pal/common")
if (NOT LUCI_INTERPRETER_PAL_DIR)
    set(LUCI_INTERPRETER_PAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/pal/mcu")
endif()

if (NOT LUCI_INTERPRETER_KERNELS_BUILD_LIST)
    set(KERNEL_REGISTER_FILE "${LUCI_INTERPRETER_PAL_DIR}/KernelsToBuild.lst")
else()
    set(KERNEL_REGISTER_FILE ${LUCI_INTERPRETER_KERNELS_BUILD_LIST})
endif()

if (NOT DEFINED CUSTOM_LUCI_INTERPRETER_SUFFIX)
    set(LUCI_INTERPRETER_SUFFIX "")
else()
    set(LUCI_INTERPRETER_SUFFIX ${CUSTOM_LUCI_INTERPRETER_SUFFIX})
endif()

if (DIS_QUANT)
    add_definitions(-DDIS_QUANT)
endif()

if (DIS_FLOAT)
    add_definitions(-DDIS_FLOAT)
endif()

if (ENABLE_TRAINING)
    add_definitions(-DENABLE_TRAINING)
endif()

add_compile_options(-fno-exceptions)
add_compile_options(-Os)

# AFAIK, this will enable leak sanitizer, too
if(ENABLE_SANITIZER)
    add_compile_options(-fsanitize=address)
    add_link_options(-fsanitize=address)
endif(ENABLE_SANITIZER)

add_subdirectory(src)
