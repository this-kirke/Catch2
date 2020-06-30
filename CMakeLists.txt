set( Catch2__DIR "${CMAKE_CURRENT_LIST_DIR}" CACHE STRING "The root directory for the Catch2 module." )

add_subdirectory( ${Catch2__DIR}/Catch2 )

#*************************************************************************#
#*  Public Function:
#*      catch2__add_test
#*  Description:
#*      Adds a test group to be built according to the supplied parameters
#*  Usage:  
#*      catch2__add_test(
#*          NAME <name>
#*          SOURCES <source_1 source_2 ...>
#*          DEPENDENCIES <dependency_1 dependency_2 ...>
#*      )
#*************************************************************************#
function( catch2__add_test )

    set( option_args )
    set( one_value_args NAME )
    set( multi_value_args SOURCES DEPENDENCIES )

    cmake_parse_arguments( catch2__add_test "${option_args}" "${one_value_args}" "${multi_value_args}" ${ARGN} )

    #   Uncomment below to inspect parameters passed to function
    #[[
    message( "Parameters passed to function catch2__add_test:")
    message( "NAME:\t${catch2__add_test_NAME}" )
    message( "SOURCES:\t${catch2__add_test_SOURCES}")
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
        "${catch2__add_test_DEPENDENCIES}"
    )

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

    set( option_args GENERATE_COVERAGE_REPORT )
    set( one_value_args COVERAGE_OUTPUT_DIR )
    set( multi_value_args )

    cmake_parse_arguments( catch2__build_test_executable "${option_args}" "${one_value_args}" "${multi_value_args}" ${ARGN} )

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


        add_custom_command(
            TARGET test__all
            POST_BUILD
            COMMAND test__all
        )

        if( catch2__build_test_executable_GENERATE_COVERAGE_REPORT )

            find_program( LCOV_EXECUTABLE lcov REQUIRED )
            find_program( GENHTML_EXECUTABLE genhtml REQUIRED )

            if( NOT catch2__build_test_executable_COVERAGE_OUTPUT_DIR )
                set( catch2__build_test_executable_COVERAGE_OUTPUT_DIR ${CMAKE_BINARY_DIR}/coverage )
            endif( NOT catch2__build_test_executable_COVERAGE_OUTPUT_DIR )

            file( MAKE_DIRECTORY ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR} )

            set_target_properties(
                test__all
                PROPERTIES
                COMPILE_FLAGS "--coverage"
                LINK_FLAGS "--coverage"
            )

            add_custom_command(
                TARGET test__all
                POST_BUILD
                COMMAND lcov  --quiet --directory ${CMAKE_BINARY_DIR} --capture --output-file ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR}/coverage.info
                # Workaround - explicitly remove files from coverage report that commonly show up, but are not part of the project being built.
                COMMAND lcov --quiet --remove ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR}/coverage.info -o ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR}/coverage.info '*9*' '*Catch2*'
                COMMAND genhtml --quiet ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR}/coverage.info -o ${catch2__build_test_executable_COVERAGE_OUTPUT_DIR}
            )
        
        endif( catch2__build_test_executable_GENERATE_COVERAGE_REPORT )


    endif( _catch2__TESTS )

endfunction( catch2__build_test_executable )