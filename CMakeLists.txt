cmake_minimum_required(VERSION 3.18)
project(speed-sam LANGUAGES CXX CUDA)

# Set C++ standard (inherited from parent project)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Ensure using paths defined by parent project
if(NOT TENSORRT_ROOT)
    set(TENSORRT_ROOT "C:/dev/TensorRT-8.6.1.6" CACHE PATH "Path to TensorRT installation")
endif()

if(NOT OpenCV_DIR)
    set(OpenCV_DIR "C:/dev/opencv/vc_build/install" CACHE PATH "Path to OpenCV installation")
endif()

message(STATUS "SPEED-SAM using TensorRT at: ${TENSORRT_ROOT}")
message(STATUS "SPEED-SAM using OpenCV at: ${OpenCV_DIR}")

# Find CUDA
find_package(CUDA REQUIRED)

# Find OpenCV
find_package(OpenCV REQUIRED)
if(NOT OpenCV_FOUND)
    message(FATAL_ERROR "OpenCV not found in speed-sam submodule. Please verify OpenCV_DIR: ${OpenCV_DIR}")
endif()

# Set TensorRT paths
find_path(TENSORRT_INCLUDE_DIR NvInfer.h 
    PATHS ${TENSORRT_ROOT}/include 
    NO_DEFAULT_PATH)
    
find_library(NVINFER_LIBRARY nvinfer 
    PATHS ${TENSORRT_ROOT}/lib 
    NO_DEFAULT_PATH)
    
find_library(NVONNXPARSER_LIBRARY nvonnxparser 
    PATHS ${TENSORRT_ROOT}/lib 
    NO_DEFAULT_PATH)

if(NOT TENSORRT_INCLUDE_DIR OR NOT NVINFER_LIBRARY OR NOT NVONNXPARSER_LIBRARY)
    message(FATAL_ERROR "Required TensorRT libraries not found. Verify TENSORRT_ROOT: ${TENSORRT_ROOT}")
endif()

# Create static library
add_library(SpeedSAM STATIC)

# Set source files
file(GLOB SOURCES 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp
)

# Set header files
file(GLOB HEADERS 
    ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h
)

# Add source and header files to library
target_sources(SpeedSAM PRIVATE ${SOURCES} ${HEADERS})

# Set include directories
target_include_directories(SpeedSAM 
    PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE
        ${CUDA_INCLUDE_DIRS}
        ${TENSORRT_INCLUDE_DIR}
        ${OpenCV_INCLUDE_DIRS}
)

# Set library dependencies
target_link_libraries(SpeedSAM 
    PUBLIC
        ${OpenCV_LIBS}
    PRIVATE
        ${NVINFER_LIBRARY}
        ${NVONNXPARSER_LIBRARY}
        ${CUDA_LIBRARIES}
        ${CUDA_CUBLAS_LIBRARIES}
)

# Split DLL copying into multiple commands to prevent failures
add_custom_command(TARGET SpeedSAM POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${TENSORRT_ROOT}/lib/nvinfer.dll"
        "$<TARGET_FILE_DIR:SpeedSAM>"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${TENSORRT_ROOT}/lib/nvonnxparser.dll"
        "$<TARGET_FILE_DIR:SpeedSAM>"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "C:/dev/cudnn-windows-x86_64-8.9.7.29_cuda12-archive/bin/cudnn64_8.dll"
        "$<TARGET_FILE_DIR:SpeedSAM>"
    COMMENT "Copying essential DLLs to output directory"
)

# Copy optional DLLs if they exist
foreach(dll_file 
        "nvinfer_builder_resource.dll"
        "nvinfer_plugin.dll")
    if(EXISTS "${TENSORRT_ROOT}/lib/${dll_file}")
        add_custom_command(TARGET SpeedSAM POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${TENSORRT_ROOT}/lib/${dll_file}"
                "$<TARGET_FILE_DIR:SpeedSAM>"
            COMMENT "Copying ${dll_file}"
        )
    else()
        message(STATUS "${dll_file} not found in ${TENSORRT_ROOT}/lib - skipping")
    endif()
endforeach()

# Copy assets and model directories to output directory for convenience
add_custom_command(TARGET SpeedSAM POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_CURRENT_SOURCE_DIR}/assets"
        "$<TARGET_FILE_DIR:SpeedSAM>/assets"
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${CMAKE_CURRENT_SOURCE_DIR}/model"
        "$<TARGET_FILE_DIR:SpeedSAM>/model"
    COMMENT "Copying assets and model directories to output directory"
)

# Create a more complete batch file with all required paths
file(WRITE ${CMAKE_BINARY_DIR}/run.bat
    "@echo off\n"
    "echo Setting up environment variables for TensorRT...\n"
    ":: Add TensorRT and CUDA paths to system path\n"
    "set PATH=${TENSORRT_ROOT}\\lib;${TENSORRT_ROOT}\\bin;C:\\dev\\cudnn-windows-x86_64-8.9.7.29_cuda12-archive\\bin;C:\\Program Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v12.6\\bin;%PATH%\n"
    "\n"
    ":: Set TensorRT environment variables (needed for builder)\n"
    "set TENSORRT_PATH=${TENSORRT_ROOT}\n"
    "set TRT_BUILDER_RESOURCE_PATH=${TENSORRT_ROOT}\\lib\\nvinfer_builder_resource.dll\n"
    "\n"
    "echo Running SpeedSAM...\n"
    "\"${CMAKE_BINARY_DIR}/Debug/SpeedSAM.exe\"\n"
    "if ERRORLEVEL 1 (\n"
    "    echo Error: Application exited with code %ERRORLEVEL%\n"
    "    pause\n"
    ")\n"
)

# Set CUDA compilation options
set_target_properties(SpeedSAM PROPERTIES 
    CUDA_SEPARABLE_COMPILATION ON
)

# Set compilation definitions
target_compile_definitions(SpeedSAM PRIVATE 
    USE_TENSORRT 
    USE_CUDA
)

# Export include directories
set(SPEED_SAM_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include CACHE INTERNAL "SPEED-SAM include directories")

# Output configuration information
message(STATUS "SPEED-SAM configuration:")
message(STATUS "  CUDA libraries: ${CUDA_LIBRARIES}")
message(STATUS "  TensorRT include: ${TENSORRT_INCLUDE_DIR}")
message(STATUS "  TensorRT libraries: ${NVINFER_LIBRARY}, ${NVONNXPARSER_LIBRARY}")
message(STATUS "  OpenCV libraries: ${OpenCV_LIBS}")
message(STATUS "  Include directory exported as: ${SPEED_SAM_INCLUDE_DIRS}")
