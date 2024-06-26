CMAKE_MINIMUM_REQUIRED(VERSION 3.14)

# Mac rpath Policy
IF(POLICY CMP0042)
          CMAKE_POLICY(SET CMP0042 NEW)
ENDIF(POLICY CMP0042)
SET(CMAKE_MACOSX_RPATH ON)

# Linux rpath settings
SET(CMAKE_SKIP_BUILD_RPATH FALSE)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

SET(CMAKE_POSITION_INDEPENDENT_CODE ON)

#SET(CMAKE_C_FLAGS "-Wall")

# Configuration Options
OPTION(BUILD_SHARED_LIBS "Whether to build shared libraries" ON)
MESSAGE(STATUS "BUILD_SHARED_LIBS: ${BUILD_SHARED_LIBS}")

if (BUILD_SHARED_LIBS)
   OPTION(REDSET_LINK_STATIC "Default to static linking? (Needed for Cray)" OFF)
else()
   OPTION(REDSET_LINK_STATIC "Default to static linking? (Needed for Cray)" ON)
endif()
MESSAGE(STATUS "REDSET_LINK_STATIC: ${REDSET_LINK_STATIC}")

OPTION(ENABLE_OPENMP "Whether to enable OpenMP support" OFF)
MESSAGE(STATUS "ENABLE_OPENMP: ${ENABLE_OPENMP}")

OPTION(ENABLE_PTHREADS "Whether to enable pthreads support" OFF)
MESSAGE(STATUS "ENABLE_PTHREADS: ${ENABLE_PTHREADS}")

OPTION(ENABLE_CUDA "Whether to enable CUDA support" OFF)
MESSAGE(STATUS "ENABLE_CUDA: ${ENABLE_CUDA}")

OPTION(ENABLE_TESTS "Whether to build tests" ON)
MESSAGE(STATUS "ENABLE_TESTS: ${ENABLE_TESTS}")

IF(ENABLE_CUDA)
    # we define the project languages conditionally since listing CUDA
    # causes the github actions build to fail since nvcc is not available
    # and adding nvcc is complicated
    PROJECT(REDSET VERSION 0.4.0 LANGUAGES C CUDA)

    # if nvcc can be added, then one can move the project line
    # back to the top, always list CUDA, and selectively enable it
    #   https://cliutils.gitlab.io/modern-cmake/chapters/packages/CUDA.html
    #ENABLE_LANGUAGE(CUDA)

    SET(HAVE_CUDA TRUE)
ELSE(ENABLE_CUDA)
    PROJECT(REDSET VERSION 0.4.0 LANGUAGES C)
ENDIF(ENABLE_CUDA)

# Find Packages & Files

LIST(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

## KVTREE
FIND_PACKAGE(kvtree REQUIRED)
LIST(APPEND REDSET_EXTERNAL_LIBS kvtree::kvtree)
LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS kvtree::kvtree-static)
LIST(APPEND REDSET_SERIAL_LIBS kvtree::kvtree_base)
LIST(APPEND REDSET_SERIAL_STATIC_LIBS kvtree::kvtree_base-static)

## RankStr
FIND_PACKAGE(rankstr REQUIRED)
LIST(APPEND REDSET_EXTERNAL_LIBS rankstr::rankstr)
LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS rankstr::rankstr-static)

## MPI
INCLUDE(SetupMPI)
IF(MPI_C_FOUND)
    IF(ENABLE_CUDA)
        # HACK: IBM MPI compiler wrappers add -pthread, which trips up nvcc.
        # A work around is to just drop it.
        GET_TARGET_PROPERTY(MPI_C_TMP MPI::MPI_C INTERFACE_LINK_LIBRARIES)
        STRING(REPLACE " -pthread" "" MPI_C_TMP "${MPI_C_TMP}")
        SET_TARGET_PROPERTIES(MPI::MPI_C PROPERTIES
          INTERFACE_LINK_LIBRARIES "${MPI_C_TMP}"
        )
    ENDIF(ENABLE_CUDA)

    LIST(APPEND REDSET_EXTERNAL_LIBS MPI::MPI_C)
    LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS MPI::MPI_C)
