#The name of the project
PROJECT(gpuNUFFT)

#The CMake Minimum version that is required. The FindCUDA script
#is distributed since version 2.8
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${PROJECT_SOURCE_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${PROJECT_SOURCE_DIR}/bin)

#for older CMAKE versions these vars have to be set
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
SET(ARCHIVE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

#patch output path
SET(PATCH_OUTPUT_DIR ${PROJECT_SOURCE_DIR}/deploy)
SET(PATCH_OUTPUT_PATH ${PATCH_OUTPUT_DIR}/gpuNUFFT.zip)

#mex file export dir
SET(MEX_EXPORT_DIR ${PROJECT_SOURCE_DIR}/../gpuNUFFT/@gpuNUFFT/private CACHE STRING "Folder in which mex files are exported to.")

#Searching CUDA
FIND_PACKAGE(CUDA REQUIRED)

#Searching MATLAB
if(WIN32)
  SET(MATLAB_ROOT_DIR "C:\\Program Files\\MATLAB\\R2011a" CACHE STRING "MATLAB Installation Directory")
elseif(UNIX)
  SET(MATLAB_ROOT_DIR "/usr/local/MATLAB/R2010b" CACHE STRING "MATLAB Installation Directory")
endif()

SET(MATLAB_DIR = "${MATLAB_ROOT_DIR}")

FIND_PACKAGE(Matlab)

IF (EXISTS "${MATLAB_ROOT_DIR}")
  MESSAGE(STATUS "MATLAB Installation found in ${MATLAB_ROOT_DIR}")
else()
  MESSAGE(FATAL_ERROR "Please set variable MATLAB_ROOT_DIR correctly!")
endif()

#Options
#General DEBUG output 
SET (DEBUG false)
OPTION(WITH_DEBUG "Enable DEBUG messages" OFF)
if (WITH_DEBUG)
 SET (DEBUG true)
endif()
MESSAGE(STATUS "Setting DEBUG Option to ${DEBUG}")

#Matlab DEBUG output
SET (MATLAB_DEBUG false)
OPTION(WITH_MATLAB_DEBUG "Enable DEBUG messages for MATLAB calls" OFF)
if (WITH_MATLAB_DEBUG)
 SET (MATLAB_DEBUG true)
endif()
MESSAGE(STATUS "Setting MATLAB DEBUG Option to ${MATLAB_DEBUG}")

#Enable ATOMIC kernel  
SET(GEN_ATOMIC ON CACHE BOOL "Enable atomic kernel generation (Compute Capability 2.0 needed). Only turn it off when old architectures (<2.0) have to be supported.")

#Enable Google Tests
SET(GEN_TESTS OFF CACHE BOOL "Enable and generate simple GOOGLE test framework unit tests")

#Enable/Disable GPU double precision 
SET(GPU_DOUBLE_PREC OFF CACHE BOOL "Enable double precision floating point operations on GPU (Compute Capability 1.3 needed)")

if(GPU_DOUBLE_PREC)
  SET(PREC_SUFFIX "_d")
else(GPU_DOUBLE_PREC)
  SET(PREC_SUFFIX "_f")
endif(GPU_DOUBLE_PREC)

IF(CMAKE_BUILD_TYPE MATCHES Debug)
  MESSAGE("debug mode")
  SET(CUDA_NVCC_FLAGS "-arch;sm_20;--ptxas-options=-v")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -g")
ELSE(CMAKE_BUILD_TYPE)
  SET(CUDA_NVCC_FLAGS "-arch;sm_20;")
ENDIF()

MESSAGE(STATUS "setting NVCC FLAGS to: ${CUDA_NVCC_FLAGS}")

#LIB and MEX-file names
SET(GRID_LIB_NAME "gpuNUFFT${PREC_SUFFIX}")
SET(GRID_LIB_ATM_NAME "gpuNUFFT_ATM${PREC_SUFFIX}")

SET(GRID_MEX_FORW_NAME "mex_gpuNUFFT_forw${PREC_SUFFIX}")
SET(GRID_MEX_ADJ_NAME "mex_gpuNUFFT_adj${PREC_SUFFIX}")

SET(GRID_MEX_FORW_ATM_NAME "mex_gpuNUFFT_forw_atomic${PREC_SUFFIX}")
SET(GRID_MEX_ADJ_ATM_NAME "mex_gpuNUFFT_adj_atomic${PREC_SUFFIX}")

#Precomputation MEX file
SET(GRID_MEX_PRECOMP_NAME "mex_gpuNUFFT_precomp${PREC_SUFFIX}")

MESSAGE(STATUS "creating lib with name: ${GRID_LIB_NAME}")
SET(WARNING "/* WARNING: Automatically generated file. Please do not modify this file. */")
CONFIGURE_FILE( ${CMAKE_SOURCE_DIR}/inc/config.hpp.cmake ${CMAKE_SOURCE_DIR}/inc/config.hpp)
CONFIGURE_FILE( ${CMAKE_SOURCE_DIR}/inc/cufft_config.hpp.cmake ${CMAKE_SOURCE_DIR}/inc/cufft_config.hpp)

#Include dirs
include_directories(inc)
SET(GPUNUFFT_INC_DIR ${CMAKE_SOURCE_DIR}/inc)
SET(GPUNUFFT_INCLUDE ${GPUNUFFT_INC_DIR}/cuda_utils.hpp 
										 ${GPUNUFFT_INC_DIR}/cuda_utils.cuh
										 ${GPUNUFFT_INC_DIR}/config.hpp
										 ${GPUNUFFT_INC_DIR}/gpuNUFFT_utils.hpp
										 ${GPUNUFFT_INC_DIR}/gpuNUFFT_types.hpp
										 ${GPUNUFFT_INC_DIR}/gpuNUFFT_kernels.hpp
										 ${GPUNUFFT_INC_DIR}/precomp_kernels.hpp
										 ${GPUNUFFT_INC_DIR}/precomp_utils.hpp
                     ${GPUNUFFT_INC_DIR}/gpuNUFFT_operator.hpp
										 ${GPUNUFFT_INC_DIR}/balanced_operator.hpp
										 ${GPUNUFFT_INC_DIR}/texture_gpuNUFFT_operator.hpp
										 ${GPUNUFFT_INC_DIR}/balanced_gpuNUFFT_operator.hpp
                     ${GPUNUFFT_INC_DIR}/gpuNUFFT_operator_factory.hpp
										 ${GPUNUFFT_INC_DIR}/balanced_texture_gpuNUFFT_operator.hpp)
					 
SET(MATLAB_HELPER_INCLUDE ${GPUNUFFT_INC_DIR}/matlab_helper.h)
SET(CONFIG_INCLUDE ${GPUNUFFT_INC_DIR}/config.hpp ${GPUNUFFT_INC_DIR}/cufft_config.hpp)

#Adding src directory to the project

ADD_SUBDIRECTORY(src)
if(GEN_TESTS)
	# For make-based builds, defines make target named test.
	# For Visual Studio builds, defines Visual Studio project named RUN_TESTS.
	enable_testing()
	ADD_SUBDIRECTORY(test)
	add_test(
		NAME runUnitTests
		COMMAND runUnitTests
	)
	add_test(
		NAME runGPUUnitTests
		COMMAND runGPUUnitTests
	)

	if(GEN_ATOMIC)
		add_test(
			NAME runGPUATMUnitTests
			COMMAND runGPUATMUnitTests)
	endif(GEN_ATOMIC)
endif(GEN_TESTS)

ADD_SUBDIRECTORY(doc)

#CREATE zip Archive using JAVA in order to deploy project patch
if (GEN_ZIP)
	find_package(Java)

	MESSAGE(${PATCH_OUTPUT_PATH})
	execute_process(
	    COMMAND 
		"find" "${CMAKE_CURRENT_SOURCE_DIR}/src" "-not" "-name" "CMakeLists.txt" "-not" "-type" "d" 
OUTPUT_VARIABLE _file_list
	)
#MESSAGE(${_file_list})	
	execute_process( 
    COMMAND "${Java_JAR_EXECUTABLE}" "cfM" "${PATCH_OUTPUT_PATH}" "`" 
		"find" "${CMAKE_CURRENT_SOURCE_DIR}/src" "-not" "-name" "CMakeLists.txt" "-not" "-type" "d" "`"
#		"-C" ${CMAKE_CURRENT_SOURCE_DIR} "inc" 
#		"-C" ${CMAKE_CURRENT_SOURCE_DIR} "matlab/gpuNUFFT"
	    RESULT_VARIABLE _result
	)
endif(GEN_ZIP)

