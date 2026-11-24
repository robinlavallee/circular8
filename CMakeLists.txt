cmake_minimum_required(VERSION 3.16)

# Circular8 can be built with any CMake generator, for example:
#   cmake -S . -B build -G "Unix Makefiles"               # Makefile
#   cmake -S . -B build -G "CodeBlocks - Unix Makefiles"  # Code::Blocks project (+ Makefile)
#   cmake -S . -B build -G "Visual Studio 17 2022" -A x64 # Visual Studio solution
# See README.md ("Building with CMake") for details.

project(circular8 LANGUAGES C)

# ---------------------------------------------------------------------------
# Options (mirror the defines used by circular8.cbp)
# ---------------------------------------------------------------------------
option(CIRCULAR8_ENABLE_FFMPEG     "Enable background video playback through FFmpeg" ON)
option(CIRCULAR8_ENABLE_CONTROLLER "Enable ADX/serial controller (touch + LED) support" ON)
option(CIRCULAR8_UNITY_BUILD       "Compile everything through src/everything.c (faster on MinGW)" ${MINGW})
option(CIRCULAR8_ENABLE_SANITIZERS "Enable UBSan in Debug builds (GCC/Clang, non-Windows)" ON)

# Directory holding prebuilt Windows libraries, laid out like the .cbp expects:
#   clibs/SDL3, clibs/SDL3_image, clibs/SDL3_mixer, clibs/SDL3_ttf, clibs/ffmpeg
set(CIRCULAR8_CLIBS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/clibs" CACHE PATH
    "Folder with prebuilt dependencies (SDL3*, ffmpeg) for platforms without a package manager")
foreach(_dep IN ITEMS SDL3 SDL3_image SDL3_mixer SDL3_ttf ffmpeg)
    if(IS_DIRECTORY "${CIRCULAR8_CLIBS_DIR}/${_dep}")
        list(APPEND CMAKE_PREFIX_PATH "${CIRCULAR8_CLIBS_DIR}/${_dep}")
    endif()
endforeach()

# Default to a Debug build for single-config generators (Makefiles, Ninja, Code::Blocks)
get_property(_is_multi_config GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT _is_multi_config AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS Debug Release RelWithDebInfo MinSizeRel)
endif()

# ---------------------------------------------------------------------------
# Dependencies
# ---------------------------------------------------------------------------
find_package(SDL3       REQUIRED CONFIG COMPONENTS SDL3)
find_package(SDL3_image REQUIRED CONFIG)
find_package(SDL3_ttf   REQUIRED CONFIG)
find_package(SDL3_mixer REQUIRED CONFIG)

if(CIRCULAR8_ENABLE_FFMPEG)
    set(_ffmpeg_libs avformat avcodec avutil)
    find_package(PkgConfig QUIET)
    if(PKG_CONFIG_FOUND)
        pkg_check_modules(FFMPEG QUIET IMPORTED_TARGET libavformat libavcodec libavutil)
    endif()
    if(FFMPEG_FOUND)
        set(CIRCULAR8_FFMPEG_TARGETS PkgConfig::FFMPEG)
    else()
        # Fallback for Windows / prebuilt FFmpeg (e.g. clibs/ffmpeg or -DFFMPEG_ROOT=...)
        find_path(FFMPEG_INCLUDE_DIR libavformat/avformat.h HINTS "${FFMPEG_ROOT}" PATH_SUFFIXES include)
        if(NOT FFMPEG_INCLUDE_DIR)
            message(FATAL_ERROR "FFmpeg (libavformat/libavcodec/libavutil) not found. "
                                "Install it, set FFMPEG_ROOT, or configure with -DCIRCULAR8_ENABLE_FFMPEG=OFF.")
        endif()
        foreach(_lib IN LISTS _ffmpeg_libs)
            find_library(FFMPEG_${_lib}_LIBRARY ${_lib} HINTS "${FFMPEG_ROOT}" PATH_SUFFIXES lib)
            if(NOT FFMPEG_${_lib}_LIBRARY)
                message(FATAL_ERROR "FFmpeg library '${_lib}' not found. Set FFMPEG_ROOT or disable CIRCULAR8_ENABLE_FFMPEG.")
            endif()
            add_library(FFmpeg::${_lib} UNKNOWN IMPORTED)
            set_target_properties(FFmpeg::${_lib} PROPERTIES
                IMPORTED_LOCATION "${FFMPEG_${_lib}_LIBRARY}"
                INTERFACE_INCLUDE_DIRECTORIES "${FFMPEG_INCLUDE_DIR}")
            list(APPEND CIRCULAR8_FFMPEG_TARGETS FFmpeg::${_lib})
        endforeach()
    endif()
