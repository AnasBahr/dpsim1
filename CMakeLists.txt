# Please keep 3.14 as minimum requirement for building on OPAL-RT targets
cmake_minimum_required(VERSION 3.14...3.23)
project(DPsim
		VERSION 1.1.1
		DESCRIPTION "C++ Power System Simulation Library"
		LANGUAGES CXX
		HOMEPAGE_URL "https://dpsim.fein-aachen.org/")

set(PROJECT_AUTHOR "The DPsim Authors.")
set(PROJECT_COPYRIGHT "2017-2023, The DPsim Authors.")
set(PROJECT_DESCRIPTION "C++ Power System Simulation Library")
set(PROJECT_HOMEPAGE_URL "https://dpsim.fein-aachen.org/")
set(PROJECT_VCS_URL "https://github.com/sogno-platform/dpsim")

# Build options
option(WITH_LTO									"Enable Link Time Optimization in Release builds" OFF)
option(WITH_MARCH_NATIVE				"Optimize build for native host architecture" OFF)

# Required for dpsim_python
if(POLICY CMP0076)
	cmake_policy(SET CMP0076 NEW)
endif()

# Solve DOWNLOAD_EXTRACT_TIMESTAMP warning
if(POLICY CMP0135)
	cmake_policy(SET CMP0135 NEW)
endif()

# Allow setting Graphviz_ROOT for windows builds
if(WIN32)
	cmake_policy(SET CMP0074 NEW)
endif()

# Allow shadowing of CMake targets
# As we include dependencies via add_subdirectory()
# we import their targets into the DPsim CMake tree
# which might cause collissions in target names.
set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

# Select C++-Version
# For the pybind11-python interface to compile, a minimum version of C++14 is required
# If you need to compile using C++11, either disable the python interface or modify the source-files in Source/pybind,
# replacing all occurences of py::overload_cast with py::detail::overload_cast_impl as specified on
# https://pybind11.readthedocs.io/en/stable/classes.html#overloaded-methods
set(CMAKE_CXX_STANDARD 17)

if(CMAKE_CXX_COMPILER_ID STREQUAL Intel)
	# Needed by icc, but for some reason not added by CMake
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
endif()

if(NOT WIN32)
	set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

include(CheckCXXCompilerFlag)
if(MSVC)
	# Silence Visual Studio deprecation warnings
	add_definitions(-D_SILENCE_ALL_CXX17_DEPRECATION_WARNINGS)
	add_definitions(-D_SILENCE_EXPERIMENTAL_FILESYSTEM_DEPRECATION_WARNING)

	# Silence Visual Studio error C2220
	add_definitions(-D_SILENCE_STDEXT_ARR_ITERS_DEPRECATION_WARNING)

	# Set exception handling for portability
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")

	check_cxx_compiler_flag("/W4 /WX" CXX_SUPPORTS_WERROR)
	if(CXX_SUPPORTS_WERROR)
	# TODO: activate this again after fixing warnings
	#		set(DPSIM_CXX_FLAGS /W4 /WX)
	endif()
else()
	check_cxx_compiler_flag("-Wall -Werror" CXX_SUPPORTS_WERROR)
	if(CXX_SUPPORTS_WERROR)
		set(DPSIM_CXX_FLAGS -Wall -Werror)
	endif()
endif()

include(CheckSymbolExists)
check_symbol_exists(timerfd_create sys/timerfd.h HAVE_TIMERFD)
check_symbol_exists(getopt_long getopt.h HAVE_GETOPT)
if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
	add_compile_options(-Ofast)

	if(WITH_MARCH_NATIVE)
		add_compile_options(-march=native)
	endif()

	if(WITH_LTO)
		add_compile_options(-flto)
	endif()
endif()

# Get version info and buildid from Git
include(GetVersion)
GetVersion(${PROJECT_SOURCE_DIR} "DPSIM")

# Deprecated build options
# TODO: Remove deprecated options in future release
option(WITH_EIGEN_MODULE     "Fetch Eigen3 as module (deprecated option)" OFF)
option(WITH_KLU   	     "Build with KLU as module (deprecated option)" ON)
option(WITH_SPDLOG_SUBMODULE "Fetch spdlog as module (deprecated option)" OFF)
option(WITH_CIM_SUBMODULE    "Fetch CIMpp as module (deprecated option)" OFF)
option(WITH_PYBIND_MODULE    "Fetch pybind11 as module (deprecated option)" OFF)
option(GET_GRID_DATA         "Fetch grid data (deprecated option)" ON)

