
cmake_minimum_required(VERSION 3.15)
include(cmake/version.cmake)

option(MSDFGEN_USE_OPENMP "Build with OpenMP support for multithreaded code" OFF)
option(MSDFGEN_USE_SKIA "Build with the Skia library" OFF)
option(MSDFGEN_DYNAMIC_RUNTIME "Link dynamic runtime library instead of static" On)
option(BUILD_SHARED_LIBS "Generate dynamic library files instead of static" OFF)


get_property(MULTI_CONFIG GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT MULTI_CONFIG AND NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE not set, defaulting to Release")
    set(CMAKE_BUILD_TYPE Release)
endif()

if(MSDFGEN_DYNAMIC_RUNTIME)
    set(MSDFGEN_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
else()
    set(MSDFGEN_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if(BUILD_SHARED_LIBS)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()


# Version is specified in vcpkg.json
project(msdfgen VERSION ${MSDFGEN_VERSION} LANGUAGES CXX)

if(MAX_WARNING_LEVEL)
    if (MSVC)
        add_compile_options(/W4)
    else()
        add_compile_options(-Wall -Wextra -Wpedantic)
    endif()
endif()

file(GLOB_RECURSE MSDFGEN_CORE_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "core/*.h" "core/*.hpp")
file(GLOB_RECURSE MSDFGEN_CORE_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "core/*.cpp")
file(GLOB_RECURSE MSDFGEN_EXT_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "ext/*.h" "ext/*.hpp")
file(GLOB_RECURSE MSDFGEN_EXT_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "ext/*.cpp" "lib/*.cpp")

# Core library
add_library(msdfgen-core "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen.h" ${MSDFGEN_CORE_HEADERS} ${MSDFGEN_CORE_SOURCES})
add_library(msdfgen::msdfgen-core ALIAS msdfgen-core)
set_target_properties(msdfgen-core PROPERTIES PUBLIC_HEADER "${MSDFGEN_CORE_HEADERS}")
set_property(TARGET msdfgen-core PROPERTY MSVC_RUNTIME_LIBRARY "${MSDFGEN_MSVC_RUNTIME}")
target_compile_definitions(msdfgen-core PUBLIC
    MSDFGEN_VERSION=${MSDFGEN_VERSION}
    MSDFGEN_VERSION_MAJOR=${MSDFGEN_VERSION_MAJOR}
    MSDFGEN_VERSION_MINOR=${MSDFGEN_VERSION_MINOR}
    MSDFGEN_VERSION_REVISION=${MSDFGEN_VERSION_REVISION}
    MSDFGEN_COPYRIGHT_YEAR=${MSDFGEN_COPYRIGHT_YEAR}
)
target_include_directories(msdfgen-core INTERFACE
    $<INSTALL_INTERFACE:include/msdfgen>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
)
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT msdfgen-core)



if(MSDFGEN_USE_OPENMP)
    # Note: Clang doesn't support OpenMP by default...
    find_package(OpenMP REQUIRED C  OMPONENTS CXX)
    target_compile_definitions(msdfgen-core PUBLIC MSDFGEN_USE_OPENMP)
    target_link_libraries(msdfgen-core PUBLIC OpenMP::OpenMP_CXX)
endif()

if(BUILD_SHARED_LIBS AND WIN32)
    target_compile_definitions(msdfgen-core PRIVATE "MSDFGEN_PUBLIC=__declspec(dllexport)")
    target_compile_definitions(msdfgen-core INTERFACE "MSDFGEN_PUBLIC=__declspec(dllimport)")
else()
    target_compile_definitions(msdfgen-core PUBLIC MSDFGEN_PUBLIC=)
endif()
add_subdirectory(3rdparty)
# Extensions library
add_library(msdfgen-ext "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen-ext.h" ${MSDFGEN_EXT_HEADERS} ${MSDFGEN_EXT_SOURCES})
add_library(msdfgen::msdfgen-ext ALIAS msdfgen-ext)
set_target_properties(msdfgen-ext PROPERTIES PUBLIC_HEADER "${MSDFGEN_EXT_HEADERS}")
set_property(TARGET msdfgen-ext PROPERTY MSVC_RUNTIME_LIBRARY "${MSDFGEN_MSVC_RUNTIME}")
target_compile_definitions(msdfgen-ext INTERFACE MSDFGEN_EXTENSIONS)
#if(NOT MSDFGEN_DISABLE_SVG)
    #target_link_libraries(msdfgen-ext PRIVATE tinyxml2::tinyxml2)
#else()
target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_DISABLE_SVG)
#endif()
#if(NOT MSDFGEN_DISABLE_PNG)
    #target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_USE_LIBPNG)
    #target_link_libraries(msdfgen-ext PRIVATE PNG::PNG)
#else()
    target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_DISABLE_PNG)
#endif()
target_link_libraries(msdfgen-ext PRIVATE freetype msdfgen::msdfgen-core)
target_include_directories(msdfgen-ext
    PUBLIC
        $<INSTALL_INTERFACE:include/msdfgen>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include
)
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT msdfgen-ext)
if(MSDFGEN_USE_SKIA)
    set(MSDFGEN_SKIA_LIB skia)
    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    set(THREADS_PREFER_PTHREAD_FLAG TRUE)
    find_package(Threads REQUIRED)
    if(NOT TARGET skia)
        if(MSDFGEN_USE_VCPKG)
            find_package(unofficial-skia REQUIRED)
            set(MSDFGEN_SKIA_LIB unofficial::skia::skia)
        else()
            find_package(skia REQUIRED)
        endif()
    endif()
    target_compile_features(msdfgen-ext PUBLIC cxx_std_17)
    target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_USE_SKIA)
    target_link_libraries(msdfgen-ext PRIVATE Threads::Threads ${MSDFGEN_SKIA_LIB})
endif()
if(BUILD_SHARED_LIBS AND WIN32)
    target_compile_definitions(msdfgen-ext PRIVATE "MSDFGEN_EXT_PUBLIC=__declspec(dllexport)")
    target_compile_definitions(msdfgen-ext INTERFACE "MSDFGEN_EXT_PUBLIC=__declspec(dllimport)")
else()
    target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_EXT_PUBLIC=)
endif()
add_library(msdfgen-full INTERFACE)
add_library(msdfgen::msdfgen ALIAS msdfgen-full)
target_link_libraries(msdfgen-full INTERFACE msdfgen::msdfgen-core msdfgen::msdfgen-ext)


# Hide ZERO_CHECK and ALL_BUILD targets
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER meta)

