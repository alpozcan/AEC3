cmake_minimum_required(VERSION 3.0)

set(PROJECT webrtc_aec3)

set(CMAKE_CXX_STANDARD 17)

project(${PROJECT} VERSION 0.1.0)

# Determine Android ABI early so we can filter platform-specific sources
if (CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(ANDROID_ABI ${CMAKE_ANDROID_ARCH_ABI})
else()
    set(ANDROID_ABI "")
endif()

add_compile_options(-Wall -fPIC -Wno-deprecated -fexceptions)

# Avoid forcing -m64 when targeting Android/ARM; only add -m64 on non-Android hosts
if (NOT CMAKE_SYSTEM_NAME STREQUAL "Android")
    add_compile_options(-m64)
endif()

if(APPLE)
    add_definitions(-DWEBRTC_MAC -DWEBRTC_POSIX)
elseif(UNIX AND NOT APPLE)
    add_definitions(-DWEBRTC_LINUX -DWEBRTC_POSIX)
elseif(WIN32)
    add_definitions(-DWEBRTC_WIN)
endif()

file(GLOB AECM_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/api/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/aec3/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/aec3/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/include/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/include/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/logging/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/logging/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/resampler/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/resampler/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/system_wrappers/source/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base/system_wrappers/source/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/base/system_wrappers/include/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/memory/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/memory/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/strings/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/strings/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/system/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/system/*.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/jsoncpp/src/lib_json/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/base/jsoncpp/src/lib_json/*.h
)
list(FILTER AECM_SRC EXCLUDE REGEX ".*unittest.cc$")
# Exclude x86 SSE-specific sources when building for non-x86 targets (e.g., arm64-v8a)
if (NOT (ANDROID_ABI MATCHES "^(x86|x86_64)$" OR CMAKE_SYSTEM_PROCESSOR MATCHES "^(i386|x86_64|x86)$"))
    list(FILTER AECM_SRC EXCLUDE REGEX ".*sse.*")
endif()
set(SRC_FILES ${AECM_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/checks.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/time_utils.h
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base/sanitizer.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/cascaded_biquad_filter.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft_mips.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft_neon.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/cascaded_biquad_filter.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft.h
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft_tables_common.h
)

add_library(${PROJECT} STATIC
    ${SRC_FILES}
)

# Architecture-specific additions: only compile SSE/x86 sources on x86 targets.
# On Android, CMAKE_ANDROID_ARCH_ABI is set (e.g. arm64-v8a, armeabi-v7a, x86, x86_64).
if (CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(ANDROID_ABI ${CMAKE_ANDROID_ARCH_ABI})
else()
    set(ANDROID_ABI "")
endif()

if (ANDROID_ABI MATCHES "^(x86|x86_64)$" OR CMAKE_SYSTEM_PROCESSOR MATCHES "^(i386|x86_64|x86)$")
    list(APPEND SRC_FILES ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility/ooura_fft_sse2.cc)
    # headers like ooura_fft_tables_neon_sse2.h are included by sources when needed
endif()

# If we appended any files, update the target sources.
if (TARGET ${PROJECT})
    target_sources(${PROJECT} PRIVATE ${SRC_FILES})
endif()

target_include_directories(${PROJECT} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/
    ${CMAKE_CURRENT_SOURCE_DIR}/api
    ${CMAKE_CURRENT_SOURCE_DIR}/base/rtc_base
    ${CMAKE_CURRENT_SOURCE_DIR}/base
    ${CMAKE_CURRENT_SOURCE_DIR}/base/system_wrappers/include
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing
    ${CMAKE_CURRENT_SOURCE_DIR}/base/abseil
    ${CMAKE_CURRENT_SOURCE_DIR}/base/jsoncpp/include
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/aec3/
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/resampler
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/logging
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/include
    ${CMAKE_CURRENT_SOURCE_DIR}/audio_processing/utility
)

set_target_properties(${PROJECT} PROPERTIES LINKER_LANGUAGE CXX)