# Build options
option(FETCH_EIGEN          "Fetch Eigen3 as module" ${WITH_EIGEN_MODULE})
option(FETCH_KLU            "Fetch KLU as module" ${WITH_KLU})
option(FETCH_SPDLOG         "Fetch spdlog as module" ${WITH_SPDLOG_SUBMODULE})
option(FETCH_CIMPP          "Fetch CIMpp as module" ${WITH_CIM_SUBMODULE})
option(FETCH_PYBIND         "Fetch pybind11 as module" ${WITH_PYBIND_MODULE})
option(FETCH_GRID_DATA      "Fetch grid data" ${GET_GRID_DATA})
option(FETCH_FILESYSTEM     "Fetch standalone implememtation of std::filesystem" OFF)

option(WITH_JSON            "Include json library" ON)
option(WITH_PROFILING       "Add `-pg` profiling flag to compiliation" OFF)
option(WITH_ASAN            "Adds compiler flags to use the address sanitizer" OFF)
option(WITH_TSAN            "Adds compiler flags to use the thread sanitizer" OFF)
option(WITH_SPARSE          "Use sparse matrices in MNA-Solver"	ON)

option(BUILD_SHARED_LIBS    "Build shared library" OFF)
option(DPSIM_BUILD_EXAMPLES "Build C++ examples" ON)
option(DPSIM_BUILD_DOC      "Build documentation" ON)

option(CGMES_BUILD          "Build with CGMES instead of CIMpp" OFF)

find_package(Threads REQUIRED)
find_package(Sundials)
find_package(OpenMP)
find_package(CUDA)
find_package(GSL)
find_package(Graphviz)
find_package(VILLASnode)
find_package(MAGMA)
find_package(Python3 COMPONENTS Interpreter Development)

if(FETCH_FILESYSTEM)
	include(FetchFilesystem)
else()
	find_package(Filesystem REQUIRED)
endif()

if(FETCH_EIGEN OR WIN32)
	include(FetchEigen)
else()
	find_package(Eigen3 3.0 REQUIRED)
endif()

if (FETCH_KLU)
	include(FetchKLU)
endif()

if(FETCH_SPDLOG OR WIN32)
	include(FetchSpdlog)
else()
	find_package(spdlog 1.5.0 REQUIRED)
	find_package(fmt 6.1.2 REQUIRED)
endif()

if(FETCH_CIMPP OR WIN32)
	include(FetchCIMpp)
else()
	find_package(CIMpp)
endif()

if(FETCH_PYBIND)
	include(FetchPybind)
else()
	find_package(pybind11 CONFIG QUIET)
endif()

if(FETCH_GRID_DATA)
	include(FetchGridData)
endif()

if(WITH_JSON)
	include(FetchJsonLibrary)
endif()

if(WITH_PROFILING)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg")
	list(APPEND DPSIM_CXX_FLAGS "-pg")
endif()

if(WITH_ASAN)
	if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -fsanitize=pointer-compare -fsanitize=pointer-subtract -fsanitize-address-use-after-scope")
	else()
		message(WARNING "WITH_ASAN is only supported when using GCC")
	endif()
endif()

if(WITH_TSAN)
	if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread")
	else()
		message(WARNING "WITH_TSAN is only supported when using GCC")
	endif()
endif()

include(FetchReaderWriterQueue)

if("${CMAKE_SYSTEM}" MATCHES "Linux")
	set(Linux_FOUND ON)
elseif("${CMAKE_SYSTEM}" MATCHES "Darwin")
	set(MacOS_FOUND ON)
endif()

include(CMakeDependentOption)
cmake_dependent_option(WITH_SUNDIALS        "Enable sundials solver suite"          ON  "Sundials_FOUND"   OFF)
cmake_dependent_option(WITH_VILLAS          "Enable VILLASnode interface"           ON  "VILLASnode_FOUND" OFF)
cmake_dependent_option(WITH_RT              "Enable real-time features"             ON  "Linux_FOUND"      OFF)
cmake_dependent_option(WITH_CIM             "Enable support for parsing CIM"        ON  "CIMpp_FOUND"      OFF)
cmake_dependent_option(WITH_OPENMP          "Enable OpenMP-based parallelisation"   ON  "OPENMP_FOUND"     OFF)
cmake_dependent_option(WITH_CUDA            "Enable CUDA-based parallelisation"     OFF "CUDA_FOUND"       OFF)
cmake_dependent_option(WITH_GRAPHVIZ        "Enable Graphviz Graphs"                ON  "GRAPHVIZ_FOUND"   OFF)
cmake_dependent_option(WITH_PYBIND          "Enable PYBIND support"                 ON  "pybind11_FOUND"   OFF)
cmake_dependent_option(WITH_MAGMA           "Enable MAGMA features"                 ON  "MAGMA_FOUND"      OFF)
cmake_dependent_option(WITH_MNASOLVERPLUGIN "Enable MNASolver Plugins"              ON  "NOT WIN32"        OFF)