ELSE(MPI_C_FOUND)
    MESSAGE(FATAL_ERROR
        "Could not find MPI! "
        "Either add an MPI compiler to your path "
        "or force CMake to build using the correct compiler (`export CC=mpicc`).")
ENDIF(MPI_C_FOUND)

## ZLIB
FIND_PACKAGE(ZLIB REQUIRED)
INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIRS})
LIST(APPEND REDSET_EXTERNAL_LIBS ZLIB::ZLIB)
LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS ZLIB::ZLIB)
LIST(APPEND REDSET_SERIAL_LIBS ZLIB::ZLIB)
LIST(APPEND REDSET_SERIAL_STATIC_LIBS ZLIB::ZLIB)

IF(ENABLE_OPENMP)
    FIND_PACKAGE(OpenMP)
    IF(OpenMP_FOUND)
        SET(HAVE_OPENMP TRUE)
        LIST(APPEND REDSET_EXTERNAL_LIBS OpenMP::OpenMP_C)
        LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS OpenMP::OpenMP_C)
    ENDIF(OpenMP_FOUND)
ENDIF(ENABLE_OPENMP)

# PTHREADS
IF(ENABLE_PTHREADS)
  FIND_PACKAGE(Threads REQUIRED)
  IF(CMAKE_USE_PTHREADS_INIT)
      SET(HAVE_PTHREADS TRUE)
      LIST(APPEND REDSET_EXTERNAL_LIBS Threads::Threads)
      LIST(APPEND REDSET_EXTERNAL_STATIC_LIBS Threads::Threads)
  ENDIF()
ENDIF(ENABLE_PTHREADS)

## HEADERS
INCLUDE(CheckIncludeFile)
INCLUDE(GNUInstallDirs)

# Subdirectories
ADD_SUBDIRECTORY(src)

IF(ENABLE_TESTS)
   INCLUDE(CTest)
   INCLUDE(REDSET_ADD_TEST_PARALLEL)
   INCLUDE(REDSET_ADD_TEST_SERIAL)
   ADD_SUBDIRECTORY(test)
ENDIF(ENABLE_TESTS)

# Generate config.h with all our build #defs
CONFIGURE_FILE(${PROJECT_SOURCE_DIR}/cmake/config.h.in ${PROJECT_BINARY_DIR}/config.h)
SET_SOURCE_FILES_PROPERTIES(${PROJECT_BINARY_DIR}/config.h PROPERTIES GENERATED TRUE)

# Special Install Files
# INSTALL(FILES README.md DESTINATION ${CMAKE_INSTALL_DATADIR}/kvtree)

# CMake configs

# Install the exports file
INSTALL(EXPORT redsetTargets FILE redsetTargets.cmake NAMESPACE redset:: DESTINATION share/redset/cmake)

# Export so we can use in-tree
EXPORT(EXPORT redsetTargets NAMESPACE redset:: FILE redsetTargets.cmake)

# Setup the package config
INCLUDE(CMakePackageConfigHelpers)
CONFIGURE_PACKAGE_CONFIG_FILE(cmake/redsetConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/redsetConfig.cmake INSTALL_DESTINATION share/redset/cmake)
WRITE_BASIC_PACKAGE_VERSION_FILE(${CMAKE_CURRENT_BINARY_DIR}/redsetConfigVersion.cmake COMPATIBILITY SameMajorVersion)

# Install package config
INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/redsetConfig.cmake ${CMAKE_CURRENT_BINARY_DIR}/redsetConfigVersion.cmake DESTINATION share/redset/cmake)

# Package
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "REDSET Redundancy Set for encoding/decoding a set of files")
SET(CPACK_PACKAGE_VENDOR "Lawrence Livermore National Laboratory")
# SET(CPACK_PACKAGE_DESCRIPTION_FILE "${PROJECT_SOURCE_DIR}/README.md")
# SET(CPACK_RESOURCE_FILE_LICENSE "${PROJECT_SOURCE_DIR}/LICENSE.TXT")