endif()

# ---------------------------------------------------------------------------
# Sources
# ---------------------------------------------------------------------------
set(CIRCULAR8_SOURCES
    src/main.c
    src/events.c
    src/gameinit.c
    src/language.c
    src/load.c
    src/miscutils.c
    src/render.c
    src/texture.c
    src/game/chart.c
    src/game/difficultyselect.c
    src/game/ingame.c
    src/game/mainmenu.c
    src/game/options.c
    src/game/profileselect.c
    src/game/rendersong.c
    src/game/scorescreen.c
    src/game/songselect.c
    src/thirdparty/inih/ini.c
    src/thirdparty/SHA-256/sha256.c
)
if(CIRCULAR8_ENABLE_FFMPEG)
    list(APPEND CIRCULAR8_SOURCES src/video.c)
endif()
if(CIRCULAR8_ENABLE_CONTROLLER)
    list(APPEND CIRCULAR8_SOURCES
        src/controller/led.c
        src/controller/touch.c
        src/thirdparty/serial/serial.c)
endif()

file(GLOB_RECURSE CIRCULAR8_HEADERS CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h")

if(CIRCULAR8_UNITY_BUILD)
    # everything.c #includes all the .c files above, so they are only listed
    # (for IDEs) but not compiled on their own.
    set_source_files_properties(${CIRCULAR8_SOURCES} PROPERTIES HEADER_FILE_ONLY ON)
    add_executable(circular8 src/everything.c ${CIRCULAR8_SOURCES} ${CIRCULAR8_HEADERS})
else()
    add_executable(circular8 ${CIRCULAR8_SOURCES} ${CIRCULAR8_HEADERS})
endif()

source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}/src" PREFIX "src"
             FILES ${CIRCULAR8_SOURCES} ${CIRCULAR8_HEADERS})

set_target_properties(circular8 PROPERTIES
    C_STANDARD 99
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS OFF
    # Assets (gfx/, sfx/, fonts/, profiles/, levels/, options.ini) are loaded
    # relative to the working directory, so run/debug from the repository root.
    VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    XCODE_SCHEME_WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    # Mirror the bin/<Config>/ layout of the Code::Blocks project.
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
)
if(MSVC)
    # MSVC has no C99 switch; C11 is a superset and enables its conforming preprocessor.
    set_target_properties(circular8 PROPERTIES C_STANDARD 11)
endif()
set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" PROPERTY VS_STARTUP_PROJECT circular8)

