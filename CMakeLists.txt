﻿# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.

cmake_minimum_required (VERSION 3.26)
include(FetchContent)
include(cmake/external/onnxruntime_external_deps.cmake)

project (Generators LANGUAGES C CXX)
set(USE_CUDA 1) # "Build with CUDA support"
set(USE_ORT_EXT 0) # "Build with Onnxruntime Extensions tokenizer support"

# Checking if CUDA is supported
include(CheckLanguage)
check_language(CUDA)
if (CMAKE_CUDA_COMPILER)
    message(STATUS "CUDA found CUDAToolkit_VERSION ${CUDAToolkit_VERSION} with CMAKE_CUDA_COMPILER ${CMAKE_CUDA_COMPILER}" )
    set(CMAKE_CUDA_RUNTIME_LIBRARY Shared)
else()
    message(STATUS "CUDA not found")
endif()

if("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" AND CMAKE_C_COMPILER_VERSION VERSION_LESS 9)
  message(FATAL_ERROR  "GCC version must be greater than or equal to 9")
endif()

if("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" AND CMAKE_C_COMPILER_VERSION VERSION_LESS 10)
  add_compile_definitions(USE_CXX17=1)
  message("USE C++17")
  set(CMAKE_CXX_STANDARD 17)
else()
  set(CMAKE_CXX_STANDARD 20)
endif()

set(GENERATORS_ROOT ${PROJECT_SOURCE_DIR}/src)
set(MODELS_ROOT ${PROJECT_SOURCE_DIR}/src/models)
set(TESTS_ROOT ${PROJECT_SOURCE_DIR}/src/tests)
set(PYTHON_ROOT ${PROJECT_SOURCE_DIR}/src/python)

