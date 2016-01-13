cmake_minimum_required(VERSION 2.8)
project (Kokkos)
 
enable_language(CXX)

set(TOP_LEVEL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/..)

add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} -VV -C ${CMAKE_CFG_INTDIR})

macro(add_test_executable TEST_NAME_)
    set(TEST_NAME "${TEST_NAME_}")
    add_executable (${TEST_NAME} EXCLUDE_FROM_ALL ${ARGN})
    # target_compile_options(${TEST_NAME} PUBLIC ${CXX_EXTRA_FLAGS})
    if(WIN32)
        add_test(NAME ${TEST_NAME} WORKING_DIRECTORY ${LIBRARY_OUTPUT_PATH} COMMAND ${TEST_NAME}${CMAKE_EXECUTABLE_SUFFIX})
    else()
        add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
    endif()
    add_dependencies(check ${TEST_NAME})
    set_tests_properties(${TEST_NAME} PROPERTIES FAIL_REGULAR_EXPRESSION "FAILED")
endmacro(add_test_executable)

include(CTest)

set(KOKKOS_GTEST_PATH ${TOP_LEVEL_DIR}/tpls/gtest)
include_directories(${KOKKOS_GTEST_PATH})

macro(add_gtest_executable)
  add_test_executable(${ARGN} ${KOKKOS_GTEST_PATH}/gtest/gtest-all.cc)
endmacro(add_gtest_executable)

set(KOKKOS_INSTALL_INCLUDE_DIR ${CMAKE_INSTALL_PREFIX}/kokkos)
set(KOKKOS_CORE_CONFIG ${CMAKE_CURRENT_BINARY_DIR}/KokkosCore_config.h)
install(FILES ${KOKKOS_CORE_CONFIG} DESTINATION ${KOKKOS_INSTALL_INCLUDE_DIR}/)

