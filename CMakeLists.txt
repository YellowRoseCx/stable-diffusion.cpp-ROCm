cmake_minimum_required(VERSION 3.12)
project("stable-diffusion")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT XCODE AND NOT MSVC AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(SD_STANDALONE ON)
else()
    set(SD_STANDALONE OFF)
endif()

add_subdirectory(ggml)

set(SD_LIB stable-diffusion)

add_library(${SD_LIB} stable-diffusion.h stable-diffusion.cpp)
target_link_libraries(${SD_LIB} PUBLIC ggml)

if (GGML_HIPBLAS)
    list(APPEND CMAKE_PREFIX_PATH /opt/rocm)

    find_package(hip)
    find_package(hipblas)
    find_package(rocblas)

    if (${hipblas_FOUND} AND ${hip_FOUND})
        message(STATUS "HIP and hipBLAS found")
        target_compile_definitions(${SD_LIB} PRIVATE GGML_USE_HIPBLAS GGML_USE_CUBLAS)
        add_library(ggml-rocm OBJECT ./ggml/src/ggml-cuda.cu ./ggml/src/ggml-cuda.h)
        target_compile_definitions(ggml-rocm PRIVATE GGML_CUDA_DMMV_X=${LLAMA_CUDA_DMMV_X})
        target_compile_definitions(ggml-rocm PRIVATE GGML_CUDA_MMV_Y=${LLAMA_CUDA_MMV_Y})
        target_compile_definitions(ggml-rocm PRIVATE K_QUANTS_PER_ITERATION=${LLAMA_CUDA_KQUANTS_ITER})
        target_compile_definitions(ggml-rocm PRIVATE CC_TURING=1000000000)
        set_source_files_properties(ggml-cuda.cu PROPERTIES LANGUAGE CXX)
        target_link_libraries(ggml-rocm PRIVATE hip::device PUBLIC hip::host roc::rocblas roc::hipblas)
        target_link_libraries(${SD_LIB} PRIVATE ggml-rocm)  # Link ggml-rocm to your library
    else()
        message(WARNING "hipBLAS or HIP not found. Try setting CMAKE_PREFIX_PATH=/opt/rocm")
    endif()
endif()

target_include_directories(${SD_LIB} PUBLIC .)
target_compile_features(${SD_LIB} PUBLIC cxx_std_11)
