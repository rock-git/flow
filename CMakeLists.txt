cmake_minimum_required(VERSION 3.24)
cmake_policy(SET CMP0167 NEW)
project(flow LANGUAGES C CXX ASM)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
enable_testing()

option(FLOW_BUILD_TESTS "Build flow_test (unit test runner)" ON)

add_compile_definitions(BOOST_SYSTEM_NO_DEPRECATED HAVE_OPENSSL)
add_compile_options(-fno-omit-frame-pointer -Wall -Wno-sign-compare
                    -Wno-unused-parameter -Wno-unused-function
                    -Wno-delete-non-virtual-dtor)

find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(Boost REQUIRED COMPONENTS filesystem)
find_package(fmt CONFIG QUIET)
if(NOT fmt_FOUND)
  include(FetchContent)
  FetchContent_Declare(
    fmt
    URL "https://github.com/fmtlib/fmt/archive/11.1.4.tar.gz"
    URL_HASH SHA256=ac366b7b4c2e9f0dde63a59b3feb5ee59b67974b14ee5dc9ea8ad78aa2c1ee1e)
  FetchContent_MakeAvailable(fmt)
endif()

# Name kept so flow's sources/CMake match upstream FoundationDB.
add_library(boost_target INTERFACE)
target_link_libraries(boost_target INTERFACE Boost::boost Boost::filesystem)

add_subdirectory(contrib)
add_subdirectory(flow)
add_subdirectory(fdbrpc)
add_subdirectory(examples)
