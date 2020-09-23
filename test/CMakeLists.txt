set( Catch2__TEST_DIR "${CMAKE_CURRENT_LIST_DIR}" )

add_library(
    libtest
    ${Catch2__TEST_DIR}/src/test.c
)

target_include_directories(
    libtest
    PUBLIC
    ${Catch2__TEST_DIR}/include
)

target_link_libraries(
    libtest
    PRIVATE
    gcov
)

set_target_properties(
    libtest
    PROPERTIES
    OUTPUT_NAME test
    COMPILE_FLAGS "--coverage"
    LINK_FLAGS= "--coverage"
)

catch2__add_test(
    NAME test__catch2
    SOURCES "${Catch2__TEST_DIR}/test__catch2.cpp"
    LINK_LIBRARIES libtest
)

catch2__build_test_executable()
