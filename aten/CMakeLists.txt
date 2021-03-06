if (CAFFE2_CMAKE_BUILDING_WITH_MAIN_REPO)
  if (NOT BUILD_ATEN)
    return()
  endif()
else()
  cmake_minimum_required(VERSION 3.0 FATAL_ERROR)
  include(CMakeDependentOption)
  option(USE_CUDA "Use CUDA" ON)
  option(USE_ROCM "Use ROCm" OFF)
  option(USE_CUDNN "Use cuDNN" ON)
  option(USE_MKLDNN "Use MKLDNN" ON)
  cmake_dependent_option(
      USE_CUDNN "Use cuDNN" ON
      "USE_CUDA" OFF)
  option(ATEN_NO_TEST "Do not build ATen test binaries" OFF)
  option(ATEN_NO_CONTRIB "Do not build ATen contrib" OFF)

  # Legacy options, which we will eventually remove
  cmake_dependent_option(
      NO_CUDA "Legacy no CUDA" OFF
      "USE_CUDA" ON)
  cmake_dependent_option(
      WITH_CUDNN "Legacy cuDNN" ON
      "USE_CUDNN" OFF)
  cmake_dependent_option(
      NO_MKLDNN "Legacy no MKLDNN" OFF
      "USE_MKLDNN" ON)

  # Flag for shared dependencies
  set(BUILD_ATEN ON)
endif()
if (NOT USE_CUDA)
  set(NO_CUDA ON)
endif()

# Create the project in all cases
project(ATen CXX C)

# Find modules
list(APPEND CMAKE_MODULE_PATH
  /usr/lib/x86_64-linux-gnu/
  ${CMAKE_CURRENT_SOURCE_DIR}/../cmake/Modules
  ${CMAKE_CURRENT_SOURCE_DIR}/../cmake/public
  ${CMAKE_CURRENT_SOURCE_DIR}/../cmake/Modules_CUDA_fix)
list(APPEND CMAKE_LIBRARY_PATH /usr/lib/x86_64-linux-gnu/)

cmake_policy(SET CMP0012 NEW)

# Polyfill for upstream FindCUDA
include(CMakeInitializeConfigs)

#############################################

set(ATen_CPU_SRCS)
set(ATen_CPU_TEST_SRCS)
set(ATen_CPU_INCLUDE)
set(ATen_THIRD_PARTY_INCLUDE)
set(ATen_CUDA_SRCS)
set(ATen_CUDA_TEST_SRCS)
set(ATen_CUDA_INCLUDE)
set(ATen_CPU_DEPENDENCY_LIBS)
set(ATen_CUDA_DEPENDENCY_LIBS)
set(ATen_PUBLIC_CUDA_DEPENDENCY_LIBS)
SET(ATEN_INSTALL_BIN_SUBDIR "bin" CACHE PATH "ATen install binary subdirectory")
SET(ATEN_INSTALL_LIB_SUBDIR "lib" CACHE PATH "ATen install library subdirectory")
SET(ATEN_INSTALL_INCLUDE_SUBDIR "include" CACHE PATH "ATen install include subdirectory")

if (NOT CAFFE2_CMAKE_BUILDING_WITH_MAIN_REPO)
  # ---[ Build variables set within the cmake tree
  include(../cmake/BuildVariables.cmake)
  set(CAFFE2_WHITELIST "" CACHE STRING "A whitelist file of files that one should build.")

  # ---[ Misc checks to cope with various compiler modes
  include(../cmake/MiscCheck.cmake)

  # External projects
  include(ExternalProject)

  # ---[ Utils
  # TODO: merge the following 3 files into cmake/public/utils.cmake.
  include(../cmake/Utils.cmake)
  include(../cmake/public/utils.cmake)

  # ---[ Dependencies
  include(../cmake/Dependencies.cmake)
  list(APPEND ATen_CPU_INCLUDE ${Caffe2_CPU_INCLUDE})
  list(APPEND ATen_CUDA_INCLUDE ${Caffe2_GPU_INCLUDE})
  list(APPEND ATen_CPU_DEPENDENCY_LIBS ${Caffe2_DEPENDENCY_LIBS})
  list(APPEND ATen_CUDA_DEPENDENCY_LIBS ${Caffe2_CUDA_DEPENDENCY_LIBS})
  list(APPEND ATen_PUBLIC_CUDA_DEPENDENCY_LIBS
    ${Caffe2_PUBLIC_CUDA_DEPENDENCY_LIBS})
endif()

if(NOT NO_CUDA)
  list(APPEND ATen_CUDA_INCLUDE ${CUDA_INCLUDE_DIRS})
