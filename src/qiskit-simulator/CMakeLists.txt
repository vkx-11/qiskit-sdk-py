# CMake config file to build the C++ Simulator
#
# For Windows, we always build static executables. QISKit provides with the 
# necessary external binary libraries (.lib and .dll), these .lib comes from an
# external source and they are simple import files, so we still need the .dll
# files in order to run the final executable. We only support MinGW64 toolchain
# so the static linking makes sure that other required libraries from this
# toolchain are included in the final executable. As MinGW64 can be installed
# anywhere in the system, the user needs to provide the PATH where the pthreads
# library is installed using the USER_LIB_PATH variable, like:
#     C:\..\out> cmake -DUSER_LIB_PATH=C:\path\to\mingw64\lib -G "MinGW Makefiles" ..
#
# For Linux and Mac, we can build both statically or dynamically. The former is the
# default. If you want to build a dynamic executable, you need to set
# STATIC_LINKING to True, example:
#     out$ cmake -DSTATIC_LINKING=False ..
#
# For Mac, you probably need to install static versions of the toolchain in order
# to make a static executable. Additionaly, the OpenMP features are only supported
# on GNU g++ comipler, CLang doesn't include it so if you are building with CLang
# you won't get all the performance.

project(qiskit_simulator VERSION 1.0 LANGUAGES CXX)

set(QISKIT_SIMULATOR_SRC_DIR "${PROJECT_SOURCE_DIR}/src")
set(QISKIT_SIMULATOR_SRC
    "${QISKIT_SIMULATOR_SRC_DIR}/main.cpp")
set(QISKIT_SIMULATOR_EXTERNAL_LIBS
    "${QISKIT_SIMULATOR_SRC_DIR}/third-party/headers"
    "${QISKIT_SIMULATOR_SRC_DIR}/third-party/win64/lib"
	"${USER_LIB_PATH}")

# Target definition
add_executable(qiskit_simulator ${QISKIT_SIMULATOR_SRC})

# Target properties: C++ program
set_target_properties(qiskit_simulator PROPERTIES LINKER_LANGUAGE CXX)

# Toolchain options
set_property(TARGET qiskit_simulator PROPERTY CXX_STANDARD 14)

# Compiler flags
enable_cxx_compiler_flag_if_supported("-O3")
enable_cxx_compiler_flag_if_supported("-march=native")
enable_cxx_compiler_flag_if_supported("-fopenmp")

# We force static linking on Windows
if(STATIC_LINKING OR MINGW)
	# Hack: Seems like enable_cxx_compiler_flag_if_supported() is not properly
	# working on MacOS, when a flag is not supported, it cascades errors
	# to the rest of the flgas being tested... and -static compilation on Mac
	# with gcc is failing...
	if(NOT APPLE)
		enable_cxx_compiler_flag_if_supported("-static")
	endif()
	enable_cxx_compiler_flag_if_supported("-static-libstdc++")
	enable_cxx_compiler_flag_if_supported("-static-libgcc")
endif()

# Warnings and Errors
enable_cxx_compiler_flag_if_supported("-pedantic")
enable_cxx_compiler_flag_if_supported("-Wall")
enable_cxx_compiler_flag_if_supported("-Wfloat-equal")
enable_cxx_compiler_flag_if_supported("-Wundef")
enable_cxx_compiler_flag_if_supported("-Wcast-align")
enable_cxx_compiler_flag_if_supported("-Wwrite-strings")
enable_cxx_compiler_flag_if_supported("-Wmissing-declarations")
enable_cxx_compiler_flag_if_supported("-Wredundant-decls")
enable_cxx_compiler_flag_if_supported("-Wshadow")
enable_cxx_compiler_flag_if_supported("-Woverloaded-virtual")

