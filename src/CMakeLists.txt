
# Copyright (c) 2012, 2013 openMVG authors.

# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

PROJECT(openMVG C CXX)

# ==============================================================================
# OpenMVG build options
# ==============================================================================
OPTION(OpenMVG_BUILD_TESTS "Build OpenMVG tests" OFF)
OPTION(OpenMVG_BUILD_EXAMPLES "Build OpenMVG example applications. Does not affect binaries under 'software'" ON)
option(OpenMVG_BUILD_COVERAGE "Enable code coverage generation (gcc only)" OFF)
OPTION(OpenMVG_USE_OPENMP "Enable OpenMP parallelization" ON)

# By default build in Release mode
IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE "Release")
ENDIF(NOT CMAKE_BUILD_TYPE)

IF (WIN32)
  ADD_DEFINITIONS(-DNOMINMAX)
ENDIF (WIN32)

# ==============================================================================
# Additional cmake find modules
# ==============================================================================
SET(CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmakeFindModules)
INCLUDE(OptimizeForArchitecture)
OptimizeForArchitecture()
IF (SSE2_FOUND)
  IF (MSVC AND NOT ${CMAKE_CL_64})
    ADD_DEFINITIONS(/arch:SSE2)
  ENDIF (MSVC AND NOT ${CMAKE_CL_64})
  ADD_DEFINITIONS(-DUSE_SSE)
ENDIF (SSE2_FOUND)
IF (UNIX AND NOT OpenMVG_BUILD_COVERAGE)
  SET(CMAKE_C_FLAGS_RELEASE "-O3")
  SET(CMAKE_CXX_FLAGS_RELEASE "-O3")
ENDIF ()

# ==============================================================================
# OpenMP detection
# ==============================================================================
IF (OpenMVG_USE_OPENMP)
  FIND_PACKAGE(OpenMP)
  IF (OPENMP_FOUND)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    OPTION(OpenMVG_USE_OPENMP "Use OpenMP for parallelization" ON)
    ADD_DEFINITIONS(-DUSE_OPENMP)
    IF (NOT MSVC)
      LIST(APPEND OPENMVG_LIBRARY_DEPENDENCIES gomp)
    ENDIF (NOT MSVC)
ELSE (OpenMVG_USE_OPENMP)
    OPTION(OpenMVG_USE_OPENMP "Use OpenMP for parallelization" OFF)
    SET(OpenMVG_USE_OPENMP OFF CACHE BOOL "Use OpenMP for parallelization")
    REMOVE_DEFINITIONS(-DUSE_OPENMP)
  ENDIF (OPENMP_FOUND)
ENDIF (OpenMVG_USE_OPENMP)