target_include_directories(circular8 PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/src")

target_compile_definitions(circular8 PRIVATE
    _TIME_BITS=64
    _FILE_OFFSET_BITS=64
    _DEFAULT_SOURCE
    INI_MAX_LINE=1024
    $<$<CONFIG:Release,MinSizeRel>:NDEBUG>
    $<$<BOOL:${CIRCULAR8_ENABLE_FFMPEG}>:Circular8_ENABLE_FFmpeg>
    $<$<BOOL:${CIRCULAR8_ENABLE_CONTROLLER}>:Circular8_ENABLE_CONTROLLER>
)

if(WIN32)
    target_compile_definitions(circular8 PRIVATE
        WIN32_LEAN_AND_MEAN NOSERVICE NOIME NOATOM NOGDI NOGDICAPMASKS NOMETAFILE
        NOMINMAX NOMSG NOOPENFILE NOSCROLL NOSOUND NOSYSMETRICS NOTEXTMETRIC NOWH
        NOCOMM NOKANJI NOCRYPT NOMCX)
endif()

target_link_libraries(circular8 PRIVATE
    SDL3_ttf::SDL3_ttf
    SDL3_image::SDL3_image
    SDL3_mixer::SDL3_mixer
    SDL3::SDL3
    ${CIRCULAR8_FFMPEG_TARGETS}
)
if(NOT WIN32)
    target_link_libraries(circular8 PRIVATE m)
endif()

# ---------------------------------------------------------------------------
# Compiler flags (based on circular8.cbp)
# ---------------------------------------------------------------------------
if(MSVC)
    target_compile_options(circular8 PRIVATE /W3 /utf-8)
    target_compile_definitions(circular8 PRIVATE _CRT_SECURE_NO_WARNINGS)
else()
    target_compile_options(circular8 PRIVATE
        -Wall -Wextra -Wshadow -Wunreachable-code -Wswitch-enum
        -Wformat=2 -Wno-format-nonliteral
        -Werror=implicit-function-declaration)
    if(NOT WIN32)
        target_compile_options(circular8 PRIVATE -pedantic)
    endif()
    if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
        target_compile_options(circular8 PRIVATE
            -Wcast-align -Wswitch-default -Winit-self -Wshift-overflow=2 -Wduplicated-cond)
    elseif(CMAKE_C_COMPILER_ID MATCHES "Clang")
        target_compile_options(circular8 PRIVATE
            -Wshift-overflow -Wno-incompatible-pointer-types-discards-qualifiers)
    endif()

    # Debug: stack protector + UBSan (as in the "Debug" / "Clang-Dev" targets)
    if(CIRCULAR8_ENABLE_SANITIZERS AND NOT WIN32)
        set(_san -fsanitize=undefined)
        if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
            list(APPEND _san -fsanitize=bounds-strict)
        endif()
        target_compile_options(circular8 PRIVATE
            $<$<CONFIG:Debug>:-fstack-protector-all>
            "$<$<CONFIG:Debug>:${_san}>")
        target_link_options(circular8 PRIVATE
            $<$<CONFIG:Debug>:-fstack-protector-all>
            "$<$<CONFIG:Debug>:${_san}>")
    endif()

    # Release hardening (as in the "Release" target)
    if(NOT WIN32)
        # -U first: some toolchains (e.g. Ubuntu's GCC) predefine _FORTIFY_SOURCE
        target_compile_options(circular8 PRIVATE
            "$<$<NOT:$<CONFIG:Debug>>:-fstack-protector-strong;-U_FORTIFY_SOURCE;-D_FORTIFY_SOURCE=2>")
        include(CheckPIESupported)
        check_pie_supported()
        set_target_properties(circular8 PROPERTIES POSITION_INDEPENDENT_CODE ON)
        if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
            target_link_options(circular8 PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:LINKER:-z,relro,-z,now>)
        endif()
    endif()
endif()

# ---------------------------------------------------------------------------
# Windows: copy SDL3 DLLs next to the executable so it runs from the IDE
# ---------------------------------------------------------------------------
if(WIN32 AND NOT CMAKE_VERSION VERSION_LESS 3.21)
    add_custom_command(TARGET circular8 POST_BUILD
        COMMAND "$<IF:$<BOOL:$<TARGET_RUNTIME_DLLS:circular8>>,${CMAKE_COMMAND};-E;copy_if_different;$<TARGET_RUNTIME_DLLS:circular8>;$<TARGET_FILE_DIR:circular8>,${CMAKE_COMMAND};-E;true>"
        COMMAND_EXPAND_LISTS
        COMMENT "Copying runtime DLLs next to circular8")
endif()