target_include_directories(qiskit_simulator PRIVATE ${QISKIT_SIMULATOR_SRC_DIR})
target_include_directories(qiskit_simulator PRIVATE ${QISKIT_SIMULATOR_SRC_DIR}/backends)
target_include_directories(qiskit_simulator PRIVATE ${QISKIT_SIMULATOR_SRC_DIR}/engines)
target_include_directories(qiskit_simulator PRIVATE ${QISKIT_SIMULATOR_SRC_DIR}/utilities)
target_include_directories(qiskit_simulator PRIVATE ${QISKIT_SIMULATOR_SRC_DIR}/third-party/headers)

# For header only libraries
SET(CMAKE_FIND_LIBRARY_PREFIXES ${CMAKE_FIND_LIBRARY_PREFIXES} "")
SET(CMAKE_FIND_LIBRARY_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES} .hpp)

# We force static linking on Windows
if(STATIC_LINKING OR MINGW)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .a .lib ${CMAKE_FIND_LIBRARY_SUFFIXES})
endif()

# Looking for external libraries
# This is a header-only library so we don't want it to link it, but we still
# want CMake to find it.
find_library(LIB_JSON
	NAMES json
	PATHS ${QISKIT_SIMULATOR_EXTERNAL_LIBS})
if(NOT LIB_JSON)
    message(FATAL_ERROR "JSON library not found!. Please provide with a USER_LIB_PATH to CMake so it can be searched there")
endif()

find_library(LIB_LAPACK
	NAMES lapack
	PATHS ${QISKIT_SIMULATOR_EXTERNAL_LIBS})
if(NOT LIB_LAPACK)
    message(FATAL_ERROR "LAPACK library not found!. Please provide with a USER_LIB_PATH to CMake so it can be searched there")
endif()

find_library(LIB_BLAS
	NAMES blas
	PATHS ${QISKIT_SIMULATOR_EXTERNAL_LIBS})
if(NOT LIB_BLAS)
    message(FATAL_ERROR "BLAS library not found!. Please provide with a USER_LIB_PATH to CMake so it can be searched there")
endif()

find_library(LIB_THREADS
	NAMES pthread
	PATHS ${QISKIT_SIMULATOR_EXTERNAL_LIBS})
if(NOT LIB_THREADS)
    message(FATAL_ERROR "Pthreads library not found!. Please provide with a USER_LIB_PATH to CMake so it can be searched there")
endif()

set(LIBRARIES PRIVATE ${LIB_BLAS}
                      ${LIB_LAPACK}
					  ${LIB_THREADS})

# Linking
target_link_libraries(qiskit_simulator ${LIBRARIES})

set(QISKIT_SIMULATOR_OUTPUT_DIR $<TARGET_FILE_DIR:qiskit_simulator>
	CACHE INTERNAL "Output directory for building QISKit C++ Simulator")

if(MINGW)
	set(QISKIT_SIMULATOR_THIRD_PARTY_DLLS
		"${QISKIT_SIMULATOR_SRC_DIR}/third-party/win64/dll/libblas.dll"
		"${QISKIT_SIMULATOR_SRC_DIR}/third-party/win64/dll/libgfortran_64-3.dll"
		"${QISKIT_SIMULATOR_SRC_DIR}/third-party/win64/dll/libquadmath_64-0.dll"
		CACHE INTERNAL "Third-party C++ Simulator DLLs")

	foreach(dll_file ${QISKIT_SIMULATOR_THIRD_PARTY_DLLS})
		add_custom_command(
			TARGET qiskit_simulator
			POST_BUILD
			COMMAND ${CMAKE_COMMAND}
			ARGS -E copy ${dll_file} ${QISKIT_SIMULATOR_OUTPUT_DIR}/
		)
		# For 'make clean' target
		get_filename_component(FINAL_FILE ${dll_file} NAME)
		set_property(DIRECTORY APPEND PROPERTY
			ADDITIONAL_MAKE_CLEAN_FILES
				${QISKIT_SIMULATOR_OUTPUT_DIR}/${FINAL_FILE})
	endforeach()
endif()

# Tests
# TODO: Enable them when ready
#add_subdirectory(${QISKIT_SIMULATOR_DIR}/test)