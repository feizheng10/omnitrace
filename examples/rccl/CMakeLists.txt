cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

project(omnitrace-rccl-example LANGUAGES CXX)

if(OMNITRACE_DISABLE_EXAMPLES)
    get_filename_component(_DIR ${CMAKE_CURRENT_LIST_DIR} NAME)

    if(${PROJECT_NAME} IN_LIST OMNITRACE_DISABLE_EXAMPLES OR ${_DIR} IN_LIST
                                                             OMNITRACE_DISABLE_EXAMPLES)
        return()
    endif()
endif()

function(rccl_message _MSG_TYPE)
    if("${CMAKE_PROJECT_NAME}" STREQUAL "omnitrace"
       AND "$ENV{OMNITRACE_CI}"
       AND "${_MSG_TYPE}" MATCHES "WARNING")
        set(_MSG_TYPE STATUS) # don't generate warnings during CI
    endif()
    if("${CMAKE_PROJECT_NAME}" STREQUAL "omnitrace")
        omnitrace_message(${_MSG_TYPE} ${ARGN})
    else()
        message(${_MSG_TYPE} ${ARGN})
    endif()
endfunction()

find_package(hip HINTS ${ROCmVersion_DIR} PATHS ${ROCmVersion_DIR})

if(NOT hip_FOUND)
    rccl_message(AUTHOR_WARNING "${PROJECT_NAME} skipped. Missing HIP...")
    return()
endif()

if("${CMAKE_PROJECT_NAME}" STREQUAL "omnitrace" AND "$ENV{OMNITRACE_CI}")
    find_package(rccl QUIET) # avoid generating warning in CI
else()
    find_package(rccl)
endif()

if(NOT rccl_FOUND)
    rccl_message(AUTHOR_WARNING "${PROJECT_NAME} skipped. Missing RCCL...")
    return()
endif()

if(hip_FOUND AND rccl_FOUND)
    include(FetchContent)
    fetchcontent_declare(
        rccl-tests GIT_REPOSITORY https://github.com/ROCmSoftwarePlatform/rccl-tests.git)

    # After the following call, the CMake targets defined by googletest and Catch2 will be
    # available to the rest of the build
    fetchcontent_populate(rccl-tests)

    get_filename_component(rccl_ROOT_DIR "${rccl_INCLUDE_DIR}" DIRECTORY)

    rccl_message(STATUS "Building rccl-tests...")
    execute_process(
        COMMAND make HIP_HOME=${ROCM_PATH} RCCL_HOME=${rccl_ROOT_DIR}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/_deps/rccl-tests-src
        RESULT_VARIABLE _RCCL_BUILD_RET
        ERROR_VARIABLE _RCCL_BUILD_ERR
        OUTPUT_VARIABLE _RCCL_BUILD_OUT
        OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_STRIP_TRAILING_WHITESPACE)

    if(NOT _RCCL_BUILD_RET EQUAL 0)
        rccl_message(${_RCCL_BUILD_OUT})
        rccl_message(AUTHOR_WARNING "Failed to build rccl-tests: ${_RCCL_BUILD_ERR}")
    else()
        file(GLOB RCCL_TEST_EXECUTABLES
             ${CMAKE_BINARY_DIR}/_deps/rccl-tests-src/build/*_perf)
        set(_RCCL_TEST_TARGETS)

        foreach(_EXE ${RCCL_TEST_EXECUTABLES})
            get_filename_component(_EXE_NAME "${_EXE}" NAME)
            execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${_EXE}
                                    ${CMAKE_CURRENT_BINARY_DIR}/${_EXE_NAME})
            add_executable(rccl-tests::${_EXE_NAME} IMPORTED GLOBAL)
            set_property(
                TARGET rccl-tests::${_EXE_NAME}
                PROPERTY IMPORTED_LOCATION ${CMAKE_CURRENT_BINARY_DIR}/${_EXE_NAME})
            list(APPEND _RCCL_TEST_TARGETS "rccl-tests::${_EXE_NAME}")
        endforeach()

        set(RCCL_TEST_TARGETS
            "${_RCCL_TEST_TARGETS}"
            CACHE INTERNAL "rccl-test targets")
    endif()
endif()
