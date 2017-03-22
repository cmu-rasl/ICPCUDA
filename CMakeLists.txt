cmake_minimum_required(VERSION 2.6.0)

project(ICP)

find_package(Pangolin REQUIRED HINTS ../../Pangolin/build)
find_package(CUDA REQUIRED)
find_package(Sophus REQUIRED)

#remove this as soon as eigen is shipped with FindEigen.cmake
get_filename_component(EIGEN_ROOT "/usr/include/local/eigen3" PATH)
if(PKG_CONFIG_FOUND)
  pkg_check_modules(PC_EIGEN eigen3)
endif(PKG_CONFIG_FOUND)
find_path(EIGEN_INCLUDE_DIRS Eigen/Core
  HINTS ${PC_EIGEN_INCLUDEDIR} ${PC_EIGEN_INCLUDE_DIRS}
  "${EIGEN_ROOT}" "$ENV{EIGEN_ROOT}"
  PATHS "$ENV{PROGRAMFILES}/Eigen 3.0.0" "$ENV{PROGRAMW6432}/Eigen 3.0.0"
  "$ENV{PROGRAMFILES}/Eigen" "$ENV{PROGRAMW6432}/Eigen"
  PATH_SUFFIXES eigen3 include/eigen3 include)
find_package_handle_standard_args(eigen DEFAULT_MSG EIGEN_INCLUDE_DIRS)
set(EIGEN_DEFINITIONS ${EIGEN_DEFINITIONS} -DEIGEN_USE_NEW_STDVECTOR
  -DEIGEN_YES_I_KNOW_SPARSE_MODULE_IS_NOT_STABLE_YET)

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake)
include(${PROJECT_SOURCE_DIR}/cmake/CudaComputeTargetFlags.cmake)
include(${PROJECT_SOURCE_DIR}/cmake/CudaAutodetectCompute.cmake)

include_directories(${Pangolin_INCLUDE_DIRS})
include_directories(${CUDA_INCLUDE_DIRS})
include_directories(${EIGEN_INCLUDE_DIRS})
include_directories(${Sophus_INCLUDE_DIR})
include_directories(include)

file(GLOB srcs src/*.cpp)
file(GLOB cuda src/Cuda/*.cu)
file(GLOB containers src/Cuda/containers/*.cpp)

set(ICPCUDA_known_gpu_archs "20 30 32 35 37 50 52 53")
unset(CUDA_ARCH_BIN CACHE)

detect_installed_gpus(CUDA_ARCH_BIN)

set(CUDA_ARCH_PTX "" CACHE STRING "Specify 'virtual' PTX arch to build PTX intermediate code for. Example: 1.0 1.2 or 10 12")

APPEND_TARGET_ARCH_FLAGS()
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}  "-Xcompiler;-fPIC;-std=c++11")
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "--ftz=true;--prec-div=false;--prec-sqrt=false")

CUDA_COMPILE(cuda_objs ${cuda})
add_library(cudaicp SHARED src/ICPOdometry.cpp
  ${cuda} ${cuda_objs}
  ${containers})
target_link_libraries(cudaicp
  ${Pangolin_LIBRARIES}
  ${Eigen_LIBRARIES}
  ${CUDA_LIBRARIES}
  )

IF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64")
  set(CMAKE_CXX_FLAGS "-O3 -march=native -msse2 -msse3 -Wall -std=c++11")
  message("[ICPCUDA] x86 System Detected! Compute Architecture auto detected to be ${CUDA_ARCH_BIN}")
ELSEIF(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm")
  set(CMAKE_CXX_FLAGS "-O3 -march=armv8-a -Wall -std=c++11")
  message("[ICPCUDA] Jetson TX1 System Detected! Compute Architecture auto detected to be ${CUDA_ARCH_BIN}")
ENDIF()

add_executable(ICP
  src/ICP.cpp
  )
target_link_libraries(ICP cudaicp)
