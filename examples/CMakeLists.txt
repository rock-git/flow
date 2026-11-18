add_executable(hello hello.cpp)
target_link_libraries(hello PRIVATE "$<LINK_LIBRARY:WHOLE_ARCHIVE,flow>" stacktrace)
