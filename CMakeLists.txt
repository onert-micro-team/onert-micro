# =============================================================================
# onert-micro - Top-level CMake Configuration
# =============================================================================

# Set minimum required CMake version
cmake_minimum_required(VERSION 3.15)

# Define project name and enable testing
project(onert-micro)
enable_testing()

# Set C++ standard to C++14
set(CMAKE_CXX_STANDARD 14)

# Configure RPATH settings for proper library linking
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH "$ORIGIN/../lib:$ORIGIN/")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Export compile commands for tools like clangd
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Set default target architecture, CPU, and OS if not defined
if (NOT DEFINED TARGET_ARCH)
    set(TARGET_ARCH "armv7em")
endif()

if (NOT DEFINED TARGET_CPU)
    set(TARGET_CPU "cortex-m7")
endif()

if (NOT DEFINED TARGET_OS)
    set(TARGET_OS "generic")
endif()

# Include utility functions and macros
include(${CMAKE_CURRENT_SOURCE_DIR}/infra/onert-micro/utils.cmake)

# Find Google Test package for testing support
nnas_find_package(GTest QUIET)

# Option to enable testing with Google Test
option(ENABLE_TEST "Build Tests using Google Test" ${GTest_FOUND})

if(${ENABLE_TEST} AND NOT ${GTest_FOUND})
    message(FATAL_ERROR "Google Test is required to enable test")
endif(${ENABLE_TEST} AND NOT ${GTest_FOUND})

# Option to enable code coverage analysis
option(ENABLE_COVERAGE "Build for coverage test" OFF)
if(${ENABLE_COVERAGE} AND NOT ${ENABLE_TEST})
    message(FATAL_ERROR "Test should be enabled to measure test coverage")
endif(${ENABLE_COVERAGE} AND NOT ${ENABLE_TEST})

if(${ENABLE_TEST})
    include(CTest)
endif(${ENABLE_TEST})

# =============================================================================
# Interface Libraries for Common Settings
# =============================================================================

# Create interface library for common compiler flags
add_library(onert_micro_common INTERFACE)
if(ENABLE_STRICT_BUILD)
    target_compile_options(onert_micro_common INTERFACE -Werror -Wall -Wextra -Wno-reorder)
endif(ENABLE_STRICT_BUILD)

# Create interface library for coverage analysis
add_library(onert_micro_coverage INTERFACE)
if(ENABLE_COVERAGE)
    target_compile_options(onert_micro_coverage INTERFACE -g -O0 -fprofile-arcs -ftest-coverage)
    target_link_libraries(onert_micro_coverage INTERFACE gcov)
endif(ENABLE_COVERAGE)

# =============================================================================
# ARM Toolchain Configuration
# =============================================================================

# Set ARM compiler tools
set(ARM_C_COMPILER "arm-none-eabi-gcc")
set(ARM_ASM_COMPILER "arm-none-eabi-gcc")
set(ARM_CXX_COMPILER "arm-none-eabi-g++")
set(ARM_OBJCOPY "arm-none-eabi-objcopy")

# Verify ARM compiler is available
find_program(ARM_C_COMPILER_PATH ${ARM_C_COMPILER})

if (NOT ARM_C_COMPILER_PATH)
    message(STATUS "Build onert-micro failed: ARM compiler is NOT FOUND")
endif ()

# Define schema target names
set(OM_CIRCLE_SCHEMA onert_micro_circle_schema)
set(OM_TRAININFO_SCHEMA onert_micro_traininfo_schema)

# =============================================================================
# Git Hash commit Header Generation
# =============================================================================

# Generate git_version.h
set(OM_GIT_VERSION_FILENAME "git_version.h")
set(OM_GIT_VERSION_TEMPLATE_FILENAME "git_version.h.in")

# Add command to generate git version header
add_custom_command(
        OUTPUT
        ${CMAKE_CURRENT_BINARY_DIR}/gen/${OM_GIT_VERSION_FILENAME}
        ALL
        COMMAND
        ${CMAKE_COMMAND}
        -D IN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/${OM_GIT_VERSION_TEMPLATE_FILENAME}
        -D OUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/gen/${OM_GIT_VERSION_FILENAME}
        -P ${CMAKE_CURRENT_SOURCE_DIR}/infra/onert-micro/cmake/CheckGit.cmake
        WORKING_DIRECTORY
        ${CMAKE_SOURCE_DIR}
)

# Ensure git version header is written to binary directory
file(WRITE ${CMAKE_BINARY_DIR}/${OM_GIT_VERSION_FILENAME})

