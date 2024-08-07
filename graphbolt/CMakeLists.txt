cmake_minimum_required(VERSION 3.18)
project(graphbolt C CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(USE_CUDA)
  message(STATUS "Build graphbolt with CUDA support")
  enable_language(CUDA)
  add_definitions(-DGRAPHBOLT_USE_CUDA)
endif()

# For windows, define NOMINMAX to avoid conflict with std::min/max
if(MSVC)
  add_definitions(-DNOMINMAX)
endif()

# Find PyTorch cmake files and PyTorch versions with the python interpreter
# $PYTHON_INTERP ("python3" or "python" if empty)
if(NOT PYTHON_INTERP)
  find_program(PYTHON_INTERP NAMES python3 python)
endif()

message(STATUS "Using Python interpreter: ${PYTHON_INTERP}")

file(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/find_cmake.py FIND_CMAKE_PY)
execute_process(
  COMMAND ${PYTHON_INTERP} ${FIND_CMAKE_PY}
  OUTPUT_VARIABLE TORCH_PREFIX_VER
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

message(STATUS "find_cmake.py output: ${TORCH_PREFIX_VER}")
list(GET TORCH_PREFIX_VER 0 TORCH_PREFIX)
list(GET TORCH_PREFIX_VER 1 TORCH_VER)

message(STATUS "Configuring for PyTorch ${TORCH_VER}")
string(REPLACE "." ";" TORCH_VERSION_LIST ${TORCH_VER})

set(Torch_DIR "${TORCH_PREFIX}/Torch")
message(STATUS "Setting directory to ${Torch_DIR}")

find_package(Torch REQUIRED)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TORCH_C_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g3 -ggdb")

set(LIB_GRAPHBOLT_NAME "graphbolt_pytorch_${TORCH_VER}")

set(BOLT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(BOLT_INCLUDE "${CMAKE_CURRENT_SOURCE_DIR}/include")
file(GLOB BOLT_HEADERS ${BOLT_INCLUDE})
file(GLOB BOLT_SRC ${BOLT_DIR}/*.cc)
if(USE_CUDA)
  file(GLOB BOLT_CUDA_SRC
    ${BOLT_DIR}/cuda/*.cu
    ${BOLT_DIR}/cuda/*.cc
  )
  list(APPEND BOLT_SRC ${BOLT_CUDA_SRC})
  if(DEFINED ENV{CUDAARCHS})
    set(CMAKE_CUDA_ARCHITECTURES $ENV{CUDAARCHS})
  endif()
  set(CMAKE_CUDA_ARCHITECTURES_FILTERED ${CMAKE_CUDA_ARCHITECTURES})
  # CUDA extension supports only sm_70 and up (Volta+).
  list(FILTER CMAKE_CUDA_ARCHITECTURES_FILTERED EXCLUDE REGEX "[2-6][0-9]")
  list(LENGTH CMAKE_CUDA_ARCHITECTURES_FILTERED CMAKE_CUDA_ARCHITECTURES_FILTERED_LEN)
  if(CMAKE_CUDA_ARCHITECTURES_FILTERED_LEN EQUAL 0)
    # Build the CUDA extension at least build for Volta.
    set(CMAKE_CUDA_ARCHITECTURES_FILTERED "70")
  endif()
  set(LIB_GRAPHBOLT_CUDA_NAME "${LIB_GRAPHBOLT_NAME}_cuda")
endif()

add_library(${LIB_GRAPHBOLT_NAME} SHARED ${BOLT_SRC} ${BOLT_HEADERS})
include_directories(BEFORE ${BOLT_DIR}
                           ${BOLT_HEADERS}
                           "../third_party/pcg/include"
                           "../third_party/phmap")
target_link_libraries(${LIB_GRAPHBOLT_NAME} "${TORCH_LIBRARIES}")

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  if(USE_LIBURING)
    add_definitions(-DHAVE_LIBRARY_LIBURING)
    include(ExternalProject)
    set(LIBURING_INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/third_party/liburing)
    set(LIBURING_C_COMPILER "${CMAKE_C_COMPILER} -w")
    ExternalProject_Add(
      liburing
      SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../third_party/liburing
      CONFIGURE_COMMAND <SOURCE_DIR>/configure --cc=${LIBURING_C_COMPILER} --cxx=${CMAKE_CXX_COMPILER} --prefix=/
      # In order to avoid the error `error: redefinition of 'struct in6_pktinfo'` on ubi7
      # when building examples, let's build src only.
      BUILD_COMMAND bash -c "make -j 4 -C src/"
      BUILD_IN_SOURCE ON
      INSTALL_COMMAND make install DESTDIR=${LIBURING_INSTALL_DIR}
      BUILD_BYPRODUCTS ${LIBURING_INSTALL_DIR}/lib/liburing.a
      BUILD_BYPRODUCTS ${LIBURING_INSTALL_DIR}/include
      DOWNLOAD_EXTRACT_TIMESTAMP true
    )
    set(LIBURING_INCLUDE ${LIBURING_INSTALL_DIR}/include)
    set(LIBURING ${LIBURING_INSTALL_DIR}/lib/liburing.a)

    target_include_directories(${LIB_GRAPHBOLT_NAME} PRIVATE ${LIBURING_INCLUDE})
    add_dependencies(${LIB_GRAPHBOLT_NAME} liburing)
    target_link_libraries(${LIB_GRAPHBOLT_NAME} ${CMAKE_CURRENT_BINARY_DIR}/third_party/liburing/lib/liburing.a)
    message(STATUS "Build graphbolt with liburing.")
  endif(USE_LIBURING)
endif()

if(USE_CUDA)
  file(GLOB BOLT_CUDA_EXTENSION_SRC
    ${BOLT_DIR}/cuda/extension/*.cu
    ${BOLT_DIR}/cuda/extension/*.cc
  )
  # Until https://github.com/NVIDIA/cccl/issues/1083 is resolved, we need to
  # compile the cuda/extension folder with Volta+ CUDA architectures.
  add_library(${LIB_GRAPHBOLT_CUDA_NAME} STATIC ${BOLT_CUDA_EXTENSION_SRC} ${BOLT_HEADERS})
  target_link_libraries(${LIB_GRAPHBOLT_CUDA_NAME} "${TORCH_LIBRARIES}")

  set_target_properties(${LIB_GRAPHBOLT_NAME} PROPERTIES CUDA_STANDARD 17)
  set_target_properties(${LIB_GRAPHBOLT_CUDA_NAME} PROPERTIES CUDA_STANDARD 17)
  set_target_properties(${LIB_GRAPHBOLT_CUDA_NAME} PROPERTIES CUDA_ARCHITECTURES "${CMAKE_CUDA_ARCHITECTURES_FILTERED}")
  set_target_properties(${LIB_GRAPHBOLT_CUDA_NAME} PROPERTIES POSITION_INDEPENDENT_CODE TRUE)
  message(STATUS "Use external CCCL library for a consistent API and performance for graphbolt.")
  include_directories(BEFORE
                      "../third_party/cccl/thrust"
                      "../third_party/cccl/cub"
                      "../third_party/cccl/libcudacxx/include"
                      "../third_party/cuco/include")

  message(STATUS "Use HugeCTR gpu_cache for graphbolt with INCLUDE_DIRS $ENV{GPU_CACHE_INCLUDE_DIRS}.")
  target_include_directories(${LIB_GRAPHBOLT_NAME} PRIVATE $ENV{GPU_CACHE_INCLUDE_DIRS})
  target_link_directories(${LIB_GRAPHBOLT_NAME} PRIVATE ${GPU_CACHE_BUILD_DIR})
  target_link_libraries(${LIB_GRAPHBOLT_NAME} gpu_cache)
  
  get_property(archs TARGET ${LIB_GRAPHBOLT_NAME} PROPERTY CUDA_ARCHITECTURES)
  message(STATUS "CUDA_ARCHITECTURES for graphbolt: ${archs}")

  get_property(archs TARGET ${LIB_GRAPHBOLT_CUDA_NAME} PROPERTY CUDA_ARCHITECTURES)
  message(STATUS "CUDA_ARCHITECTURES for graphbolt extension: ${archs}")

  target_link_libraries(${LIB_GRAPHBOLT_NAME} ${LIB_GRAPHBOLT_CUDA_NAME})
endif()

# The Torch CMake configuration only sets up the path for the MKL library when
# using the conda distribution. The following is a workaround to address this
# when using a standalone installation of MKL.
if(DEFINED MKL_LIBRARIES)
  target_link_directories(${LIB_GRAPHBOLT_NAME} PRIVATE
                          ${MKL_ROOT}/lib/${MKL_ARCH})
endif()
