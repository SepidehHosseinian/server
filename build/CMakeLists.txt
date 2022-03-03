# Copyright (c) 2019-2021, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required (VERSION 3.18)
project (triton-inference-server LANGUAGES C CXX)

include(CMakeDependentOption)
include(ExternalProject)
include(GNUInstallDirs)

# Backends
option(TRITON_ENABLE_TENSORRT "Include TensorRT backend in server" OFF)
option(TRITON_ENABLE_TENSORFLOW "Include TensorFlow backend in server" OFF)
option(TRITON_ENABLE_ONNXRUNTIME "Include ONNXRuntime backend in server" OFF)
option(TRITON_ENABLE_ONNXRUNTIME_TENSORRT
  "Enable TensorRT execution provider for ONNXRuntime backend in server" OFF)
option(TRITON_ENABLE_ONNXRUNTIME_OPENVINO
  "Enable OpenVINO execution provider for ONNXRuntime backend in server" OFF)
option(TRITON_ENABLE_PYTORCH "Include PyTorch backend in server" OFF)
option(TRITON_ENABLE_PYTHON "Include Python backend in server" OFF)
option(TRITON_ENABLE_ENSEMBLE "Include ensemble support in server" OFF)

# Endpoints
option(TRITON_ENABLE_HTTP "Include HTTP API in server" ON)
option(TRITON_ENABLE_GRPC "Include GRPC API in server" ON)
option(TRITON_ENABLE_SAGEMAKER "Include AWS SageMaker API in server" OFF)
option(TRITON_ENABLE_VERTEX_AI "Include Vertex AI API in server" OFF)
option(TRITON_ENABLE_METRICS "Include metrics support in server" ON)
option(TRITON_ENABLE_METRICS_GPU "Include GPU metrics support in server" ON)

# Cloud storage
option(TRITON_ENABLE_GCS "Include GCS Filesystem support in server" OFF)
option(TRITON_ENABLE_S3 "Include S3 Filesystem support in server" OFF)
option(TRITON_ENABLE_AZURE_STORAGE "Include Azure Storage Filesystem support in server" OFF)

# Multiple paths may be specified by separating them with semicolon
set(TRITON_ONNXRUNTIME_INCLUDE_PATHS "" CACHE PATH "Paths to ONNXRuntime includes")
set(TRITON_PYTORCH_INCLUDE_PATHS "" CACHE PATH "Paths to PyTorch includes")
# Used for the case where TensorRT is not installed under default search path,
# i.e. Windows build, this flag should be able to remove once TensorRT backend
# is ported to use backend API
set(TRITON_TENSORRT_INCLUDE_PATHS "" CACHE PATH "Paths to TensorRT includes")
set(TRITON_EXTRA_LIB_PATHS "" CACHE PATH "Extra library paths for Triton Server build")

option(TRITON_ENABLE_LOGGING "Include logging support in server" ON)
option(TRITON_ENABLE_STATS "Include statistics collections in server" ON)
option(TRITON_ENABLE_TRACING "Include tracing support in server" OFF)
option(TRITON_ENABLE_NVTX "Include NVTX support in server" OFF)
option(TRITON_ENABLE_ASAN "Build with address sanitizer" OFF)
option(TRITON_ENABLE_GPU "Enable GPU support in server" ON)
option(TRITON_ENABLE_MALI_GPU "Enable Arm Mali GPU support in server" OFF)
set(TRITON_MIN_COMPUTE_CAPABILITY "6.0" CACHE STRING
    "The minimum CUDA compute capability supported by Triton" )