# ==============================================================================
# enable code coverage generation (only with GCC)
# ==============================================================================
IF(OpenMVG_BUILD_COVERAGE AND CMAKE_COMPILER_IS_GNUCXX)
  MESSAGE("OpenMVG_BUILD_COVERAGE enabled")
  SET(CMAKE_BUILD_TYPE "Debug")
  ADD_DEFINITIONS(--coverage -fprofile-arcs -ftest-coverage)
  SET(CMAKE_EXE_LINKER_FLAGS
    "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
ENDIF()

# ==============================================================================
# IMAGE IO detection
# ==============================================================================
IF (NOT APPLE)
  FIND_PACKAGE(JPEG QUIET)
  FIND_PACKAGE(PNG QUIET)
ENDIF (NOT APPLE)

# ==============================================================================
# SUBMODULE CONFIGURATION
# ==============================================================================
#- glfw
# ==============================================================================
SET(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "Do not build the GLFW example programs")
SET(GLFW_BUILD_TESTS    OFF CACHE BOOL "Do not build the GLFW tests programs")
SET(GLFW_BUILD_DOCS     OFF CACHE BOOL "Do not build the GLFW documentation")
ADD_SUBDIRECTORY(dependencies/glfw)
INCLUDE_DIRECTORIES(SYSTEM dependencies/glfw/include)

#===============================
#---- LINEAR PROGRAMMING SOLVER
#===============================

#- Mosek (linear programming interface)
#===============================================================================
FIND_PACKAGE(Mosek)
IF(MOSEK_FOUND)
  ADD_DEFINITIONS(-DOPENMVG_HAVE_MOSEK)
  INCLUDE_DIRECTORIES(
    ${MOSEK_INCLUDE}
    ./dependencies/osi_clp/Osi/src/OsiMsk/
  )
ENDIF(MOSEK_FOUND)

#- osi_clp (linear programming interface)
# ==============================================================================
ADD_SUBDIRECTORY(dependencies/osi_clp)
INCLUDE_DIRECTORIES(
  ./dependencies/osi_clp/CoinUtils/src/
  ./dependencies/osi_clp/Clp/src/
  ./dependencies/osi_clp/Osi/src/Osi/
  ./dependencies/osi_clp/Clp/src/OsiClp/
)

#===============================
#--END-- LINEAR PROGRAMMING SOLVER
#===============================


# ==============================================================================
# Opencv is not used by openMVG but some samples show how to use openCV
#  and openMVG simultaneously
# ==============================================================================
SET(USE_OPENCV OFF CACHE BOOL "Do not build opencv+openMVG samples programs")
IF (USE_OPENCV)
# In order use openMVG+openCV samples set USE_OPENCV to ON and set the OpenCV_DIR
#  to the build directory of openCV. Prefer use static version of openCV libraries.
  SET(OpenCV_DIR "/home/pierre/Documents/Dev_OpenCV/opencv_Build")
ENDIF (USE_OPENCV)



# ==============================================================================
# Enable cmake UNIT TEST framework
# ==============================================================================
IF (OpenMVG_BUILD_TESTS)
  ENABLE_TESTING()
ENDIF (OpenMVG_BUILD_TESTS)
# MACRO to ease UnitTesting
MACRO (UNIT_TEST NAMESPACE NAME EXTRA_LIBS)
  IF (OpenMVG_BUILD_TESTS)
    ADD_EXECUTABLE(${NAMESPACE}_test_${NAME} ${NAME}_test.cpp)
    TARGET_LINK_LIBRARIES(${NAMESPACE}_test_${NAME}
                          ${EXTRA_LIBS} # Extra libs MUST be first.
                          CppUnitLite ${OPENMVG_LIBRARY_DEPENDENCIES})
    ADD_TEST(${NAMESPACE}_test_${NAME} ${NAMESPACE}_test_${NAME})
  ENDIF (OpenMVG_BUILD_TESTS)
ENDMACRO (UNIT_TEST)


# Configure Eigen to use only MPL2 licensed code
ADD_DEFINITIONS(-DEIGEN_MPL2_ONLY)

# ==============================================================================
# Third-party libraries:
# ==============================================================================
ADD_SUBDIRECTORY(third_party)

# ==============================================================================
# Include directories
# ==============================================================================
SET(OpenMVG_INCLUDES
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/
  ${JPEG_INCLUDE_DIR}
  ${PNG_INCLUDE_DIRS}
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/eigen
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/lemon
  ${PROJECT_BINARY_DIR}/third_party/lemon
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/ceres-solver/include
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/ceres-solver/internal/ceres/miniglog
  ${PROJECT_BINARY_DIR}/third_party/ceres-solver/config
  ${CMAKE_CURRENT_SOURCE_DIR}/third_party/flann/src/cpp
  CACHE PATH "include directories for openMVG and its dependencies"
)

INCLUDE_DIRECTORIES(${OpenMVG_INCLUDES})

# ==============================================================================
# openMVG modules
# ==============================================================================
# The openMVG library itself
ADD_SUBDIRECTORY(openMVG)

# software under patent
# Included for research purpose only
ADD_SUBDIRECTORY(patented)

# openMVG tutorial examples
IF (OpenMVG_BUILD_EXAMPLES)
  ADD_SUBDIRECTORY(openMVG_Samples)
ENDIF (OpenMVG_BUILD_EXAMPLES)

# ==============================================================================
# openMVG libraries
# ==============================================================================
SET(OpenMVG_LIBS
  openMVG_image
  openMVG_multiview
  openMVG_lInftyComputerVision)
# concat lists
SET(OpenMVG_LIBS "${OpenMVG_LIBS}" ${openMVG_linearProgramming})
SET(OpenMVG_LIBS "${OpenMVG_LIBS}"
  openMVG_system
  #third_party libs
  vlsift
  stlplus
  flann_cpp_s
  ceres
  lemon)
SET(OpenMVG_LIBS "${OpenMVG_LIBS}"
  ${OPENMVG_LIBRARY_DEPENDENCIES} CACHE STRING "openMVG library names")

# Complete software build on openMVG
ADD_SUBDIRECTORY(software)

# ==============================================================================
# Documentation
# --------------------------
# Sphinx detection
# ==============================================================================
FIND_PACKAGE(Sphinx)
IF (EXISTS ${SPHINX_EXECUTABLE})
  SET(SPHINX_HTML_DIR "${CMAKE_CURRENT_BINARY_DIR}/htmlDoc")

  CONFIGURE_FILE(
    "${CMAKE_CURRENT_SOURCE_DIR}/../docs/sphinx/rst/conf.py"
    "${CMAKE_CURRENT_BINARY_DIR}/conf.py"
    @ONLY)

  ADD_CUSTOM_TARGET(doc ALL
    ${SPHINX_EXECUTABLE}
    -b html
    "${CMAKE_CURRENT_SOURCE_DIR}/../docs/sphinx/rst"
    "${SPHINX_HTML_DIR}"
    COMMENT "Building HTML documentation with Sphinx")

ELSE (EXISTS ${SPHINX_EXECUTABLE})
  MESSAGE("Sphinx need to be installed to generate the documentation")
ENDIF (EXISTS ${SPHINX_EXECUTABLE})

