cmake_minimum_required(VERSION 3.5)

set(CORROSION_VERBOSE_OUTPUT ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED 1)

execute_process(
    COMMAND rustup target list --installed
    OUTPUT_VARIABLE RUST_TARGETS
)
# Propagate arch to rust build for CI
set(Rust_CARGO_TARGET "")
if("${OS_NAME}" STREQUAL "linux")
    if ("${OS_ARCH}" STREQUAL "arm64")
        set(Rust_CARGO_TARGET "aarch64-unknown-linux-gnu")
    elseif("${CMAKE_CXX_COMPILER}" MATCHES "aarch64")
        set(Rust_CARGO_TARGET ${RUST_ENV_VARS} CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc)
        set(Rust_CARGO_TARGET "aarch64-unknown-linux-gnu")
    else()
        string(FIND "${RUST_TARGETS}" "musl" MUSL_TARGET_FOUND)
        if(NOT MUSL_TARGET_FOUND EQUAL -1)
            set(Rust_CARGO_TARGET "x86_64-unknown-linux-musl")
        else()
            set(Rust_CARGO_TARGET "x86_64-unknown-linux-gnu")
        endif()
    endif()
elseif("${OS_NAME}" STREQUAL "osx")
    if ("${OSX_BUILD_ARCH}" STREQUAL "arm64")
        set(Rust_CARGO_TARGET "aarch64-apple-darwin")
    elseif ("${OSX_BUILD_ARCH}" STREQUAL "x86_64")
        set(Rust_CARGO_TARGET "x86_64-apple-darwin")
    elseif ("${OS_ARCH}" STREQUAL "arm64")
        set(Rust_CARGO_TARGET "aarch64-apple-darwin")
    endif()
elseif(WIN32)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDLL") # Use MD_DynamicRelease
    if (MINGW AND "${OS_ARCH}" STREQUAL "arm64")
        set(Rust_CARGO_TARGET "aarch64-pc-windows-gnu")
    elseif (MINGW AND "${OS_ARCH}" STREQUAL "amd64")
        set(Rust_CARGO_TARGET "x86_64-pc-windows-gnu")
    elseif (MSVC AND "${OS_ARCH}" STREQUAL "arm64")
        set(Rust_CARGO_TARGET "aarch64-pc-windows-msvc")
    elseif (MSVC AND "${OS_ARCH}" STREQUAL "amd64")
        set(Rust_CARGO_TARGET "x86_64-pc-windows-msvc")
    endif()
endif()

execute_process(
    COMMAND rustup target list --installed
    OUTPUT_VARIABLE RUST_TARGETS
)
string(FIND "${RUST_TARGETS}" "wasm32-unknown-emscripten" WASM_TARGET_FOUND)

if (NOT WASM_TARGET_FOUND EQUAL -1)
  set(Rust_CARGO_TARGET "wasm32-unknown-emscripten")
endif()


include(FetchContent)

FetchContent_Declare(
    Corrosion
    GIT_REPOSITORY https://github.com/corrosion-rs/corrosion.git
    GIT_TAG a1a1aaa057a5da656c06c3d8505b767a4e941709
)
# Set any global configuration variables such as `Rust_TOOLCHAIN` before this line!
FetchContent_MakeAvailable(Corrosion)

message(STATUS "RUST_TARGETS: ${RUST_TARGETS}")
message(STATUS "WASM_TARGET_FOUND: ${WASM_TARGET_FOUND}")
message(STATUS "TARGET: ${TARGET}")
message(STATUS "DUCKDB_BUILD_TYPE: ${DUCKDB_BUILD_TYPE}")
message(STATUS "TARGET NAME: ${TARGET_NAME}")
message(STATUS "DUCKDB_PLATFORM: ${DUCKDB_PLATFORM}")
message(STATUS "OS_ARCH: ${OS_ARCH}")
message(STATUS "OS_NAME: ${OS_NAME}")
message(STATUS "Rust_CARGO_TARGET: ${Rust_CARGO_TARGET}")
# We currently only support the predefined targets.
if ("${Rust_CARGO_TARGET}" STREQUAL "")
   message(FATAL_ERROR "Failed to detect the correct platform")
endif()


# Import targets defined in a package or workspace manifest `Cargo.toml` file
corrosion_import_crate(MANIFEST_PATH "${CMAKE_SOURCE_DIR}/../text_embedding_rust/Cargo.toml"
CRATES "text_embedding_rust"
)

# Set extension name here
set(TARGET_NAME embed_text)

set(EXTENSION_NAME ${TARGET_NAME}_extension)
set(LOADABLE_EXTENSION_NAME ${TARGET_NAME}_loadable_extension)

project(${TARGET_NAME})
include_directories(src/include)

set(EXTENSION_SOURCES src/embed_text_extension.cpp)

build_static_extension(${TARGET_NAME} ${EXTENSION_SOURCES})
build_loadable_extension(${TARGET_NAME} " " ${EXTENSION_SOURCES})

if(APPLE)
    # link some mac specific libraries, and Metal for fast embedding inference
    set_property(TARGET text_embedding_rust APPEND PROPERTY
        INTERFACE_LINK_LIBRARIES
        "-framework CoreFoundation"
        "-framework Security"
        "-framework SystemConfiguration"
        "-framework Metal"
        "-framework CoreFoundation"
        "-framework MetalPerformanceShaders"
    )
elseif (WIN32)
    target_link_libraries(text_embedding_rust INTERFACE bcrypt)
endif()

target_link_libraries(${EXTENSION_NAME} text_embedding_rust)
target_link_libraries(${LOADABLE_EXTENSION_NAME} text_embedding_rust)


install(
  TARGETS ${EXTENSION_NAME} text_embedding_rust
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
