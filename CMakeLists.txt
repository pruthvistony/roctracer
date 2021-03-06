################################################################################
##
## Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.
##
## MIT LICENSE:
## Permission is hereby granted, free of charge, to any person obtaining a copy of
## this software and associated documentation files (the "Software"), to deal in
## the Software without restriction, including without limitation the rights to
## use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
## of the Software, and to permit persons to whom the Software is furnished to do
## so, subject to the following conditions:
##
## The above copyright notice and this permission notice shall be included in all
## copies or substantial portions of the Software.
##
## THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
## IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
## FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
## AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
## LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
## OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
## SOFTWARE.
##
################################################################################

cmake_minimum_required ( VERSION 3.5.0 )

## Verbose output.
set ( CMAKE_VERBOSE_MAKEFILE TRUE CACHE BOOL "Verbose Output" FORCE )

## Set module name and project name.
set ( ROCTRACER_NAME "roctracer" )
set ( ROCTRACER_TARGET "${ROCTRACER_NAME}64" )
set ( ROCTRACER_LIBRARY "lib${ROCTRACER_TARGET}" )
project ( ${ROCTRACER_TARGET} )

## Adding default path cmake modules
list ( APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules" )
## Include common cmake modules
include ( utils )
## Set build environment
include ( env )

## Setup the package version.
get_version ( "1.0.0" )
message ( "-- LIB-VERSION: ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}" )

set ( BUILD_VERSION_MAJOR ${VERSION_MAJOR} )
set ( BUILD_VERSION_MINOR ${VERSION_MINOR} )
set ( BUILD_VERSION_PATCH ${VERSION_PATCH} )
set ( LIB_VERSION_STRING "${BUILD_VERSION_MAJOR}.${BUILD_VERSION_MINOR}.${BUILD_VERSION_PATCH}" )
if ( DEFINED VERSION_BUILD AND NOT ${VERSION_BUILD} STREQUAL "" )
  message ( "VERSION BUILD DEFINED ${VERSION_BUILD}" )
  set ( BUILD_VERSION_PATCH "${BUILD_VERSION_PATCH}-${VERSION_BUILD}" )
endif ()
set ( BUILD_VERSION_STRING "${BUILD_VERSION_MAJOR}.${BUILD_VERSION_MINOR}.${BUILD_VERSION_PATCH}" )

## Set target and root/lib/test directory
set ( TARGET_NAME "${ROCTRACER_TARGET}" )
set ( ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}" )
set ( LIB_DIR "${ROOT_DIR}/src" )
set ( TEST_DIR "${ROOT_DIR}/test" )

## Build library
include ( ${LIB_DIR}/CMakeLists.txt )

## Set the VERSION and SOVERSION values
set_property ( TARGET ${TARGET_NAME} PROPERTY VERSION "${LIB_VERSION_STRING}" )
set_property ( TARGET ${TARGET_NAME} PROPERTY SOVERSION "${BUILD_VERSION_MAJOR}" )

# If the library is a release, strip the target library
if ( "${CMAKE_BUILD_TYPE}" STREQUAL release )
  add_custom_command ( TARGET ${ROCTRACER_TARGET} POST_BUILD COMMAND ${CMAKE_STRIP} *.so )
endif ()

## Build tests
add_subdirectory ( ${TEST_DIR} ${PROJECT_BINARY_DIR}/test )

## Install and packaging
set ( CMAKE_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX}/${ROCTRACER_NAME} )
message ( "---------Install-Dir: ${CMAKE_INSTALL_PREFIX}" )

add_custom_target ( inc-link ALL WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                    COMMAND ${CMAKE_COMMAND} -E create_symlink ../${ROCTRACER_NAME}/include inc-link )
add_custom_target ( so-link ALL WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                    COMMAND ${CMAKE_COMMAND} -E create_symlink ../${ROCTRACER_NAME}/lib/${ROCTRACER_LIBRARY}.so so-link )

## Install information
install ( TARGETS ${ROCTRACER_TARGET} LIBRARY DESTINATION lib )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctracer.h DESTINATION include )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctracer_hip.h DESTINATION include )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctracer_hcc.h DESTINATION include )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctracer_ext.h DESTINATION include )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/ext/prof_protocol.h DESTINATION include/ext )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/ext/hsa_rt_utils.hpp DESTINATION include/ext )
install ( FILES ${PROJECT_BINARY_DIR}/inc-link DESTINATION ../include RENAME ${ROCTRACER_NAME} )
install ( FILES ${PROJECT_BINARY_DIR}/so-link DESTINATION ../lib RENAME ${ROCTRACER_LIBRARY}.so )
install ( FILES ${PROJECT_BINARY_DIR}/test/libtracer_tool.so DESTINATION tool )

## rocTX
set ( ROCTX_TARGET "roctx64" )
set ( ROCTX_LIBRARY "lib${ROCTX_TARGET}" )

add_custom_target ( so-roctx-link ALL WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                    COMMAND ${CMAKE_COMMAND} -E create_symlink ../${ROCTRACER_NAME}/lib/${ROCTX_LIBRARY}.so so-roctx-link )

install ( TARGETS "roctx64" LIBRARY DESTINATION lib )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctx.h DESTINATION include )
install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/inc/roctracer_roctx.h DESTINATION include )
install ( FILES ${PROJECT_BINARY_DIR}/so-roctx-link DESTINATION ../lib RENAME ${ROCTX_LIBRARY}.so )

## KFD wrapper
if ( DEFINED KFD_WRAPPER )
  install ( TARGETS "kfdwrapper64" LIBRARY DESTINATION lib )
endif ()

## Packaging directives
set ( CPACK_GENERATOR "DEB" "RPM" "TGZ" )
set ( CPACK_PACKAGE_NAME "${ROCTRACER_NAME}-dev" )
set ( CPACK_PACKAGE_VENDOR "AMD" )
set ( CPACK_PACKAGE_VERSION_MAJOR ${BUILD_VERSION_MAJOR} )
set ( CPACK_PACKAGE_VERSION_MINOR ${BUILD_VERSION_MINOR} )
set ( CPACK_PACKAGE_VERSION_PATCH ${BUILD_VERSION_PATCH} )
set ( CPACK_PACKAGE_CONTACT "Advanced Micro Devices Inc." )
set ( CPACK_PACKAGE_DESCRIPTION_SUMMARY "AMD ROCTRACER library" )
set ( CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE" )

## Debian package specific variables
set ( CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/DEBIAN/postinst;${CMAKE_CURRENT_SOURCE_DIR}/DEBIAN/prerm" )

## RPM package specific variables
set ( CPACK_RPM_PRE_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/RPM/rpm_post" )
set ( CPACK_RPM_POST_UNINSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/RPM/rpm_postun" )

include ( CPack )
