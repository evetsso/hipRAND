# MIT License
#
# Copyright (c) 2018-2023 Advanced Micro Devices, Inc. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.16 FATAL_ERROR)


if(DEFINED ENV{ROCM_PATH})
  set (ROCM_PATH "$ENV{ROCM_PATH}" CACHE PATH "The path to the ROCm installation.")
endif()
set (ROCM_PATH "/opt/rocm" CACHE PATH "The path to the ROCm installation.")

# Install prefix
if (WIN32)
  SET( CMAKE_INSTALL_PREFIX "C:/hipSDK" CACHE PATH "Install path" FORCE )
  SET( INSTALL_PREFIX "C:/hipSDK" )
  SET( CPACK_SET_DESTDIR FALSE )
  SET( CPACK_PACKAGE_INSTALL_DIRECTORY "C:/hipSDK" )
  SET( CPACK_PACKAGING_INSTALL_PREFIX "" )
  set( CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF )
else()
  set(CMAKE_INSTALL_PREFIX "${ROCM_PATH}" CACHE PATH "Install path prefix, prepended onto install directories")

  #Adding CMAKE_PREFIX_PATH
  list( APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/llvm ${ROCM_PATH} )
  if(NOT CPACK_PACKAGING_INSTALL_PREFIX)
      set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
  endif()
endif()

#
# hipRAND project
#
project(hipRAND CXX)
set(hipRAND_VERSION "2.10.16")

# Build options
option(BUILD_FORTRAN_WRAPPER "Build Fortran wrapper" OFF)
option(BUILD_TEST "Build tests (requires googletest)" OFF)
option(BUILD_BENCHMARK "Build benchmarks" OFF)
option(BUILD_ADDRESS_SANITIZER "Build with address sanitizer enabled" OFF)
option(DOWNLOAD_ROCRAND "Download rocRAND if it is not found in the default install directory or the manually-specified directory" OFF)
set(ROCRAND_PATH "" CACHE PATH "Path to rocRAND install.")

# CMake modules
list(APPEND CMAKE_MODULE_PATH
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules
    ${HIP_DIR}/cmake ${ROCM_PATH}/hip/cmake # FindHIP.cmake
    ${ROCM_PATH}/llvm
)

# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "" "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE CACHE BOOL "Add paths to linker search and installed rpath")

# Set the default value of BUILD_SHARED_LIBS
set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared")

# Include cmake scripts
include(cmake/Dependencies.cmake)

# Detect compiler support for target ID
# This section is deprecated. Please use rocm_check_target_ids for future use.
if( CMAKE_CXX_COMPILER MATCHES ".*/hipcc$" )
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} "--help"
        OUTPUT_VARIABLE CXX_OUTPUT
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_STRIP_TRAILING_WHITESPACE)
    string(REGEX MATCH ".mcode\-object\-version" TARGET_ID_SUPPORT ${CXX_OUTPUT})
endif()

# Verify that hipcc or HIP-aware clang is used on ROCM platform
# TODO: Fix VerifyCompiler for Windows
if (NOT WIN32)
  include(cmake/VerifyCompiler.cmake)
else()
  list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH} ${ROCM_PATH}/hip ${ROCM_PATH}/llvm)
  find_package(hip REQUIRED CONFIG PATHS ${HIP_DIR} ${ROCM_PATH})
endif()

# Build option to disable -Werror
option(DISABLE_WERROR "Disable building with Werror" ON)

# Build CXX flags
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
if(DISABLE_WERROR)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Werror")
endif()

# FOR HANDLING ENABLE/DISABLE OPTIONAL BACKWARD COMPATIBILITY for FILE/FOLDER REORG
option(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY "Build with file/folder reorg with backward compatibility enabled" OFF)


# HIP on Windows: xhip is required with clang++ to get __half defined
if (WIN32)
  add_compile_options(-xhip)
endif()

if(BUILD_ADDRESS_SANITIZER AND BUILD_SHARED_LIBS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -shared-libasan")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -shared-libasan")
  add_link_options(-fuse-ld=lld)
endif()

# FOR HANDLING ENABLE/DISABLE OPTIONAL BACKWARD COMPATIBILITY for FILE/FOLDER REORG
option(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY "Build with file/folder reorg with backward compatibility enabled" ON)

# Set version variables
rocm_setup_version( VERSION "${hipRAND_VERSION}" )
set ( hiprand_VERSION ${hipRAND_VERSION} )
# Old-style version number used within the library's API. rocrand_get_version should be modified.
math(EXPR hiprand_VERSION_NUMBER "${hipRAND_VERSION_MAJOR} * 100000 + ${hipRAND_VERSION_MINOR} * 100 + ${hipRAND_VERSION_PATCH}")
set(hiprand_SOVERSION 1.1)

# Print configuration summary
include(cmake/Summary.cmake)
print_configuration_summary()

# hipRAND library
# This adds library targets: hiprand,
# also includes Fortran wrapper
add_subdirectory(library)

# Tests
if(BUILD_TEST)
    enable_testing()
    rocm_package_setup_component(clients)
    rocm_package_setup_client_component(tests)
    add_subdirectory(test)
endif()

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
if(WIN32)
    set(CPACK_SOURCE_GENERATOR "ZIP")
    set(CPACK_GENERATOR "ZIP")
    set(CMAKE_INSTALL_PREFIX "C:/hipSDK" CACHE PATH "Install path" FORCE)
    set(INSTALL_PREFIX "C:/hipSDK")
    set(CPACK_SET_DESTDIR OFF)
    set(CPACK_PACKAGE_INSTALL_DIRECTORY "C:/hipSDK")
    set(CPACK_PACKAGING_INSTALL_PREFIX "")
    set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
else()
    if(NOT CPACK_PACKAGING_INSTALL_PREFIX)
        set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
    endif()
endif()

if( HIP_RUNTIME_LOWER STREQUAL "rocclr" )
    rocm_package_add_dependencies("hip-rocclr >= 3.5.0")
elseif( HIP_RUNTIME STREQUAL "cuda" )
    rocm_package_add_dependencies("hip-nvcc >= 3.5.0")
endif( )
rocm_package_add_dependencies("rocrand >= ${hipRAND_VERSION}")

set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "\${CPACK_PACKAGING_INSTALL_PREFIX}" "\${CPACK_PACKAGING_INSTALL_PREFIX}/include")

if (BUILD_WITH_LIB STREQUAL "CUDA")
    set(package_name hiprand-alt)
else()
    set(package_name hiprand)
endif()

set(HIPRAND_CONFIG_DIR "\${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}" CACHE PATH "Path placed into ldconfig file")

rocm_create_package(
        NAME ${package_name}
        DESCRIPTION "Radeon Open Compute RAND library"
        MAINTAINER "hipRAND Maintainer <hiprand-maintainer@amd.com>"
        LDCONFIG
        LDCONFIG_DIR ${HIPRAND_CONFIG_DIR}
)
