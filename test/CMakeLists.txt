add_subdirectory(unittests)

# Add any new sources here
set(SOURCES_HELPME
    cartesiantransform.h
    fftw_wrapper.h
    gamma.h
    gridsize.h
    helpme.h
    lapack_wrapper.h
    matrix.h
    memory.h
    mpi_wrapper.h
    powers.h
    splines.h
    string_utils.h
)

foreach(SOURCE_FILE ${SOURCES_HELPME})
    list (APPEND HELPME_INPUT_SOURCES "${PROJECT_SOURCE_DIR}/src/${SOURCE_FILE}")
endforeach()

include_directories(${PROJECT_SOURCE_DIR}/src)
include_directories(${PROJECT_SOURCE_DIR}/single_include)
include_directories(${FFTW_INCLUDES})
link_libraries(${BLAS_LIBRARIES} ${LAPACK_LIBRARIES} ${FFTW_LIBRARIES})

if(HAVE_MPI)
    link_libraries(${MPI_CXX_LIBRARIES})
endif()

# Ensure headers are inlined every time a build is triggered
add_custom_command(
    OUTPUT StandaloneHeader
    COMMAND ${PYTHON_EXECUTABLE} inline_headers.py
    DEPENDS ${HELPME_INPUT_SOURCES}
    WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}/tools"
    COMMENT "Generating single header..."
)

add_custom_target(SingleHeader
                  DEPENDS StandaloneHeader
)

# Find only the static library for linking with C++/C/Fortran wrappers
set(helpmestatic_name ${CMAKE_BINARY_DIR}/src/helpme.a)

# C example
add_executable (RunCWrapper fullexample.c)
add_dependencies(RunCWrapper helpmestatic)
target_link_libraries(RunCWrapper ${helpmestatic_name})
set_target_properties(RunCWrapper PROPERTIES LINKER_LANGUAGE CXX)
add_test(NAME CWrapperTest COMMAND RunCWrapper)

# CXX DHFR benchmark example
add_executable (DHFRBenchmark dhfr_benchmark.cpp)
configure_file(data/dhfr_charges.txt . COPYONLY)
configure_file(data/dhfr_c6s.txt . COPYONLY)
configure_file(data/dhfr_coords.txt . COPYONLY)
#target_compile_options(DHFRBenchmark  PRIVATE -g)

# CXX example
add_executable (RunCXXWrapper fullexample.cpp)
add_test(NAME CXXWrapperTest COMMAND RunCXXWrapper)

# CXX example using standalone header file
add_executable (RunCXXWrapperStandalone fullexample.cpp)
add_dependencies(RunCXXWrapperStandalone SingleHeader)
target_compile_definitions(RunCXXWrapperStandalone PRIVATE "BUILD_STANDALONE=1")
add_test(NAME CXXWrapperStandaloneTest COMMAND RunCXXWrapperStandalone)

# CXX MPI
if(HAVE_MPI)
    # CXX example
    add_executable (RunCXXMPIWrapper fullexample_parallel.cpp)
    add_test(NAME CXXMPIWrapperTestX
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCXXMPIWrapper 2 1 1 ${MPIEXEC_POSTFLAGS})
    add_test(NAME CXXMPIWrapperTestY
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCXXMPIWrapper 1 2 1 ${MPIEXEC_POSTFLAGS})
    add_test(NAME CXXMPIWrapperTestZ
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCXXMPIWrapper 1 1 2 ${MPIEXEC_POSTFLAGS})

    # C example
    add_executable (RunCMPIWrapper fullexample_parallel.c)
    add_dependencies(RunCMPIWrapper helpmestatic)
    target_link_libraries(RunCMPIWrapper ${helpmestatic_name})
    set_target_properties(RunCMPIWrapper PROPERTIES LINKER_LANGUAGE CXX)
    add_test(NAME CMPIWrapperTestX
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCMPIWrapper 2 1 1 ${MPIEXEC_POSTFLAGS})
    add_test(NAME CMPIWrapperTestY
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCMPIWrapper 1 2 1 ${MPIEXEC_POSTFLAGS})
    add_test(NAME CMPIWrapperTestZ
             COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunCMPIWrapper 1 1 2 ${MPIEXEC_POSTFLAGS})

    # Fortran example
    if(Fortran_ENABLED)
        link_libraries(${MPI_Fortran_LIBRARIES})
        add_executable(RunFortranMPIWrapper ${PROJECT_SOURCE_DIR}/src/helpme.F90 fullexample_parallel.F90 )
        add_dependencies(RunFortranMPIWrapper helpmestatic)
        if(CMAKE_CXX_COMPILER_ID MATCHES "AppleClang")
            set(cpplib c++)
        else()
            set(cpplib stdc++)
        endif()
        target_link_libraries(RunFortranMPIWrapper ${helpmestatic_name} ${cpplib})
        add_test(NAME FortranMPIWrapperTestX
                 COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunFortranMPIWrapper 2 1 1 ${MPIEXEC_POSTFLAGS})
        add_test(NAME FortranMPIWrapperTestY
                 COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunFortranMPIWrapper 1 2 1 ${MPIEXEC_POSTFLAGS})
        add_test(NAME FortranMPIWrapperTestZ
                 COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 2 ${MPIEXEC_PREFLAGS} ./RunFortranMPIWrapper 1 1 2 ${MPIEXEC_POSTFLAGS})
    endif()
endif()

# Fortran example
if(Fortran_ENABLED)
    add_executable(RunFortranWrapper ${PROJECT_SOURCE_DIR}/src/helpme.F90 fullexample.F90 )
    add_dependencies(RunFortranWrapper helpmestatic)
    if(CMAKE_CXX_COMPILER_ID MATCHES "AppleClang")
        set(cpplib c++)
    else()
        set(cpplib stdc++)
    endif()
    target_link_libraries(RunFortranWrapper ${helpmestatic_name} ${cpplib})
    add_test(NAME FortranWrapper COMMAND RunFortranWrapper)
endif()

# Add python tests
if(Python_ENABLED)
    add_test(NAME PythonWrappers
        COMMAND ${PYTHON_EXECUTABLE} setup.py test
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/../python
    )
endif()