# Repo tags
set(TRITON_COMMON_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/common repo")
set(TRITON_CORE_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/core repo")
set(TRITON_BACKEND_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/backend repo")
set(TRITON_THIRD_PARTY_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/third_party repo")

# Version
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/../TRITON_VERSION" TRITON_VERSION)

if(TRITON_ENABLE_METRICS AND NOT TRITON_ENABLE_STATS)
  message(FATAL_ERROR "TRITON_ENABLE_METRICS=ON requires TRITON_ENABLE_STATS=ON")
endif()

if(TRITON_ENABLE_TRACING AND NOT TRITON_ENABLE_STATS)
  message(FATAL_ERROR "TRITON_ENABLE_TRACING=ON requires TRITON_ENABLE_STATS=ON")
endif()

if(TRITON_ENABLE_TENSORRT AND NOT TRITON_ENABLE_GPU)
  message(FATAL_ERROR "TRITON_ENABLE_TENSORRT=ON requires TRITON_ENABLE_GPU=ON")
endif()

if (TRITON_ENABLE_METRICS_GPU AND NOT TRITON_ENABLE_METRICS)
  message(FATAL_ERROR "TRITON_ENABLE_METRICS_GPU=ON requires TRITON_ENABLE_METRICS=ON")
endif()

if (TRITON_ENABLE_METRICS_GPU AND NOT TRITON_ENABLE_GPU)
  message(FATAL_ERROR "TRITON_ENABLE_METRICS_GPU=ON requires TRITON_ENABLE_GPU=ON")
endif()

if(TRITON_ENABLE_ONNXRUNTIME_TENSORRT AND NOT TRITON_ENABLE_ONNXRUNTIME)
  message(FATAL_ERROR "TRITON_ENABLE_ONNXRUNTIME_TENSORRT=ON requires TRITON_ENABLE_ONNXRUNTIME=ON")
endif()
if(TRITON_ENABLE_ONNXRUNTIME_TENSORRT AND NOT TRITON_ENABLE_TENSORRT)
  message(FATAL_ERROR "TRITON_ENABLE_ONNXRUNTIME_TENSORRT=ON requires TRITON_ENABLE_TENSORRT=ON")
endif()

if(TRITON_ENABLE_ONNXRUNTIME_OPENVINO AND NOT TRITON_ENABLE_ONNXRUNTIME)
  message(FATAL_ERROR "TRITON_ENABLE_ONNXRUNTIME_OPENVINO=ON requires TRITON_ENABLE_ONNXRUNTIME=ON")
endif()

if(TRITON_ENABLE_ASAN AND TRITON_ENABLE_GPU)
  message(FATAL_ERROR "TRITON_ENABLE_ASAN=ON requires TRITON_ENABLE_GPU=OFF")
endif()

#
# Dependencies
#
include(FetchContent)

FetchContent_Declare(
  repo-third-party
  GIT_REPOSITORY https://github.com/triton-inference-server/third_party.git
  GIT_TAG ${TRITON_THIRD_PARTY_REPO_TAG}
)
set(TRITON_THIRD_PARTY_INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/third-party)
set(TRITON_THIRD_PARTY_SRC_INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/third-party-src)
FetchContent_MakeAvailable(repo-third-party)

# If CMAKE_TOOLCHAIN_FILE is set, propagate that hint path to the external
# projects.
set(_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE "")
if (CMAKE_TOOLCHAIN_FILE)
  set(_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE "-DCMAKE_TOOLCHAIN_FILE:PATH=${CMAKE_TOOLCHAIN_FILE}")
endif()

# If VCPKG_TARGET_TRIPLET is set, propagate that hint path to the external
# projects.
set(_CMAKE_ARGS_VCPKG_TARGET_TRIPLET "")
if (VCPKG_TARGET_TRIPLET)
  set(_CMAKE_ARGS_VCPKG_TARGET_TRIPLET "-DVCPKG_TARGET_TRIPLET:STRING=${VCPKG_TARGET_TRIPLET}")
endif()

# If OPENSSL_ROOT_DIR is set, propagate that hint path to the external
# projects with OpenSSL dependency.
set(_CMAKE_ARGS_OPENSSL_ROOT_DIR "")
if (OPENSSL_ROOT_DIR)
  set(_CMAKE_ARGS_OPENSSL_ROOT_DIR "-DOPENSSL_ROOT_DIR:PATH=${OPENSSL_ROOT_DIR}")
endif()

# Location where protobuf-config.cmake will be installed varies by
# platform
if (WIN32)
  set(_FINDPACKAGE_PROTOBUF_CONFIG_DIR "${TRITON_THIRD_PARTY_INSTALL_PREFIX}/protobuf/cmake")
else()
  set(_FINDPACKAGE_PROTOBUF_CONFIG_DIR "${TRITON_THIRD_PARTY_INSTALL_PREFIX}/protobuf/lib/cmake/protobuf")
endif()

#
# Build Triton Server test utilities
#
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(TRITON_TEST_UTILS_INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/test-util/install)
else()
  set(TRITON_TEST_UTILS_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
endif()

ExternalProject_Add(test-util
  PREFIX test-util
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/test-util"
  BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/test-util"
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DProtobuf_DIR:PATH=${_FINDPACKAGE_PROTOBUF_CONFIG_DIR}
    -DGTEST_ROOT:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/googletest
    -DLibevent_DIR:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevent/lib/cmake/libevent
    -DCNMEM_PATH:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/cnmem
    -DTRITON_COMMON_REPO_TAG:STRING=${TRITON_COMMON_REPO_TAG}
    -DTRITON_CORE_REPO_TAG:STRING=${TRITON_CORE_REPO_TAG}
    -DTRITON_BACKEND_REPO_TAG:STRING=${TRITON_BACKEND_REPO_TAG}
    -DTRITON_ENABLE_GPU:BOOL=${TRITON_ENABLE_GPU}
    -DTRITON_MIN_COMPUTE_CAPABILITY:STRING=${TRITON_MIN_COMPUTE_CAPABILITY}
    -DTRITON_ENABLE_TENSORRT:BOOL=${TRITON_ENABLE_TENSORRT}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_TEST_UTILS_INSTALL_PREFIX}
  DEPENDS protobuf googletest cnmem libevent
)
