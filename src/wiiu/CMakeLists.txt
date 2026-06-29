cmake_minimum_required(VERSION 3.27)
project(wiiuldr LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(REPO_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../..")
set(ELF_COMMON_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../elf_common")
set(IDA_SDK_ROOT "${REPO_ROOT}/third_party/ida-sdk/src")
set(IDA_INCLUDE_DIR "${IDA_SDK_ROOT}/include")
set(IDA_LOADER_DIR "${IDA_SDK_ROOT}/ldr")

if(NOT EXISTS "${IDA_INCLUDE_DIR}/ida.hpp")
    message(FATAL_ERROR
        "The ida-sdk submodule was not found at ${IDA_SDK_ROOT}. "
        "Initialize submodules before configuring this project.")
endif()

if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(FATAL_ERROR "The IDA 9.3 loader must be built as a 64-bit library.")
endif()

set(IDA_PLATFORM_DEFINITION)

if(WIN32)
    if(NOT MSVC)
        message(FATAL_ERROR "Windows builds require Visual Studio/MSVC.")
    endif()

    set(IDA_PLATFORM_DEFINITION __NT__)
    set(IDA_LIBRARY "${IDA_SDK_ROOT}/lib/x64_win_64/ida.lib")
elseif(APPLE)
    if(CMAKE_OSX_ARCHITECTURES)
        list(LENGTH CMAKE_OSX_ARCHITECTURES OSX_ARCHITECTURE_COUNT)
        if(NOT OSX_ARCHITECTURE_COUNT EQUAL 1)
            message(FATAL_ERROR
                "Universal macOS builds are not supported. Configure arm64 and "
                "x86_64 in separate build directories.")
        endif()
        list(GET CMAKE_OSX_ARCHITECTURES 0 TARGET_ARCHITECTURE)
    else()
        execute_process(
            COMMAND "${CMAKE_CXX_COMPILER}" -dumpmachine
            OUTPUT_VARIABLE COMPILER_TARGET
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        if(COMPILER_TARGET MATCHES "^(arm64|aarch64)-")
            set(TARGET_ARCHITECTURE arm64)
        elseif(COMPILER_TARGET MATCHES "^(x86_64|amd64)-")
            set(TARGET_ARCHITECTURE x86_64)
        else()
            message(FATAL_ERROR
                "Unable to determine the macOS target architecture from "
                "${COMPILER_TARGET}. Set CMAKE_OSX_ARCHITECTURES explicitly.")
        endif()
    endif()

    if(TARGET_ARCHITECTURE MATCHES "^(arm64|aarch64)$")
        set(IDA_ARCHITECTURE arm64)
    elseif(TARGET_ARCHITECTURE MATCHES "^(x86_64|amd64|AMD64)$")
        set(IDA_ARCHITECTURE x64)
    else()
        message(FATAL_ERROR "Unsupported macOS architecture: ${TARGET_ARCHITECTURE}")
    endif()

    set(IDA_PLATFORM_DEFINITION __MAC__)
    set(IDA_LIBRARY
        "${IDA_SDK_ROOT}/lib/${IDA_ARCHITECTURE}_mac_64/libida.dylib")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm64|aarch64)$")
        set(IDA_ARCHITECTURE arm64)
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
        set(IDA_ARCHITECTURE x64)
    else()
        message(FATAL_ERROR "Unsupported Linux architecture: ${CMAKE_SYSTEM_PROCESSOR}")
    endif()

    set(IDA_PLATFORM_DEFINITION __LINUX__)
    set(IDA_LIBRARY
        "${IDA_SDK_ROOT}/lib/${IDA_ARCHITECTURE}_linux_64/libida.so")
else()
    message(FATAL_ERROR "Unsupported build platform: ${CMAKE_SYSTEM_NAME}")
endif()

if(NOT EXISTS "${IDA_LIBRARY}")
    message(FATAL_ERROR
        "The IDA library for this platform was not found at ${IDA_LIBRARY}.")
endif()

set(SOURCES
    "${ELF_COMMON_PATH}/elf_reader.h"
    "${ELF_COMMON_PATH}/elf.h"
    cafe.h
    cafe_loader.h
    cafe_loader.cpp
    wiiu.cpp
)

add_library(wiiuldr SHARED ${SOURCES})

target_include_directories(wiiuldr PRIVATE
    "${IDA_INCLUDE_DIR}"
    "${IDA_LOADER_DIR}"
    "${ELF_COMMON_PATH}"
)

target_compile_definitions(wiiuldr PRIVATE
    __IDP__
    __EA64__
    ${IDA_PLATFORM_DEFINITION}
)

target_link_libraries(wiiuldr PRIVATE "${IDA_LIBRARY}")

set_target_properties(wiiuldr PROPERTIES
    OUTPUT_NAME "wiiu"
    PREFIX ""
)

if(MSVC)
    set_target_properties(wiiuldr PROPERTIES
        MSVC_RUNTIME_LIBRARY "MultiThreadedDLL"
    )
endif()