# Custom target to generate version header
add_custom_target(
        generate_version_header
        ALL
        COMMAND
        ${CMAKE_COMMAND}
        -D IN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/${OM_GIT_VERSION_TEMPLATE_FILENAME}
        -D OUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/gen/${OM_GIT_VERSION_FILENAME}
        -P ${CMAKE_CURRENT_SOURCE_DIR}/infra/onert-micro/cmake/CheckGit.cmake
        WORKING_DIRECTORY
        ${CMAKE_SOURCE_DIR}
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/gen/${OM_GIT_VERSION_FILENAME}
)

# =============================================================================
# External Dependencies Configuration
# =============================================================================

# Configure external dependencies (local vs downloaded)
if (NOT_BUILD_EXTERNALS)
    message(STATUS "USE LOCAL EXTERNAL")

    set(ONERT_MICRO_EXTERNAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/externals")
    set(Flatbuffers_DIR "${ONERT_MICRO_EXTERNAL_DIR}")
    set(FlatBuffersSource_DIR "${Flatbuffers_DIR}")
    set(EXT_OVERLAY_DIR "${ONERT_MICRO_EXTERNAL_DIR}")
    set(GENERATED_INCLUDE_DIR "${ONERT_MICRO_EXTERNAL_DIR}/gen")

    add_subdirectory(externals)
else ()
    message(STATUS "DOWNLOAD AND BUILD EXTERNALS")

    nnas_find_package(FlatBuffers EXACT 23.5.26 QUIET)

    if (NOT FlatBuffers_FOUND)
        message(STATUS "Build onert-micro failed: FlatBuffers 23.5.26 NOT FOUND")
        return()
    endif (NOT FlatBuffers_FOUND)

    message(STATUS "FOUND FlatBuffers")

    # Configure Circle schema
    set(SCHEMA_FILE "${NNAS_PROJECT_SOURCE_DIR}/res/CircleSchema/0.8/circle_schema.fbs")

    # need patch to circle schema 0.8 to resolve SAM issue
    set(PATCH_FILE ${CMAKE_CURRENT_SOURCE_DIR}/remove_stablehlo_from_cir0.8.patch)
    execute_process(COMMAND ${CMAKE_COMMAND} -E chdir ${NNAS_PROJECT_SOURCE_DIR} git apply ${PATCH_FILE}
            WORKING_DIRECTORY "${NNAS_PROJECT_SOURCE_DIR}"
            )

    # NOTE Copy circle_schema.fbs as schema.fbs to generate "schema_generated.fbs" instead of "circle_schema_generated.fbs"
    add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/schema.fbs"
            COMMAND ${CMAKE_COMMAND} -E copy "${SCHEMA_FILE}" schema.fbs
            WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
            DEPENDS "${SCHEMA_FILE}"
            )

    # Generate FlatBuffers target for Circle schema
    FlatBuffers_Target(${OM_CIRCLE_SCHEMA}
            OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen/circle-generated/circle"
            INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen"
            SCHEMA_DIR "${CMAKE_CURRENT_BINARY_DIR}"
            SCHEMA_FILES "schema.fbs"
            )

    # Configure training info schema
    set(SCHEMA_FILE "${NNAS_PROJECT_SOURCE_DIR}/res/CircleSchema/training/circle_traininfo.fbs")

    # NOTE Copy circle_schema.fbs as schema.fbs to generate "schema_generated.fbs" instead of "circle_schema_generated.fbs"
    add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/traininfo.fbs"
            COMMAND ${CMAKE_COMMAND} -E copy "${SCHEMA_FILE}" traininfo.fbs
            WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
            DEPENDS "${SCHEMA_FILE}"
            )

    # Generate FlatBuffers target for training info schema
    FlatBuffers_Target(${OM_TRAININFO_SCHEMA}
            OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen/circle-generated/circle"
            INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen"
            SCHEMA_DIR "${CMAKE_CURRENT_BINARY_DIR}"
            SCHEMA_FILES "traininfo.fbs"
            )

    set(Flatbuffers_DIR "${CMAKE_CURRENT_BINARY_DIR}/../../overlay/lib/cmake/flatbuffers")
    set(EXT_OVERLAY_DIR "${CMAKE_CURRENT_BINARY_DIR}/../../overlay")
    set(GENERATED_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/gen")
endif ()

# =============================================================================
# Kernel Selection Configuration
# =============================================================================

# Choosing Kernel: reference mcu, optimized cmsisnn
if (NOT KERNELS)
    message(STATUS "KERNEL variable is not defined, default reference mcu kernels will be used")
    set(OM_PAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/onert-micro/include/pal/mcu")
elseif ("${KERNELS}" STREQUAL "mcu")
    message(STATUS "ONERT_MICRO will use reference mcu kernels")
    set(OM_PAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/onert-micro/include/pal/mcu")
elseif ("${KERNELS}" STREQUAL "cmsisnn")
    message(STATUS "ONERT_MICRO will use optimized cmsisnn kernels")
    set(OM_PAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/onert-micro/include/pal/cmsisnn")
else ()
    message(STATUS "Build onert-micro: FAILED (Non-existent kernel variable. Choose one of the following options: mcu, cmsisnn)")
    return()
endif ()

# Set ARM build options
set(CMAKE_ARM_OPTIONS
        -DTARGET_CPU=${TARGET_CPU}
        -DTARGET_ARCH=${TARGET_ARCH}
        "-DEXT_OVERLAY_DIR=${EXT_OVERLAY_DIR}"
        "-DFlatBuffersSource_DIR=${FlatBuffersSource_DIR}"
        "-DCMAKE_TOOLCHAIN_FILE=${NNAS_PROJECT_SOURCE_DIR}/infra/onert-micro/cmake/buildtool/config/arm-none-eabi-gcc.cmake"
        "-DOM_PAL_DIR=${OM_PAL_DIR}"
        -DC_COMPILER=${ARM_C_COMPILER}
        -DCXX_COMPILER=${ARM_CXX_COMPILER}
        -DASM_COMPILER=${ARM_ASM_COMPILER}
        -DOBJCOPY=${ARM_OBJCOPY}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DENABLE_TEST=OFF
        -DBUILD_GTEST=OFF
        "-DNNAS_ROOT=${NNAS_PROJECT_SOURCE_DIR}"
        "-DGENERATED_INCLUDE_DIR=${GENERATED_INCLUDE_DIR}"
        )

# Add build options based on configuration
if (NOT_BUILD_EXTERNALS)
    list(APPEND CMAKE_ARM_OPTIONS "-DNOT_BUILD_EXTERNALS=ON")
else ()
    list(APPEND CMAKE_ARM_OPTIONS "-DNOT_BUILD_EXTERNALS=OFF")
endif ()

if (ENABLE_ONERT_MICRO_TEST)
    message(STATUS "Enable tests")
    list(APPEND CMAKE_ARM_OPTIONS "-DENABLE_TEST=ON")
endif ()

if (ENABLE_ONERT_MICRO_TRAINING)
    message(STATUS "ENABLE TRAINING PART")
    add_definitions(-DENABLE_TRAINING)
    list(APPEND CMAKE_ARM_OPTIONS "-DENABLE_TRAINING=ON")
endif ()

if (DIS_QUANT)
    message(STATUS "ONERT-MICRO will not use part for QUANTIZED models")
    add_definitions(-DDIS_QUANT)
    list(APPEND CMAKE_ARM_OPTIONS "-DDIS_QUANT=ON")
endif ()

if (DIS_FLOAT)
    message(STATUS "ONERT-MICRO will not use part for FLOAT models")
    add_definitions(-DDIS_FLOAT)
    list(APPEND CMAKE_ARM_OPTIONS "-DDIS_FLOAT=ON")
endif ()

if (DIS_DYN_SHAPES)
    message(STATUS "ONERT-MICRO will not use dynamic shapes")
    add_definitions(-DDIS_DYN_SHAPES)
    list(APPEND CMAKE_ARM_OPTIONS "-DDIS_DYN_SHAPES=ON")
endif()

if (OM_MEMORY_ESTIMATE)
    message(STATUS "ONERT-MICRO will use memory estimation")
    add_definitions(-DOM_MEMORY_ESTIMATE)
    list(APPEND CMAKE_ARM_OPTIONS "-DOM_MEMORY_ESTIMATE=ON")
endif()

# =============================================================================
# Standalone ARM Build Configuration
# =============================================================================

# Set up standalone ARM build directory
set(MICRO_ARM_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/standalone_arm")
file(MAKE_DIRECTORY "${MICRO_ARM_BUILD_DIR}")

set(MICRO_ARM_BUILD_DEPENDENCY "${MICRO_ARM_BUILD_DIR}/CMakeCache.txt")

# Custom command to configure standalone ARM build
add_custom_command(
        OUTPUT "${MICRO_ARM_BUILD_DEPENDENCY}"
        COMMAND "${CMAKE_COMMAND}" "${CMAKE_CURRENT_SOURCE_DIR}/standalone" ${CMAKE_ARM_OPTIONS}
        WORKING_DIRECTORY "${MICRO_ARM_BUILD_DIR}"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/standalone/CMakeLists.txt"
        VERBATIM
)

# Custom target for ARM cmake configuration
add_custom_target(onert_micro_arm_cmake DEPENDS "${MICRO_ARM_BUILD_DEPENDENCY}")

# To remove GENERATE_KERNELS_LIST_FROM and KERNELS variable from cmake cache
unset(GENERATE_KERNELS_LIST_FROM CACHE)
unset(KERNELS CACHE)
unset(USE_STATIC_KERNEL CACHE)
unset(DIS_QUANT CACHE)
unset(DIS_FLOAT CACHE)
unset(OM_MEMORY_ESTIMATE CACHE)
unset(ENABLE_ONERT_MICRO_TEST CACHE)
unset(NOT_BUILD_EXTERNALS CACHE)

# Set ARM binary path
set(MICRO_ARM_BINARY "${MICRO_ARM_BUILD_DIR}/onert_micro/src/libonert_micro_dev.a")

# Custom command to build ARM binary
add_custom_command(
        OUTPUT "${MICRO_ARM_BINARY}"
        COMMAND "${CMAKE_MAKE_PROGRAM}" onert_micro_dev -j ${CPU_COUNT}
        WORKING_DIRECTORY "${MICRO_ARM_BUILD_DIR}"
        DEPENDS onert_micro_arm_cmake ${OM_CIRCLE_SCHEMA} ${OM_TRAININFO_SCHEMA}
        VERBATIM
)

# Custom target for ARM build
add_custom_target(onert_micro_arm DEPENDS "${MICRO_ARM_BINARY}")

# =============================================================================
# Subdirectory Builds
# =============================================================================

# Add evaluation driver and training configuration tool subdirectories
add_subdirectory(eval-driver)
add_subdirectory(training-configure-tool)

# Should be after add_subdirectory
unset(ENABLE_ONERT_MICRO_TRAINING CACHE)

# =============================================================================
# Example Build Configuration
# =============================================================================

# Example build
if (NOT DEFINED BUILD_EXAMPLE)
    return()
endif ()

#MBED OS QEMU build
nnas_find_package(MbedOSSource EXACT 6.15 QUIET)

if (NOT MbedOSSource_FOUND)
    message(STATUS "Skipping luci-micro: MbedOSSource not found")
    return()
endif ()

# Set up Mbed OS build directory
set(MBED_OS_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/examples/speech_recognition/DISCO_F746/mbed-os")
file(MAKE_DIRECTORY "${MBED_OS_BUILD_DIR}")

set(MBED_OS_BUILD_DEPENDENCY "${MBED_OS_BUILD_DIR}/CMakeCache.txt")

# Set source directory paths
set(ONERTMICRO_SRC_DIR "${NNAS_PROJECT_SOURCE_DIR}/onert-micro")

# Custom command to configure Mbed OS build
add_custom_command(
        OUTPUT "${MBED_OS_BUILD_DEPENDENCY}"
        COMMAND "${CMAKE_COMMAND}" "${CMAKE_CURRENT_SOURCE_DIR}/examples/speech_recognition/DISCO_F746/mbed-os"
        -DMICRO_ARM_BUILD_DIR=${MICRO_ARM_BUILD_DIR}
        -DMbedOSSource_DIR=${MbedOSSource_DIR}
        -DFlatBuffersSource_DIR=${FlatBuffersSource_DIR}
        -DONERTMICRO_SRC_DIR=${ONERTMICRO_SRC_DIR}
        WORKING_DIRECTORY "${MBED_OS_BUILD_DIR}"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/examples/speech_recognition/DISCO_F746/mbed-os/CMakeLists.txt"
        VERBATIM
)

# Custom target for Mbed OS cmake configuration
add_custom_target(mbed_os_cmake DEPENDS "${MBED_OS_BUILD_DEPENDENCY}")

# Set Mbed OS binary path
set(MBED_OS_BINARY "${MBED_OS_BUILD_DIR}/libmbed_os.a")

# Custom command to build Mbed OS
add_custom_command(
        OUTPUT "${MBED_OS_BINARY}"
        COMMAND "${CMAKE_MAKE_PROGRAM}" mbed_os -j ${CPU_COUNT}
        WORKING_DIRECTORY "${MBED_OS_BUILD_DIR}"
        DEPENDS mbed_os_cmake
        VERBATIM
)

# Custom target for Mbed OS build
add_custom_target(mbed_os_arm DEPENDS "${MBED_OS_BINARY}")

# Set example binary path
set(BUILD_EXAMPLE_BINARY "${MBED_OS_BUILD_DIR}/example_speech_recognition.bin")

# Custom command to build example
add_custom_command(
        OUTPUT "${BUILD_EXAMPLE_BINARY}"
        COMMAND "${CMAKE_MAKE_PROGRAM}" example_speech_recognition -j ${CPU_COUNT}
        WORKING_DIRECTORY "${MBED_OS_BUILD_DIR}"
        DEPENDS mbed_os_arm "${CMAKE_CURRENT_SOURCE_DIR}/examples/speech_recognition/DISCO_F746/mbed-os/main.cpp" ${MICRO_ARM_BINARY}
        VERBATIM
)

# Custom target for example build
add_custom_target(onert_micro_build_example_arm DEPENDS "${BUILD_EXAMPLE_BINARY}")
