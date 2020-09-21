set( Catch2__DIR "${CMAKE_CURRENT_LIST_DIR}" CACHE STRING "The root directory for the Catch2 module." )

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

    endif( _catch2__TESTS )

endfunction( catch2__build_test_executable )


#*************************************************************************#
#*  Public Function:
#*      catch2__generate_coverage_report
#*  Description:
#*      Builds a single test executable containing all test groups previously
#*      added through function add_test_group above.
#*  Parameters:
#*      OUTPUT_DIR - Optional: the directory in which html coverage report
#*      will be generated.  
#*  Usage:  
#*      catch2__generate_coverage_report()
#*************************************************************************#

function( catch2__generate_coverage_report )
    find_program( LCOV_EXECUTABLE lcov REQUIRED )
    find_program( GENHTML_EXECUTABLE genhtml REQUIRED )

    set( option_args )
    set( one_value_args OUTPUT_DIR )
    set( multi_value_args )

    cmake_parse_arguments( catch2__generate_coverage_report "${option_args}" "${one_value_args}" "${multi_value_args}" ${ARGN} )

    if( NOT catch2__generate_coverage_report_OUTPUT_DIR )
        set( catch2__generate_coverage_report_OUTPUT_DIR ${CMAKE_BINARY_DIR}/coverage )
    endif( NOT catch2__generate_coverage_report_OUTPUT_DIR )

    file( MAKE_DIRECTORY ${catch2__generate_coverage_report_OUTPUT_DIR} )

    set_target_properties(
        test__all
        PROPERTIES
        COMPILE_FLAGS "--coverage"
        LINK_FLAGS "--coverage"
    )

    add_custom_command(
        TARGET test__all
        POST_BUILD
        COMMAND test__all WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMAND lcov  --quiet --directory ${CMAKE_BINARY_DIR} --capture --output-file ${catch2__generate_coverage_report_OUTPUT_DIR}/coverage.info
        # Workaround - explicitly remove files from coverage report that commonly show up, but are not part of the project being built.
        COMMAND lcov --quiet --remove ${catch2__generate_coverage_report_OUTPUT_DIR}/coverage.info -o ${catch2__generate_coverage_report_OUTPUT_DIR}/coverage.info '*9*' '*Catch2*'
        COMMAND genhtml --quiet ${catch2__generate_coverage_report_OUTPUT_DIR}/coverage.info -o ${catch2__generate_coverage_report_OUTPUT_DIR}
    )

endfunction( catch2__generate_coverage_report )
