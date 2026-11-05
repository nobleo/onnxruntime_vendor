cmake_minimum_required(VERSION 3.10)
project(onnxruntime_vendor)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)

# Handle the case in which ONNX Runtime should not be vendored, for example in
# conda-forge/robostack where a recent enough onnxruntime package is available
# in the distribution and there is no need to vendor it.
# We use the AMENT_VENDOR_POLICY option proposed in
# https://github.com/ament/ament_cmake/pull/592 and that will be introduced at
# some point in ament_vendor for this purpose.
if(DEFINED AMENT_VENDOR_POLICY AND (AMENT_VENDOR_POLICY STREQUAL "NEVER_VENDOR" OR AMENT_VENDOR_POLICY STREQUAL "NEVER_VENDOR_IGNORE_SATISFIED_CHECK"))
  ament_export_dependencies(onnxruntime)
  ament_package()
  return()
endif()

include(FetchContent)

set(ORT_VERSION "1.31.1")
# aarch64 (Jetson) CUDA: no official GPU tarball exists, so the CUDA-enabled libraries
# are taken from the PyPI onnxruntime-gpu wheel and combined with the headers of the
# CPU tarball. The wheel URL must match ORT_VERSION (PyPI only has aarch64 GPU wheels
# since 1.29.0).
set(ORT_AARCH64_WHEEL_URL "https://files.pythonhosted.org/packages/7b/dc/0e7674377dd204170a4e077c719ee735a2eea6b7f102aa66873af74cced4/onnxruntime_gpu-1.31.1-cp312-cp312-manylinux_2_34_aarch64.whl")

# Detect system and architecture for appropriate asset selection (ROS 2 distro platforms)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|ARM64")
    set(ORT_ASSET_NAME "onnxruntime-linux-aarch64-${ORT_VERSION}.tgz")
    set(ORT_USE_AARCH64_WHEEL TRUE)
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
    set(ORT_ASSET_NAME "onnxruntime-linux-x64-gpu_cuda13-${ORT_VERSION}.tgz")
  else()
    message(FATAL_ERROR "Unsupported Linux architecture: ${CMAKE_SYSTEM_PROCESSOR}")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  message(FATAL_ERROR "onnxruntime_vendor: macOS (Darwin) is not supported.")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  message(FATAL_ERROR "onnxruntime_vendor: Windows is not supported.")
else()
  message(FATAL_ERROR "Unsupported operating system: ${CMAKE_SYSTEM_NAME}")
endif()

set(ORT_DOWNLOAD_URL "https://github.com/microsoft/onnxruntime/releases/download/v${ORT_VERSION}")
set(ORT_ASSET_URL "${ORT_DOWNLOAD_URL}/${ORT_ASSET_NAME}")

message(STATUS "Downloading ONNX Runtime prebuilt binary: ${ORT_ASSET_NAME}")

if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.24")
  set(_onnxruntime_timestamp DOWNLOAD_EXTRACT_TIMESTAMP TRUE)
endif()

FetchContent_Declare(
  onnxruntime_binary
  URL "${ORT_ASSET_URL}"
  # TODO: optionally add URL_HASH SHA256=<known_sha256>
  ${_onnxruntime_timestamp}
)
FetchContent_MakeAvailable(onnxruntime_binary)

if(ORT_USE_AARCH64_WHEEL)
  FetchContent_Declare(
    onnxruntime_wheel
    URL "${ORT_AARCH64_WHEEL_URL}"
    ${_onnxruntime_timestamp}
  )
  FetchContent_MakeAvailable(onnxruntime_wheel)
endif()

# Copy to a stable, version-named directory so cmake_install.cmake does not
# reference FetchContent _deps/ path (e.g. for Debian packaging).
set(ONNXRUNTIME_ROOT_DIR "${CMAKE_BINARY_DIR}/onnxruntime_prebuilt_${ORT_VERSION}")
if(NOT EXISTS "${ONNXRUNTIME_ROOT_DIR}")
  if(ORT_USE_AARCH64_WHEEL)
    file(COPY "${onnxruntime_binary_SOURCE_DIR}/include" DESTINATION "${ONNXRUNTIME_ROOT_DIR}")
    file(GLOB _ort_wheel_libs "${onnxruntime_wheel_SOURCE_DIR}/onnxruntime/capi/libonnxruntime*.so*")
    file(COPY ${_ort_wheel_libs} DESTINATION "${ONNXRUNTIME_ROOT_DIR}/lib")
    # The wheel only ships the versioned name; consumers link against libonnxruntime.so.
    file(CREATE_LINK "libonnxruntime.so.${ORT_VERSION}"
      "${ONNXRUNTIME_ROOT_DIR}/lib/libonnxruntime.so" SYMBOLIC)
  else()
    file(COPY "${onnxruntime_binary_SOURCE_DIR}/" DESTINATION "${ONNXRUNTIME_ROOT_DIR}")
  endif()
endif()

install(DIRECTORY "${ONNXRUNTIME_ROOT_DIR}/include/"
  DESTINATION opt/${PROJECT_NAME}/include
)

install(DIRECTORY "${ONNXRUNTIME_ROOT_DIR}/lib/"
  DESTINATION opt/${PROJECT_NAME}/lib
  USE_SOURCE_PERMISSIONS
  FILES_MATCHING
  PATTERN "*"
)

ament_export_include_directories(opt/${PROJECT_NAME}/include)

# Install environment hook to set LD_LIBRARY_PATH so that libonnxruntime.so and
# the dlopen'd libonnxruntime_providers_shared.so are found at runtime.
ament_environment_hooks("${CMAKE_CURRENT_SOURCE_DIR}/env-hooks/${PROJECT_NAME}.dsv")

ament_package(
  CONFIG_EXTRAS_POST "onnxruntime_vendor-extras.cmake"
)
