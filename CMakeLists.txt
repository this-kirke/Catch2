project( Catch2 )

cmake_minimum_required( VERSION 3.12 )

set( CMAKE_VERBOSE_MAKEFILE off )

set( Catch2__DIR "${CMAKE_CURRENT_LIST_DIR}" CACHE STRING "The root directory for the Catch2 module." )

option( CATCH2__TEST "Test Catch2 functionality." OFF )

if( NOT TARGET Catch2 )
    add_subdirectory( ${Catch2__DIR}/Catch2 )
endif( NOT TARGET Catch2 )

#*************************************************************************#
#*  Public Function:
#*      catch2__add_test
#*  Description:
#*      Adds a test group to be built according to the supplied parameters
#*  Parameters:
#*      NAME - the name of the test target
#*      SOURCES - The sources used to build the test target
#*      LINK_LIBRARIES - Optional list of library targets to be linked with the test target
#*      DEPENDENCIES - Optional list of dependencies to be built before the test target
#*  Usage:
#*      catch2__add_test(
#*          NAME <name>
#*          SOURCES <source_1 source_2 ...>
#*          LINK_LIBRARIES <library_1 library_2 ...>
#*          DEPENDENCIES <dependency_1 dependency_2  ...>
#*      )
#*************************************************************************#
function( catch2__add_test )

    set( option_args )
    set( one_value_args NAME )
    set( multi_value_args SOURCES LINK_LIBRARIES DEPENDENCIES )

    cmake_parse_arguments( catch2__add_test "${option_args}" "${one_value_args}" "${multi_value_args}" ${ARGN} )

    #   Uncomment below to inspect parameters passed to function
    #[[
    message( "Parameters passed to function catch2__add_test:")
    message( "NAME:\t${catch2__add_test_NAME}" )
    message( "SOURCES:\t${catch2__add_test_SOURCES}")
    message( "LINK_LIBRARIES:\t${catch2__add_test_LINK_LIBRARIES}")
    message( "DEPENDENCIES:\t${catch2__add_test_DEPENDENCIES}")
    ]]

    #   Don't build the executable yet - we'll just build an object library that
    #   can be linked by the test runner executable later in the build process
    add_library(
        ${catch2__add_test_NAME}
        OBJECT
        ${catch2__add_test_SOURCES}
    )

    target_link_libraries(
        ${catch2__add_test_NAME}
        Catch2
        "${catch2__add_test_LINK_LIBRARIES}"
    )

    if( NOT ${catch2__add_test_DEPENDENCIES} STREQUAL "" )
        add_dependencies(
            ${catch2__add_test_NAME}
            ${catch2__add_test_DEPENDENCIES}
        )
    endif( NOT ${catch2__add_test_DEPENDENCIES} STREQUAL "" )

    #   Keep a gobal list of added tests which will be linked to the test
    #   runner executable later in the build process
    if( NOT "${catch2__add_test_NAME}" IN_LIST _catch2__TESTS )
        list( APPEND _catch2__TESTS ${catch2__add_test_NAME} )
        set( _catch2__TESTS ${_catch2__TESTS} CACHE INTERNAL "" )
    endif()

endfunction( catch2__add_test )

#*************************************************************************#
#*  Public Function:
#*      catch2__build_test_executable
#*  Description:
#*      Builds a single test executable containing all test groups previously
#*      added through function add_test_group above.
#*  Usage:
#*      catch2__build_test_executable()
#*************************************************************************#

function( catch2__build_test_executable )

    if( _catch2__TESTS )

        add_executable(
            test__all
            ${Catch2__DIR}/src/test_runner.cpp
        )

        target_link_libraries(
            test__all
            PRIVATE
            Catch2
            "${_catch2__TESTS}"
        )

        set_target_properties(
            test__all
            PROPERTIES
            COMPILE_FLAGS "--coverage"
            LINK_FLAGS "--coverage"
        )

    endif( _catch2__TESTS )

endfunction( catch2__build_test_executable )

if( CATCH2__TEST )
    add_subdirectory( ${Catch2__DIR}/test )
endif( CATCH2__TEST )
