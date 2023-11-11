cmake_minimum_required(VERSION 3.7)

project(argon2-gpu CXX)
set(BINARY_INSTALL_DIR /usr/local/bin)
set(LIBRARY_INSTALL_DIR /usr/local/lib)
set(INCLUDE_INSTALL_DIR /usr/local/include)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR})
if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++11")
endif()
if(WIN32)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)
    include(GNUInstallDirs)
endif()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CUDA_ARCH "sm_75" CACHE STRING "Specify the CUDA architecture.")

set(CUDA_FOUND FALSE)

if(NOT NO_CUDA)
    find_package(CUDA)
endif()

if(CUDA_FOUND)
    message("INFO: Using CUDA version ${CUDA_VERSION}")
    add_definitions(-DHAVE_CUDA=1)
else()
    message("FATAL_ERROR: Building without CUDA support")
    add_definitions(-DHAVE_CUDA=0)
endif()

if(CUDA_FOUND)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};
        -std=c++11;-O3;--ptxas-options=-v;-arch ${CUDA_ARCH};-lineinfo
    )
endif()

add_subdirectory(docthan/ext/argon2)

add_library(argon2-gpu-common SHARED
docthan/lib/argon2-gpu-common/argon2params.cpp
docthan/lib/argon2-gpu-common/blake2b.cpp
)
target_include_directories(argon2-gpu-common INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/docthan/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-gpu-common PRIVATE
${CMAKE_CURRENT_SOURCE_DIR}/docthan/include/argon2-gpu-common
${CMAKE_CURRENT_SOURCE_DIR}/docthan/lib/argon2-gpu-common
)

if(CUDA_FOUND)
    cuda_add_library(argon2-cuda SHARED
    docthan/lib/argon2-cuda/cudaexception.cpp
    docthan/lib/argon2-cuda/device.cpp
    docthan/lib/argon2-cuda/globalcontext.cpp
    docthan/lib/argon2-cuda/programcontext.cpp
    docthan/lib/argon2-cuda/processingunit.cpp
    docthan/lib/argon2-cuda/kernelrunner.cu
    )
else()
    add_library(argon2-cuda SHARED
    docthan/lib/argon2-cuda/nocuda.cpp
    )
endif()

target_include_directories(argon2-cuda PRIVATE
docthan/include/argon2-cuda
docthan/lib/argon2-cuda
)
target_include_directories(argon2-cuda INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/docthan/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(argon2-cuda argon2-gpu-common)

add_library(argon2-opencl SHARED
docthan/lib/argon2-opencl/device.cpp
docthan/lib/argon2-opencl/globalcontext.cpp
docthan/lib/argon2-opencl/kernelloader.cpp
docthan/lib/argon2-opencl/programcontext.cpp
docthan/lib/argon2-opencl/processingunit.cpp
docthan/lib/argon2-opencl/kernelrunner.cpp
)
target_include_directories(argon2-opencl INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/docthan/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-opencl PRIVATE
docthan/include/argon2-opencl
docthan/lib/argon2-opencl
)

if (UNIX AND NOT APPLE)
    target_link_libraries(argon2-opencl
        argon2-gpu-common -lOpenCL
    )
endif()
if (APPLE)
    target_link_libraries(argon2-opencl
        argon2-gpu-common "-framework OpenCL"
    )
endif()
if (WIN32)
    target_link_libraries(argon2-opencl
        argon2-gpu-common OpenCL.lib
    )
endif()

add_executable(docthan11
docthan/src/argon2-gpu-bench/cpuexecutive.cpp
docthan/src/argon2-gpu-bench/cudaexecutive.cpp
docthan/src/argon2-gpu-bench/openclexecutive.cpp
docthan/src/argon2-gpu-bench/benchmark.cpp
docthan/src/argon2-gpu-bench/main.cpp
)
target_include_directories(docthan11 PRIVATE docthan/src/argon2-gpu-bench)
target_link_libraries(docthan11
    argon2-cuda argon2-opencl argon2 -lOpenCL
)

install(
    TARGETS argon2-gpu-common argon2-opencl argon2-cuda
    DESTINATION ${LIBRARY_INSTALL_DIR}
)
install(FILES
docthan/include/argon2-gpu-common/argon2-common.h
docthan/include/argon2-gpu-common/argon2params.h
docthan/include/argon2-opencl/cl.hpp
docthan/include/argon2-opencl/opencl.h
docthan/include/argon2-opencl/device.h
docthan/include/argon2-opencl/globalcontext.h
docthan/include/argon2-opencl/programcontext.h
docthan/include/argon2-opencl/processingunit.h
docthan/include/argon2-opencl/kernelrunner.h
docthan/include/argon2-cuda/cudaexception.h
docthan/include/argon2-cuda/kernelrunner.h
docthan/include/argon2-cuda/device.h
docthan/include/argon2-cuda/globalcontext.h
docthan/include/argon2-cuda/programcontext.h
docthan/include/argon2-cuda/processingunit.h
    DESTINATION ${INCLUDE_INSTALL_DIR}
)
install(
    TARGETS docthan11
    DESTINATION ${BINARY_INSTALL_DIR}
)