# CUDA Being enabled will make it not a debug build without this option, so all of the C++ headers will complain
# about a mismatch with the actual debug headers and it'll fail to link. I don't know why this happens, or if this is the best fix.
if(USE_CUDA AND CMAKE_CUDA_COMPILER AND CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_compile_definitions(_DEBUG=1)
endif()

message(STATUS "Adding source files")

file(GLOB generator_srcs CONFIGURE_DEPENDS
   "${GENERATORS_ROOT}/*.h"
   "${GENERATORS_ROOT}/*.cpp"
   "${MODELS_ROOT}/*.h"
   "${MODELS_ROOT}/*.cpp"
)

file(GLOB test_srcs CONFIGURE_DEPENDS
   "${TESTS_ROOT}/*.h"
   "${TESTS_ROOT}/*.cpp"
)

file(GLOB python_srcs CMAKE_CONFIGURE_DEPENDS
   "${PYTHON_ROOT}/*.h"
   "${PYTHON_ROOT}/*.cpp"
)

if(USE_CUDA AND CMAKE_CUDA_COMPILER)
    # Don't let cmake set a default value for CMAKE_CUDA_ARCHITECTURES
    cmake_policy(SET CMP0104 OLD)
    enable_language(CUDA)
    message( STATUS "CMAKE_CUDA_COMPILER_VERSION: ${CMAKE_CUDA_COMPILER_VERSION}")
    # set(CUDA_PROPAGATE_HOST_FLAGS ON)
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -Xcudafe --diag_suppress=2803 --expt-relaxed-constexpr")
    file(GLOB generator_cuda_srcs CONFIGURE_DEPENDS
        "${GENERATORS_ROOT}/*.cu"
        "${GENERATORS_ROOT}/*.cuh"
        "${MODELS_ROOT}/*.cu"
        "${MODELS_ROOT}/*.cuh"
    )
    list(APPEND generator_srcs ${generator_cuda_srcs})
    add_compile_definitions(USE_CUDA=1)
    include_directories("${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}")
else()
    file(GLOB generator_cuda_srcs "${GENERATORS_ROOT}/*_cuda*.*")
    list(REMOVE_ITEM generator_srcs ${generator_cuda_srcs})
endif()

## TODO: this is a hack to get the shared library to export .lib files on windows. We should find a better way when we need to compile dynamic libraries
#if(WIN32)
#    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
#endif()

add_library (onnxruntime-genai ${generator_srcs})
target_include_directories(onnxruntime-genai PRIVATE ${CMAKE_SOURCE_DIR}/ort/include)

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set_target_properties(onnxruntime-genai PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

if(WIN32)
  set(ONNXRUNTIME_LIB "onnxruntime.lib")
  set(ONNXRUNTIME_FILES "onnxruntime*.dll")
  set(ONNXRUNTIME_EXTENSIONS_LIB "tfmtok_c.lib")
  set(ONNXRUNTIME_EXTENSIONS_FILES "tfmtok_c.dll")
elseif(APPLE)
  set(ONNXRUNTIME_LIB "libonnxruntime.dylib")
  set(ONNXRUNTIME_FILES "libonnxruntime*.dylib")
else()
  set(ONNXRUNTIME_LIB "libonnxruntime.so")
  set(ONNXRUNTIME_FILES "libonnxruntime*.so*")
  set(ONNXRUNTIME_EXTENSIONS_LIB "tfmtok_c.so")
endif()

add_executable (Tests ${test_srcs})
target_include_directories(Tests PRIVATE ${CMAKE_SOURCE_DIR}/ort/include)
target_link_directories(Tests PRIVATE ${CMAKE_SOURCE_DIR}/ort/lib)
target_link_libraries(Tests PRIVATE onnxruntime-genai ${ONNXRUNTIME_LIB})
#
pybind11_add_module(python ${python_srcs})
target_include_directories(python PRIVATE ${CMAKE_SOURCE_DIR}/ort/include)
target_link_directories(python PRIVATE ${CMAKE_SOURCE_DIR}/ort/lib)
target_link_libraries(python PRIVATE onnxruntime-genai ${ONNXRUNTIME_LIB})
set_target_properties(python PROPERTIES OUTPUT_NAME "onnxruntime_genai")
if(USE_CUDA AND CMAKE_CUDA_COMPILER)
    set_target_properties(onnxruntime-genai PROPERTIES LINKER_LANGUAGE CUDA)
    set_target_properties(python PROPERTIES LINKER_LANGUAGE CUDA)
    set_target_properties(Tests PROPERTIES LINKER_LANGUAGE CUDA)
    target_link_libraries(onnxruntime-genai PRIVATE cublasLt cublas cudnn curand cufft cudart)
    target_link_libraries(Tests PRIVATE cublasLt cublas cudnn curand cufft cudart)
    target_link_libraries(python PRIVATE cublasLt cublas cudnn curand cufft cudart)
endif()


# Visual C++ - /MP Multiple process build, /WX Warnings as errors
# Enable:
#   C5038: data member 'member1' will be initialized after data member 'member2'
# Disable:
#   C4100: 'this_': unreferenced formal parameter
if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /w15038 /wd4100 /W4 /WX")
endif()

# Copy the onnxruntime binaries into the build folder so it's found on launch
file(GLOB onnxruntime_libs "${CMAKE_SOURCE_DIR}/ort/${ONNXRUNTIME_FILES}")
foreach(DLL_FILE ${onnxruntime_libs})
    add_custom_command(
        TARGET onnxruntime-genai POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy
                ${DLL_FILE}
                ${CMAKE_BINARY_DIR}/$<CONFIG>/${DLL_FILE_NAME}
    )
endforeach()

if(USE_ORT_EXT)
    add_compile_definitions(USE_ORT_EXT=1)
    target_include_directories(onnxruntime-genai PRIVATE ${CMAKE_SOURCE_DIR}/ort-ext/include)
    target_include_directories(Tests PRIVATE ${CMAKE_SOURCE_DIR}/ort-ext/include)
    target_include_directories(python PRIVATE ${CMAKE_SOURCE_DIR}/ort-ext/include)
    target_link_directories(Tests PRIVATE ${CMAKE_SOURCE_DIR}/ort-ext/lib)
    target_link_directories(python PRIVATE ${CMAKE_SOURCE_DIR}/ort-ext/lib)
    target_link_libraries(Tests PRIVATE ${ONNXRUNTIME_EXTENSIONS_LIB})
    target_link_libraries(python PRIVATE ${ONNXRUNTIME_EXTENSIONS_LIB})

    # Copy the onnxruntime-extensions binaries into the build folder so it's found on launch
    file(GLOB onnxruntime_extensions_libs "${CMAKE_SOURCE_DIR}/ort-ext/lib/${ONNXRUNTIME_EXTENSIONS_FILES}")
    foreach(DLL_FILE ${onnxruntime_extensions_libs})
        add_custom_command(
            TARGET onnxruntime-genai POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy
                    ${DLL_FILE}
                    ${CMAKE_BINARY_DIR}/$<CONFIG>/${DLL_FILE_NAME}
        )
    endforeach()
 endif()

# Have visual studio put all files into one single folder vs the default split of header files into a separate folder
source_group(TREE ${GENERATORS_ROOT} FILES ${generator_srcs})
source_group("Sources" FILES ${test_srcs})
source_group("Sources" FILES ${python_srcs})

set_property (DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT Tests)
