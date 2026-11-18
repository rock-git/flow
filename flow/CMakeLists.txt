file(GLOB FLOW_SRCS CONFIGURE_DEPENDS "*.cpp" "*.c")
# Files with their own `main`, plus the test-only sources built below.
list(REMOVE_ITEM FLOW_SRCS
     ${CMAKE_CURRENT_SOURCE_DIR}/LinkTest.cpp
     ${CMAKE_CURRENT_SOURCE_DIR}/TLSTest.cpp
     ${CMAKE_CURRENT_SOURCE_DIR}/MkCertCli.cpp
     ${CMAKE_CURRENT_SOURCE_DIR}/FlowTest.cpp
     ${CMAKE_CURRENT_SOURCE_DIR}/TLSHandshakePoolTest.cpp)
# Swift hooks are only meaningful in a Swift build.
list(REMOVE_ITEM FLOW_SRCS
     ${CMAKE_CURRENT_SOURCE_DIR}/swift_concurrency_hooks.cpp
     ${CMAKE_CURRENT_SOURCE_DIR}/swift_task_priority.cpp)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
  list(APPEND FLOW_SRCS aarch64/memcmp.S aarch64/memcpy.S)
endif()

set(GEN_DIR ${CMAKE_CURRENT_BINARY_DIR}/include/flow)
file(MAKE_DIRECTORY ${GEN_DIR})

include(ApiVersions.cmake)
configure_file(ApiVersion.h.cmake ${GEN_DIR}/ApiVersion.h)
configure_file(config.h.cmake ${GEN_DIR}/config.h)

execute_process(
  COMMAND git -C ${CMAKE_SOURCE_DIR} rev-parse HEAD
  OUTPUT_VARIABLE CURRENT_GIT_VERSION
  OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
if(NOT CURRENT_GIT_VERSION)
  set(CURRENT_GIT_VERSION "unknown")
endif()
configure_file(SourceVersion.h.cmake ${GEN_DIR}/SourceVersion.h)

# ProtocolVersion.h is generated from ProtocolVersions.cmake with Jinja2.
find_package(Python3 REQUIRED COMPONENTS Interpreter)
set(PROTOCOL_VERSION_PYTHON_EXECUTABLE "${Python3_EXECUTABLE}")
find_package(Jinja2)
if(NOT Jinja2_FOUND)
  set(VENV_DIR "${CMAKE_BINARY_DIR}/protocol_version-venv")
  execute_process(COMMAND "${Python3_EXECUTABLE}" -m venv ${VENV_DIR}
                  COMMAND_ERROR_IS_FATAL ANY)
  find_program(
    VENV_Python3_EXECUTABLE
    NAMES python3 python
    PATHS ${VENV_DIR}/bin ${VENV_DIR}/Scripts
    REQUIRED NO_DEFAULT_PATH NO_CACHE)
  execute_process(
    COMMAND "${VENV_Python3_EXECUTABLE}" -m pip install -r
            "${CMAKE_CURRENT_SOURCE_DIR}/protocolversion/requirements.txt"
    COMMAND_ERROR_IS_FATAL ANY)
  set(PROTOCOL_VERSION_PYTHON_EXECUTABLE "${VENV_Python3_EXECUTABLE}")
endif()

set(PV_HEADER ${GEN_DIR}/ProtocolVersion.h)
add_custom_command(
  OUTPUT ${PV_HEADER}
  COMMAND ${PROTOCOL_VERSION_PYTHON_EXECUTABLE}
          ${CMAKE_CURRENT_SOURCE_DIR}/protocolversion/protocol_version.py
          --source ${CMAKE_CURRENT_SOURCE_DIR}/ProtocolVersions.cmake
          --generator cpp --output ${PV_HEADER}
  DEPENDS protocolversion/protocol_version.py
          protocolversion/ProtocolVersion.h.template ProtocolVersions.cmake)
add_custom_target(ProtocolVersion DEPENDS ${PV_HEADER})

find_package(Coroutines COMPONENTS Experimental Final REQUIRED)

add_library(flow STATIC ${FLOW_SRCS})
add_dependencies(flow ProtocolVersion)
target_include_directories(
  flow
  PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include" "${CMAKE_CURRENT_BINARY_DIR}/include"
  PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}")
target_link_libraries(
  flow
  PUBLIC fmt::fmt SimpleOpt crc32 libb64 OpenSSL::SSL boost_target
         Threads::Threads ${CMAKE_DL_LIBS} std::coroutines
  PRIVATE stacktrace)

if(UNIX AND NOT APPLE)
  target_link_libraries(flow PRIVATE folly_memcpy rt)
  target_compile_definitions(flow PRIVATE WITH_FOLLY_MEMCPY)
endif()
if(APPLE)
  find_library(IO_KIT IOKit)
  find_library(CORE_FOUNDATION CoreFoundation)
  target_link_libraries(flow PRIVATE ${IO_KIT} ${CORE_FOUNDATION})
endif()
if(WIN32)
  target_link_libraries(flow PUBLIC winmm.lib psapi.lib)
endif()

# Links flow into an executable so unresolved references to non-flow code
# (e.g. fdbrpc) fail the build instead of being ignored by the archive.
add_executable(flowlinktest LinkTest.cpp)
target_link_libraries(flowlinktest PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,flow>" stacktrace)

add_executable(mkcert MkCertCli.cpp)
target_link_libraries(mkcert PRIVATE flow)

if(FLOW_BUILD_TESTS)
  add_executable(flow_test FlowTest.cpp)
  target_link_libraries(flow_test PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,flow>" stacktrace)
  add_test(NAME flow_unit_tests COMMAND flow_test)
endif()
