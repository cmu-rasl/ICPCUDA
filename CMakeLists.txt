cmake_minimum_required(VERSION 3.17)

project(CUDAICP LANGUAGES CUDA CXX)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/cmake")

# We are using FetchContent over git submodules and Add_ExternalProject as the
# most modern CMake variant of adding third party CMake libs.
include(FetchContent)

find_package(Eigen3 3.3.7 QUIET)
if(NOT EIGEN3-FOUND)
  message(STATUS "[ICPCUDA]:: Cloning and configuring Eigen 3.3.7")
  FetchContent_Declare(
    eigen
    URL https://gitlab.com/libeigen/eigen/-/archive/3.3.7/eigen-3.3.7.tar.gz
    URL_HASH MD5=9e30f67e8531477de4117506fe44669b)
  if(NOT eigen_POPULATED)
    FetchContent_Populate(eigen)
    set(EIGEN3_INCLUDE_DIR ${eigen_SOURCE_DIR})
    find_package(Eigen3 3.3.7 REQUIRED)
  endif()
endif()
message(
  STATUS "[ICPCUDA]:: Eigen3 include directory used is ${EIGEN3_INCLUDE_DIR}")

# Sophus
if(NOT Sophus_FOUND)
  message(STATUS "[ICPCUDA]:: Cloning and configuring Sophus.")
  FetchContent_Declare(
    sophus
    GIT_REPOSITORY https://github.com/strasdat/Sophus.git
    GIT_TAG master)
  if(NOT sophus_POPULATED)
    FetchContent_Populate(sophus)
    option(BUILD_TESTS "" OFF)
    option(BUILD_EXAMPLES "" OFF)
    find_package(Sophus REQUIRED)
  endif()
endif()

find_package(CUDA REQUIRED)

set(EIGEN3_INCLUDE_DIR "$ENV{EIGEN3_INCLUDE_DIR}")
if(NOT EIGEN3_INCLUDE_DIR)
  message(
    FATAL_ERROR
      "Please point the environment variable EIGEN3_INCLUDE_DIR to the include directory of your Eigen3 installation."
  )
endif()

file(GLOB srcs src/*.cpp)
file(GLOB cuda src/Cuda/*.cu)
file(GLOB containers src/Cuda/containers/*.cpp)

cuda_select_nvcc_arch_flags(ARCH_FLAGS "Auto")
message("ARCH_FLAGS are ${ARCH_FLAGS}")

# set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}  "-Xcompiler;-fPIC;-std=c++11")
# set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "--ftz=true;--prec-div=false;--prec-
# sqrt=false")
set(CUDA_PROPAGATE_HOST_FLAGS False)
set(EXTRA_COMPILE_OPTIONS $<$<COMPILE_LANGUAGE:CUDA>:--expt-relaxed-constexpr
                          --ftz=true --prec-div=false --prec-sqrt=false>)

add_library(cudaicp SHARED src/ICPOdometry.cpp ${cuda} ${cuda_objs}
                           ${containers})

target_compile_options(cudaicp PRIVATE ${EXTRA_COMPILE_OPTIONS})

target_link_libraries(
  cudaicp
  PUBLIC ${CUDA_CUDART_LIBRARY} ${CUDA_CUDA_LIBRARY} ${CUDA_cudadevrt_LIBRARY}
         ${Eigen_LIBRARIES} ${CUDA_LIBRARIES} Sophus::Sophus)
set_target_properties(cudaicp PROPERTIES INSTALL_RPATH_USE_LINK_PATH
         TRUE)
target_include_directories(
  cudaicp PUBLIC ${EIGEN3_INCLUDE_DIR} ${CUDA_INCLUDE_DIRS}
                 ${Sophus_INCLUDE_DIRS})
target_include_directories(
  cudaicp PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
                 $<INSTALL_INTERFACE:include>)

if(${CMAKE_SYSTEM_PROCESSOR} MATCHES "x86_64")
  set(CMAKE_CXX_FLAGS "-O3 -march=native -msse2 -msse3 -Wall")
  message("[ICPCUDA] x86 System Detected!")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm")
  set(CMAKE_CXX_FLAGS "-O3 -march=armv8-a -Wall")
  message("[ICPCUDA] Jetson TX1 System Detected!")
elseif(${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
  set(CMAKE_CXX_FLAGS "-O3 -march=armv8-a -Wall")
  message("[ICPCUDA] Jetson TX2 System Detected!")
endif()

find_package(Pangolin REQUIRED)
if(Pangolin_FOUND)
  add_executable(ICP src/ICP.cpp)
  target_link_libraries(ICP PRIVATE cudaicp ${Pangolin_LIBRARIES})
  target_include_directories(ICP PUBLIC ${Pangolin_INCLUDE_DIRS})
endif()

# Export package for use from the build tree
set (CUDAICP_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/include" )

include(CMakePackageConfigHelpers)
set(CUDAICP_CMAKE_EXPORT_DIR share/cmake/${PROJECT_NAME})

set_target_properties(cudaicp PROPERTIES EXPORT_NAME cudaicp)

install(
  TARGETS cudaicp
  EXPORT CUDAICPTargets
  LIBRARY DESTINATION lib)

install(EXPORT CUDAICPTargets NAMESPACE CUDAICP:: DESTINATION ${CUDAICP_CMAKE_EXPORT_DIR})

export(TARGETS cudaicp FILE CUDAICPTargets.cmake)
export(PACKAGE cudaicp)

configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/CUDAICPConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/CUDAICPConfig.cmake
  INSTALL_DESTINATION ${CUDAICP_CMAKE_EXPORT_DIR}
  PATH_VARS CUDAICP_INCLUDE_DIRS)

# Install cmake targets
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/CUDAICPConfig.cmake
        DESTINATION ${CUDAICP_CMAKE_EXPORT_DIR})

# Install header files
install(DIRECTORY include/ DESTINATION include)
