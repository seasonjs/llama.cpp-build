cmake_minimum_required(VERSION 3.12)

project("llama.cpp-build")

include(FetchContent)
include(./cmake/fetch_prebuilt_ggml.cmake)

set(
    LLAMA_SOURCE_DIR
    "${CMAKE_CURRENT_SOURCE_DIR}/build/_deps/llama-src"
    CACHE PATH
    "Optional local llama source tree. When present it is preferred over downloading llama."
)

set(GGML_ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/vendor/ggml" CACHE PATH "Path to an unpacked ggml SDK")
set(GGML_RELEASE_BASE_URL "https://github.com/seasonjs/ggml.cpp-build/releases/download" CACHE STRING "Base release URL, for example https://github.com/seasonjs/ggml.cpp-build/releases/download")
set(GGML_RELEASE_TAG "v0.9.8" CACHE STRING "Release tag containing the prebuilt ggml SDK archives")

# add dependencies
get_filename_component(_llama_local_source_dir "${LLAMA_SOURCE_DIR}" ABSOLUTE BASE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Ensure non-empty build number for llama.cpp versioning

set(LLAMA_UPSTREAM_TAG "b8464" CACHE STRING "Default llama build number" FORCE)


# Strategy: ggml dynamic + llama static
# Note: We'll use BUILD_SHARED_LIBS=ON to build both as shared initially,
# but llama.cpp's CMake allows us to use ggml as dynamic

set(LLAMA_BUILD_COMMON OFF CACHE BOOL "Build common utils" FORCE)

# Important: Build shared libraries initially (ggml + llama)
set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libs" FORCE)
set(LLAMA_USE_SYSTEM_GGML ON CACHE BOOL "Use system ggml" FORCE)

message("check LLAMA_UPSTREAM_TAG ${LLAMA_UPSTREAM_TAG}")

add_compile_definitions(GGML_MAX_NAME=128)

if (EXISTS "${GGML_ROOT_DIR}")
    ggml_import_prebuilt(
        ROOT_DIR "${GGML_ROOT_DIR}"
        OUT_ROOT_DIR GGML_PACKAGE_ROOT
    )
elseif (GGML_RELEASE_BASE_URL)
    ggml_import_prebuilt(
        RELEASE_BASE_URL "${GGML_RELEASE_BASE_URL}"
        RELEASE_TAG "${GGML_RELEASE_TAG}"
        OUT_ROOT_DIR GGML_PACKAGE_ROOT
    )
else()
    message(FATAL_ERROR
        "Set GGML_ROOT_DIR to an unpacked ggml SDK or provide GGML_RELEASE_BASE_URL so CMake can download one.")
endif()

if (EXISTS "${_llama_local_source_dir}/CMakeLists.txt")
    message(STATUS "Using local llama source tree: ${_llama_local_source_dir}")
    add_subdirectory("${_llama_local_source_dir}" "${CMAKE_BINARY_DIR}/llama")
else()
    message(STATUS "Fetching llama.cpp ${LLAMA_UPSTREAM_TAG} from GitHub")
    FetchContent_Declare(
        llama
        GIT_REPOSITORY https://github.com/ggml-org/llama.cpp.git
        GIT_TAG        ${LLAMA_UPSTREAM_TAG}
        GIT_SHALLOW    TRUE
    )
    FetchContent_MakeAvailable(llama)
endif()

# We don't need to copy ggml's runtime binaries to the output directory
# ggml_copy_runtime_binaries(main_app
#     ROOT_DIR "${GGML_PACKAGE_ROOT}"
# )
