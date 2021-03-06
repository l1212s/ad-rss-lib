# ----------------- BEGIN LICENSE BLOCK ---------------------------------
#
# Copyright (c) 2018-2019 Intel Corporation
#
# Redistribution and use in source and binary forms, with or without modification,
# are permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice,
#    this list of conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright notice,
#    this list of conditions and the following disclaimer in the documentation
#    and/or other materials provided with the distribution.
#
# 3. Neither the name of the copyright holder nor the names of its contributors
#    may be used to endorse or promote products derived from this software without
#    specific prior written permission.
#
#    THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
#    ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
#    WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED.
#    IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT,
#    INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
#    BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA,
#    OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
#    WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
#    ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
#    POSSIBILITY OF SUCH DAMAGE.
#
# ----------------- END LICENSE BLOCK -----------------------------------
##
cmake_minimum_required(VERSION 3.5)
set(BUILD_TESTING "ON" CACHE BOOL "Enable test compilation")
set(BUILD_DOC "OFF" CACHE BOOL "Enable compilation of doxygen documentation")

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
project(ad_rss_lib VERSION 1.0)

set(CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
# enable a bunch of warning
add_compile_options(-Wall -Wextra -pedantic -Wfloat-equal -Wshadow -Wswitch-default -Wenum-compare -Wformat -Wformat-security)
# use some warnings only for library code not for gtest
set(TARGET_COMPILE_OPTIONS -Wconversion -Wsign-conversion)
# warnings are treated as error
add_compile_options(-Werror)

# hardening
set(HARDENING_LD_FLAGS "-Wl,-z,now -Wl,-z,relro -pie")
add_compile_options(-fstack-protector-strong -fasynchronous-unwind-tables -fno-omit-frame-pointer -fPIE -fPIC -D_FORTIFY_SOURCE=2)

# activate code coverage
set(COVERAGE_FLAG "--coverage")

add_library(${PROJECT_NAME} SHARED
  src/ad_rss/physics/Acceleration.cpp
  src/ad_rss/physics/CoordinateSystemAxis.cpp
  src/ad_rss/physics/Distance.cpp
  src/ad_rss/physics/DistanceSquared.cpp
  src/ad_rss/physics/Duration.cpp
  src/ad_rss/physics/DurationSquared.cpp
  src/ad_rss/physics/ParametricValue.cpp
  src/ad_rss/physics/Speed.cpp
  src/ad_rss/physics/SpeedSquared.cpp
  src/ad_rss/situation/LateralRelativePosition.cpp
  src/ad_rss/situation/LongitudinalRelativePosition.cpp
  src/ad_rss/situation/SituationType.cpp
  src/ad_rss/state/LateralResponse.cpp
  src/ad_rss/state/LongitudinalResponse.cpp
  src/ad_rss/world/LaneDrivingDirection.cpp
  src/ad_rss/world/LaneSegmentType.cpp
  src/ad_rss/world/ObjectType.cpp
  src/core/RssCheck.cpp
  src/core/RssResponseResolving.cpp
  src/core/RssResponseTransformation.cpp
  src/core/RssSituationChecking.cpp
  src/core/RssSituationExtraction.cpp
  src/physics/Math.cpp
  src/situation/RSSFormulas.cpp
  src/situation/RssIntersectionChecker.cpp
  src/situation/RSSSituation.cpp
  src/world/RssSituationCoordinateSystemConversion.cpp
  src/world/RssObjectPositionExtractor.cpp
  )

set_target_properties(${PROJECT_NAME} PROPERTIES
  OUTPUT_NAME ad_rss
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR})

target_include_directories(${PROJECT_NAME}
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
  src
)

target_compile_options(${PROJECT_NAME} PRIVATE ${COVERAGE_FLAG} ${TARGET_COMPILE_OPTIONS})
set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS "${COVERAGE_FLAG} ${HARDENING_LD_FLAGS}")

################################################################################
# Install section
################################################################################
# common section
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ad_rss
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
  file(RELATIVE_PATH CMAKE_INSTALL_LIBDIR_RELATIVE "${CMAKE_INSTALL_PREFIX}" "${CMAKE_INSTALL_LIBDIR}")
else()
  set(CMAKE_INSTALL_LIBDIR_RELATIVE "${CMAKE_INSTALL_LIBDIR}")
endif()

install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}_EXPORT
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR_RELATIVE}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR_RELATIVE})

set(CMAKECONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR_RELATIVE}/cmake/${PROJECT_NAME}")

install(EXPORT ${PROJECT_NAME}_EXPORT FILE ${PROJECT_NAME}Targets.cmake
  DESTINATION ${CMAKECONFIG_INSTALL_DIR})

configure_package_config_file(${PROJECT_NAME}Config.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake INSTALL_DESTINATION ${CMAKECONFIG_INSTALL_DIR})

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  DESTINATION ${CMAKECONFIG_INSTALL_DIR}
  COMPONENT libs)

################################################################################
# Test section
################################################################################

if(BUILD_TESTING)
message(STATUS "Build Testing")
  enable_testing()
  if(NOT TARGET GTest::GTest)
    find_package(GTest QUIET)
    if (NOT GTEST_FOUND)
      #As Ubuntu never ships precompiled gtest libraries, build them on-the-fly from source
      #requires gtest-dev package to be intalled
      add_subdirectory(/usr/src/gtest gtest)
      add_library(GTest::GTest ALIAS gtest)
      add_library(GTest::Main ALIAS gtest_main)
    endif()
  endif()
 add_subdirectory(tests)
endif()


################################################################################
# Doxygen documentation
################################################################################
if(BUILD_DOC)
  find_package(Doxygen REQUIRED)
  if (DOXYGEN_FOUND)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/doc/apidoc/Doxyfile.in
                   ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
                   @ONLY)
    add_custom_target(apidoc
                      COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                      COMMENT "Build doxygen documentation")
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/apidoc)
  else()
    message(FATAL_ERROR, "Doxygen needs to be installed")
  endif()
endif()
