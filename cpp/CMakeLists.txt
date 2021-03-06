CMAKE_MINIMUM_REQUIRED(VERSION 3.10.2)
PROJECT(skdecide)

OPTION(BUILD_TESTS OFF)
OPTION(ONLY_PYTHON OFF)
OPTION(BUILD_PYTHON_BINDING ON)

ENABLE_LANGUAGE(CXX)
SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_POSITION_INDEPENDENT_CODE ON)
SET(CMAKE_CXX_FLAGS "-fvisibility=hidden -fvisibility-inlines-hidden ${CMAKE_CXX_FLAGS}")

SET(LIBS "")
SET(INTERFACES "")
SET(INCLUDE_DIRS "")

# /!\ USING TBB INTRODUCES STRANGE BUGS LIKE IF THE TBB MALLOC CLASHES
#     WITH THE CPYTHON MALLOC, THUS DEACTIVATING IT FOR NOW
# # For multi-threading with C++-17
# FIND_PACKAGE(TBB)
# IF (TBB_FOUND)
#     CMAKE_POLICY(SET CMP0075 NEW)
#     SET(CMAKE_REQUIRED_LIBRARIES "${TBB_IMPORTED_TARGETS}")
#     INCLUDE(CheckIncludeFileCXX)
#     CHECK_INCLUDE_FILE_CXX(execution HAS_EXECUTION)
#     IF (HAS_EXECUTION)
#         MESSAGE ("Using C++-17 and TBB multi-threading support")
#         LIST(APPEND LIBS "${TBB_IMPORTED_TARGETS}")
#     ENDIF ()
#     SET(CMAKE_REQUIRED_LIBRARIES "")
# ENDIF ()
IF (NOT TBB_FOUND OR NOT HAS_EXECUTION)
    # Test for openmp
    FIND_PACKAGE(OpenMP)
    IF (OpenMP_CXX_FOUND)
        MESSAGE ("Using OpenMP multi-threading support")
        SET(HAS_OPENMP True)
        LIST(APPEND LIBS "OpenMP::OpenMP_CXX")
    ELSE ()
        MESSAGE ("Neither C++-17 nor OpenMP multi-threading support found: providing only single-threaded algorithms")
    ENDIF ()
ENDIF ()

# BOOST
FIND_PACKAGE(Boost)
LIST(APPEND INCLUDE_DIRS ${Boost_INCLUDE_DIRS})

# SDK

IF (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    ADD_SUBDIRECTORY(sdk/backward-cpp)
    SET_PROPERTY(TARGET backward_object PROPERTY POSITION_INDEPENDENT_CODE 1)
    LIST(APPEND INTERFACES "Backward::Backward")
ELSE ()
    SET(BACWARD_ENABLE "")
ENDIF ()

SET(JSON_BuildTests OFF CACHE INTERNAL "")
SET(JSON_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/sdk/json/single_include/nlohmann)
ADD_SUBDIRECTORY(sdk/json)
LIST(APPEND INCLUDE_DIRS "${JSON_INCLUDE_DIR}")

SET(SPDLOG_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/sdk/spdlog/include)
ADD_SUBDIRECTORY(sdk/spdlog)
LIST(APPEND INCLUDE_DIRS "${SPDLOG_INCLUDE_DIR}")
LIST(APPEND LIBS spdlog::spdlog)

CMAKE_POLICY(SET CMP0077 NEW)
SET(NNG_STATIC_LIB ON CACHE INTERNAL "")
SET(NNG_TESTS OFF CACHE INTERNAL "")
SET(NNG_TOOLS OFF CACHE INTERNAL "")
SET(NNG_ENABLE_NNGCAT OFF CACHE INTERNAL "")
ADD_SUBDIRECTORY(sdk/nng)
LIST(APPEND INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/sdk/nng/include)
LIST(APPEND LIBS nng)

LIST(APPEND INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/sdk/nngpp/include)

IF (BUILD_PYTHON_BINDING OR ONLY_PYTHON)
    ADD_SUBDIRECTORY(sdk/pybind11)
    LIST(APPEND INCLUDE_DIRS "${PYBIND11_INCLUDE_DIR}")
    LIST(APPEND INCLUDE_DIRS "${PYTHON_INCLUDE_DIRS}")
ENDIF ()

IF (BUILD_TESTS AND NOT ONLY_PYTHON)
    SET(CATCH2_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/sdk/Catch2/single_include/catch2)
    ADD_SUBDIRECTORY(sdk/Catch2)
ENDIF()

# CONFIG FILE
CONFIGURE_FILE(config.h.in ${PROJECT_BINARY_DIR}/src/config.h)
LIST(APPEND INCLUDE_DIRS "${PROJECT_BINARY_DIR}/src")

# LIB
ADD_SUBDIRECTORY(src)

# TESTS
IF (BUILD_TESTS AND NOT ONLY_PYTHON)
    ENABLE_TESTING()  # leave it outside the 'tests' directory so ctest can find the CTestTestFile.cmake in the top build directory
    ADD_SUBDIRECTORY(tests)
ENDIF()
