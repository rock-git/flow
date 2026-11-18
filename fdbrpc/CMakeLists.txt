file(GLOB RPC_SRCS CONFIGURE_DEPENDS "*.cpp")
list(REMOVE_ITEM RPC_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/fdbrpc_test.cpp)

add_library(eio STATIC libeio/eio.c)
target_compile_definitions(eio PRIVATE USE_UCONTEXT)
target_compile_options(eio PRIVATE -w)

add_library(fdbrpc STATIC ${RPC_SRCS})
target_include_directories(
  fdbrpc
  PUBLIC include
  PRIVATE . libeio ${CMAKE_SOURCE_DIR}/contrib/rapidjson)
target_link_libraries(fdbrpc PUBLIC flow libb64 md5 PRIVATE eio)

if(FLOW_BUILD_TESTS)
  add_executable(fdbrpc_test fdbrpc_test.cpp)
  target_link_libraries(fdbrpc_test PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,fdbrpc>"
                        "$<LINK_LIBRARY:WHOLE_ARCHIVE,flow>" stacktrace)
  add_test(NAME fdbrpc_unit_tests COMMAND fdbrpc_test)
endif()