endif()

set(TH_LINK_STYLE STATIC)
add_subdirectory(src/TH)
set(TH_CPU_INCLUDE
  # dense
  ${CMAKE_CURRENT_SOURCE_DIR}/src/TH
  ${CMAKE_CURRENT_SOURCE_DIR}/src/THC
  ${CMAKE_CURRENT_BINARY_DIR}/src/TH
  ${CMAKE_CURRENT_BINARY_DIR}/src/THC
  # sparse
  ${CMAKE_CURRENT_SOURCE_DIR}/src/THS
  ${CMAKE_CURRENT_SOURCE_DIR}/src/THCS
  ${CMAKE_CURRENT_BINARY_DIR}/src/THS
  ${CMAKE_CURRENT_BINARY_DIR}/src/THCS

  ${CMAKE_CURRENT_SOURCE_DIR}/src
  ${CMAKE_CURRENT_BINARY_DIR}/src
  ${CMAKE_BINARY_DIR}/aten/src)
list(APPEND ATen_CPU_INCLUDE ${TH_CPU_INCLUDE})
add_subdirectory(src/THNN)
add_subdirectory(src/THS)

# Find the HIP package, set the HIP paths, load the HIP CMake.
IF(USE_ROCM)
  include(LoadHIP)
  if (NOT PYTORCH_FOUND_HIP)
    MESSAGE(FATAL_ERROR
      "Could not find HIP installation")
  endif()
ENDIF()

IF(MSVC)
  # we want to respect the standard, and we are bored of those **** .
  ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE=1)
  LIST(APPEND CUDA_NVCC_FLAGS "-Xcompiler /wd4819 -Xcompiler /wd4503 -Xcompiler /wd4190 -Xcompiler /wd4244 -Xcompiler /wd4251 -Xcompiler /wd4275 -Xcompiler /wd4522")
ENDIF(MSVC)

if(USE_ROCM)
  SET(AT_CUDA_ENABLED 1)
  add_subdirectory(src/THC)
  add_subdirectory(src/THCUNN)
  add_subdirectory(src/THCS)
  message("ROCm is enabled.")
elseif(NOT NO_CUDA)
  SET(AT_CUDA_ENABLED 1)
  add_subdirectory(src/THC)
  add_subdirectory(src/THCUNN)
  add_subdirectory(src/THCS)
else()
  message("disabling CUDA because NO_CUDA is set")
  SET(AT_CUDA_ENABLED 0)
endif()

list(APPEND ATen_CPU_INCLUDE
  ${CMAKE_CURRENT_SOURCE_DIR}/src/THNN
  ${CMAKE_CURRENT_SOURCE_DIR}/src/THCUNN)

list(APPEND ATen_CPU_INCLUDE
  ${CMAKE_CURRENT_SOURCE_DIR}/src
  ${CMAKE_CURRENT_SOURCE_DIR}/../third_party/catch/single_include
  ${CMAKE_CURRENT_BINARY_DIR}/src/ATen)
add_subdirectory(src/ATen)

if(ATEN_NO_CONTRIB)
  message("disable contrib because ATEN_NO_CONTRIB is set")
else()
  add_subdirectory(contrib/data)
  add_subdirectory(contrib/meter)
endif()

if (CAFFE2_CMAKE_BUILDING_WITH_MAIN_REPO)
  # Pass source, includes, and libs to parent
  set(ATen_CPU_SRCS ${ATen_CPU_SRCS} PARENT_SCOPE)
  set(ATen_CUDA_SRCS ${ATen_CUDA_SRCS} PARENT_SCOPE)
  set(ATen_CPU_TEST_SRCS ${ATen_CPU_TEST_SRCS} PARENT_SCOPE)
  set(ATen_CUDA_TEST_SRCS ${ATen_CUDA_TEST_SRCS} PARENT_SCOPE)
  set(ATen_CPU_INCLUDE ${ATen_CPU_INCLUDE} PARENT_SCOPE)
  set(ATen_CUDA_INCLUDE ${ATen_CUDA_INCLUDE} PARENT_SCOPE)
  set(ATen_THIRD_PARTY_INCLUDE ${ATen_THIRD_PARTY_INCLUDE} PARENT_SCOPE)
  set(ATen_CPU_DEPENDENCY_LIBS ${ATen_CPU_DEPENDENCY_LIBS} PARENT_SCOPE)
  set(ATen_CUDA_DEPENDENCY_LIBS ${ATen_CUDA_DEPENDENCY_LIBS} PARENT_SCOPE)
endif()
