# vendor/NSTTaskScheduler/CMakeLists.txt
# Correct, minimal, working version for the real repo structure

cmake_minimum_required(VERSION 3.31)
project(NSTTaskScheduler LANGUAGES CXX)

# ------------------------------------------------------------------
# Configs (keep the 4 classic builds)
# ------------------------------------------------------------------
set(CMAKE_CONFIGURATION_TYPES "Debug;Release;Instrumented_Debug;Instrumented_Release" CACHE STRING "" FORCE)

# ------------------------------------------------------------------
# C++ settings
# ------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ------------------------------------------------------------------
# Output to Bin/ like original
# ------------------------------------------------------------------
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/Bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/Bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/Bin)

foreach(CONFIG ${CMAKE_CONFIGURATION_TYPES})
    string(TOUPPER ${CONFIG} C)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${C} ${CMAKE_CURRENT_SOURCE_DIR}/Bin)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${C} ${CMAKE_CURRENT_SOURCE_DIR}/Bin)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${C} ${CMAKE_CURRENT_SOURCE_DIR}/Bin)
endforeach()

# ------------------------------------------------------------------
# Platform detection
# ------------------------------------------------------------------
set(IS_WINDOWS ${WIN32})
set(IS_POSIX   ${UNIX} AND NOT APPLE)
set(IS_OSX     ${APPLE})

# ------------------------------------------------------------------
# Global flags (match original Premake exactly)
# ------------------------------------------------------------------
if(MSVC)
    add_compile_options(/W4 /MP /GF /Gy /fp:fast /GR- /wd4127 /arch:SSE2)
    add_link_options(/MANIFEST:NO)
    # Static runtime
    foreach(flag CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
                 CMAKE_CXX_FLAGS_INSTRUMENTED_DEBUG CMAKE_CXX_FLAGS_INSTRUMENTED_RELEASE)
        string(REPLACE "/MD" "/MT" ${flag} "${${flag}}")
    endforeach()
else()
    add_compile_options(-Wall -Wextra -Wno-invalid-offsetof -fno-omit-frame-pointer -msse2 -fPIC)
    if(NOT IS_OSX)
        add_compile_options(-fPIE)
        add_link_options(-rdynamic)
    endif()
endif()

# ------------------------------------------------------------------
# Configuration-specific defines
# ------------------------------------------------------------------
foreach(CONFIG Debug Release Instrumented_Debug Instrumented_Release)
    set(is_debug FALSE)
    set(is_instrumented FALSE)
    if(CONFIG MATCHES "Debug|Instrumented_Debug")
        set(is_debug TRUE)
    endif()
    if(CONFIG MATCHES "Instrumented")
        set(is_instrumented TRUE)
    endif()

    add_compile_definitions(
        $<$<CONFIG:${CONFIG}>:MT_UNICODE>
        $<$<BOOL:${is_debug}>:_DEBUG>
        $<$<BOOL:${is_debug}>:_CRTDBG_MAP_ALLOC>
        $<$<NOT:$<BOOL:${is_debug}>>:NDEBUG>
        $<$<BOOL:${is_instrumented}>:MT_INSTRUMENTED_BUILD>
        _CRT_SECURE_NO_WARNINGS
    )
endforeach()

# ------------------------------------------------------------------
# TaskScheduler library
# ------------------------------------------------------------------
file(GLOB_RECURSE SCHEDULER_SOURCES
    Scheduler/Source/*.cpp
    Scheduler/Include/*.h
)

# Pick correct platform implementation
if(IS_WINDOWS)
    file(GLOB PLATFORM_SOURCES Scheduler/Include/Platform/Windows/*.cpp)
else()
    file(GLOB PLATFORM_SOURCES Scheduler/Include/Platform/Posix/*.cpp)
endif()

add_library(TaskScheduler STATIC
    ${SCHEDULER_SOURCES}
    ${PLATFORM_SOURCES}
)

target_include_directories(TaskScheduler PUBLIC
    Scheduler/Include
    ThirdParty/Boost.Context
    ThirdParty/Squish
)

# ------------------------------------------------------------------
# TaskSchedulerTests executable (self-contained, no UnitTest++ needed!)
# ------------------------------------------------------------------
file(GLOB_RECURSE TEST_SOURCES
    SchedulerTests/*.cpp
    SchedulerTests/*.h
)

add_executable(TaskSchedulerTests ${TEST_SOURCES})

target_include_directories(TaskSchedulerTests PRIVATE
    Scheduler/Include
    ThirdParty/Squish
    ThirdParty/Boost.Context
)

target_link_libraries(TaskSchedulerTests PRIVATE TaskScheduler)

if(NOT IS_WINDOWS)
    target_link_libraries(TaskSchedulerTests PRIVATE pthread)
endif()

# ------------------------------------------------------------------
# Optional: run tests with ctest
# ------------------------------------------------------------------
include(CTest)
add_test(NAME NSTTaskScheduler_Tests COMMAND TaskSchedulerTests WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/Bin)

# ------------------------------------------------------------------
# Export alias
# ------------------------------------------------------------------
add_library(NST::TaskScheduler ALIAS TaskScheduler)

message(STATUS "NSTTaskScheduler configured successfully — ready for use in TinyCore!")