if(WITH_CUDA)
	# BEGIN OF WORKAROUND - enable cuda dynamic linking.
	# Starting with Cmake 3.17 we can use
	# set(CMAKE_CUDA_RUNTIME_LIBRARY_DEFAULT "SHARED") instead
	set(CMAKE_CUDA_FLAGS "" CACHE STRING "")
	if(CMAKE_CUDA_FLAGS)
		list(REMOVE_ITEM CMAKE_CUDA_FLAGS "-cudart static")
	endif()
	string(APPEND CMAKE_CUDA_FLAGS "-cudart shared")
	# END OF WORKAROUND

	enable_language(CUDA)

	# BEGIN OF WORKAROUND - enable cuda dynamic linking.
	if(CMAKE_CUDA_HOST_IMPLICIT_LINK_LIBRARIES)
		list(REMOVE_ITEM CMAKE_CUDA_HOST_IMPLICIT_LINK_LIBRARIES "cudart_static")
		list(REMOVE_ITEM CMAKE_CUDA_HOST_IMPLICIT_LINK_LIBRARIES "cudadevrt")
		list(APPEND CMAKE_CUDA_HOST_IMPLICIT_LINK_LIBRARIES "cudart")
	endif()
	if(CMAKE_CUDA_IMPLICIT_LINK_LIBRARIES)
		list(REMOVE_ITEM CMAKE_CUDA_IMPLICIT_LINK_LIBRARIES "cudart_static")
		list(REMOVE_ITEM CMAKE_CUDA_IMPLICIT_LINK_LIBRARIES "cudadevrt")
		list(APPEND CMAKE_CUDA_IMPLICIT_LINK_LIBRARIES "cudart")
	endif()
	if(CUDA_LIBRARIES)
		list(REMOVE_ITEM CUDA_LIBRARIES "${CUDA_cudart_static_LIBRARY}")
		list(APPEND CUDA_LIBRARIES "${CUDA_CUDART_LIBRARY}")
	endif()
	# END OF WORKAROUND
endif()

add_subdirectory(dpsim-models)
add_subdirectory(dpsim)

if(DPSIM_BUILD_DOC)
	add_subdirectory(docs)
endif()

if(WITH_VILLAS)
	add_subdirectory(dpsim-villas)
endif()

if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.16")
	message(VERBOSE "Compiler Release Flags: " ${CMAKE_CXX_FLAGS_RELEASE})
	message(VERBOSE "Compiler Debug Flags:   " ${CMAKE_CXX_FLAGS_DEBUG})
endif()

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
	include(FeatureSummary)
	add_feature_info(CIM             WITH_CIM             "Loading Common Information Model (CIM) files")
	add_feature_info(CUDA            WITH_CUDA            "CUDA-based parallelisation")
	add_feature_info(Graphviz        WITH_GRAPHVIZ        "Graphviz graphs")
	add_feature_info(GSL             WITH_GSL             "GNU Scientific library")
	add_feature_info(JSON            WITH_JSON            "JSON library")
	add_feature_info(MAGMA           WITH_MAGMA           "MAGMA features")
	add_feature_info(MNASolverPlugin WITH_MNASOLVERPLUGIN "MNASolver Plugins")
	add_feature_info(OpenMP          WITH_OPENMP          "OpenMP-based parallelisation")
	add_feature_info(PyBind          WITH_PYBIND          "PyBind module")
	add_feature_info(RealTime        WITH_RT              "Extended real-time features")
	add_feature_info(Sundials        WITH_SUNDIALS        "Sundials solvers")
	add_feature_info(VILLASnode      WITH_VILLAS          "Interface DPsim solvers via VILLASnode interfaces")
	add_feature_info(KLU         		 WITH_KLU             "Use custom KLU module")

	feature_summary(WHAT ALL VAR enabledFeaturesText)

	if(FOUND_GIT_VERSION)
		message(STATUS "Building ${CMAKE_PROJECT_NAME}:")
		message(STATUS "Feature summary for DPsim")
		message(STATUS "${enabledFeaturesText}")
		message(STATUS "  VERSION:         ${DPSIM_VERSION}")
		message(STATUS "  RELEASE:         ${DPSIM_RELEASE}")
		message(STATUS "  GIT_REV:         ${DPSIM_GIT_REV}")
		message(STATUS "  GIT_BRANCH:      ${DPSIM_GIT_BRANCH}")
		message(STATUS "  VARIANT:         ${DPSIM_VARIANT}")
		message(STATUS "  BUILD_ID:        ${DPSIM_BUILD_ID}")
		message(STATUS "  BUILD_DATE:      ${DPSIM_BUILD_DATE}")
	endif()
endif()

include(DPsimPackaging)