function(kokkos_add_library LIBRARY_NAME PARENT_DIR)
    set(HEADERS "")
    set(SOURCES "")

    # list(APPEND HEADERS ${KOKKOS_CORE_CONFIG})
    # install(FILES ${HEADERS} DESTINATION ${KOKKOS_INSTALL_INCLUDE_DIR}/)
    file(GLOB HEADERS_TOP ${PARENT_DIR}/*.hpp)
    list(APPEND HEADERS ${HEADERS_TOP} )
    install(FILES ${HEADERS_TOP} DESTINATION ${KOKKOS_INSTALL_INCLUDE_DIR}/)
    foreach(DIR ${ARGN})
        file(GLOB HEADERS_${DIR} ${PARENT_DIR}/${DIR}/*.hpp)
        file(GLOB SOURCES_${DIR} ${PARENT_DIR}/${DIR}/*.cpp)
        list(APPEND HEADERS ${HEADERS_${DIR}} )
        list(APPEND SOURCES ${SOURCES_${DIR}} )
        install(FILES ${HEADERS_${DIR}} DESTINATION ${KOKKOS_INSTALL_INCLUDE_DIR}/${DIR}/)
    endforeach(DIR)

    add_library(
        ${LIBRARY_NAME}
        ${HEADERS}
        ${SOURCES}
    )
    set_target_properties(${LIBRARY_NAME} PROPERTIES LINKER_LANGUAGE CXX)
endfunction()

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

## Core
#-----------------------------------------------------------------------------

set(KOKKOS_CORE_SRC_DIR ${TOP_LEVEL_DIR}/core/src)
include_directories(${KOKKOS_CORE_SRC_DIR})

kokkos_add_library(
    kokkoscore
    ${KOKKOS_CORE_SRC_DIR}
    impl
    Threads
    OpenMP
    Cuda
    Qthread
    Kalmar
)

target_link_libraries(kokkoscore -lm)

set(KOKKOS_CORE_TEST_DIR ${TOP_LEVEL_DIR}/core/unit_test)
include_directories(${KOKKOS_CORE_TEST_DIR})

macro(kokkos_add_core_test TEST_NAME)
    add_gtest_executable(${TEST_NAME} ${ARGN})
    target_link_libraries(${TEST_NAME} kokkoscore)
endmacro(kokkos_add_core_test)

kokkos_add_core_test(
    UnitTest_Serial
    ${KOKKOS_CORE_TEST_DIR}/UnitTestMain.cpp 
    ${KOKKOS_CORE_TEST_DIR}/TestSerial.cpp
)

kokkos_add_core_test(
    UnitTest_Kalmar
    ${KOKKOS_CORE_TEST_DIR}/UnitTestMain.cpp
    ${KOKKOS_CORE_TEST_DIR}/TestKalmar.cpp
)

kokkos_add_core_test(
    UnitTest_Default
    ${KOKKOS_CORE_TEST_DIR}/UnitTestMain.cpp
    ${KOKKOS_CORE_TEST_DIR}/TestDefaultDeviceType.cpp
    ${KOKKOS_CORE_TEST_DIR}/TestDefaultDeviceTypeInit.cpp
)

kokkos_add_core_test(
    UnitTest_HWLOC
    ${KOKKOS_CORE_TEST_DIR}/UnitTestMain.cpp
    ${KOKKOS_CORE_TEST_DIR}/TestHWLOC.cpp
)

kokkos_add_core_test(
    UnitTest_AllocationTracker
    ${KOKKOS_CORE_TEST_DIR}/UnitTestMain.cpp
    ${KOKKOS_CORE_TEST_DIR}/TestAllocationTracker.cpp
)

set(KOKKOS_CORE_PERF_DIR ${TOP_LEVEL_DIR}/core/perf_test)
include_directories(${KOKKOS_CORE_PERF_DIR})
kokkos_add_core_test(
    PerfTest
    ${KOKKOS_CORE_PERF_DIR}/PerfTestMain.cpp
    ${KOKKOS_CORE_PERF_DIR}/PerfTestHost.cpp
)

## Containers
#-----------------------------------------------------------------------------

set(KOKKOS_CONTAINERS_SRC_DIR ${TOP_LEVEL_DIR}/containers/src)
include_directories(${KOKKOS_CONTAINERS_SRC_DIR})

kokkos_add_library(
    kokkoscontainers
    ${KOKKOS_CONTAINERS_SRC_DIR}
    impl
)
target_link_libraries(kokkoscontainers kokkoscore)

macro(kokkos_add_containers_test TEST_NAME)
    add_gtest_executable(${TEST_NAME} ${ARGN})
    target_link_libraries(${TEST_NAME} kokkoscontainers)
endmacro(kokkos_add_containers_test)

set(KOKKOS_CONTAINERS_TEST_DIR ${TOP_LEVEL_DIR}/containers/unit_tests)
include_directories(${KOKKOS_CONTAINERS_TEST_DIR})

kokkos_add_containers_test(
    ContainersUnitTest
    ${KOKKOS_CONTAINERS_TEST_DIR}/UnitTestMain.cpp 
    ${KOKKOS_CONTAINERS_TEST_DIR}/TestCuda.cpp
    ${KOKKOS_CONTAINERS_TEST_DIR}/TestSerial.cpp
)

## Examples
#-----------------------------------------------------------------------------
set(KOKKOS_EXAMPLE_DIR ${TOP_LEVEL_DIR}/example)
macro(kokkos_add_example TEST_NAME)
    add_executable(${TEST_NAME} ${ARGN})
    target_link_libraries(${TEST_NAME} kokkoscore)
endmacro()

kokkos_add_example(
    KalmarSimple
    ${KOKKOS_EXAMPLE_DIR}/kalmar/simple/main.cpp
)

kokkos_add_example(
    KalmarAtomic
    ${KOKKOS_EXAMPLE_DIR}/kalmar/atomic/main.cpp
)

kokkos_add_example(
    KalmarReduction
    ${KOKKOS_EXAMPLE_DIR}/kalmar/reduction/main.cpp
)

kokkos_add_example(
    KalmarTeam
    ${KOKKOS_EXAMPLE_DIR}/kalmar/team/main.cpp
)

# kokkos_add_example(
#     kl
#     ${KOKKOS_EXAMPLE_DIR}/kalmar/kl/main.cpp
# )

# kokkos_add_example(
#    Kalmar3Solution
#    ${KOKKOS_EXAMPLE_DIR}/kalmar/3sol/main.cpp
# )

# kokkos_add_example(
#    Kalmar4Solution
#    ${KOKKOS_EXAMPLE_DIR}/kalmar/4sol/main.cpp
# )

