# Copyright (C) Alexander Lamaison <alexander.lamaison@gmail.com>
# Copyright (C) Viktor Szakats
#
# Redistribution and use in source and binary forms,
# with or without modification, are permitted provided
# that the following conditions are met:
#
#   Redistributions of source code must retain the above
#   copyright notice, this list of conditions and the
#   following disclaimer.
#
#   Redistributions in binary form must reproduce the above
#   copyright notice, this list of conditions and the following
#   disclaimer in the documentation and/or other materials
#   provided with the distribution.
#
#   Neither the name of the copyright holder nor the names
#   of any other contributors may be used to endorse or
#   promote products derived from this software without
#   specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
# CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
# INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
# OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
# WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
# NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
# USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
# OF SUCH DAMAGE.
#
# SPDX-License-Identifier: BSD-3-Clause

set(_libssh2_soversion 1)
set(_libssh2_libversion 1.0.1)

if(CRYPTO_BACKEND)
  list(APPEND PRIVATE_COMPILE_DEFINITIONS ${CRYPTO_BACKEND_DEFINE})
  list(APPEND PRIVATE_INCLUDE_DIRECTORIES ${CRYPTO_BACKEND_INCLUDE_DIR})
  add_feature_info("Crypto backend" ON "${CRYPTO_BACKEND}")
else()
  message(FATAL_ERROR "No suitable cryptography backend found.")
endif()

## Options

unset(_libssh2_definitions)

option(CLEAR_MEMORY "Enable clearing of memory before being freed" ON)
if(NOT CLEAR_MEMORY)
  list(APPEND _libssh2_definitions "LIBSSH2_NO_CLEAR_MEMORY")
endif()

option(ENABLE_ZLIB_COMPRESSION "Use zlib for compression" OFF)
add_feature_info(Compression ENABLE_ZLIB_COMPRESSION
  "using zlib for compression")
if(ENABLE_ZLIB_COMPRESSION)
  find_package(ZLIB REQUIRED)

  list(APPEND libssh2_INCLUDE_DIRS ${ZLIB_INCLUDE_DIRS})
  list(APPEND LIBSSH2_LIBS ${ZLIB_LIBRARIES})
  list(APPEND LIBSSH2_PC_REQUIRES_PRIVATE "zlib")
  if(ZLIB_FOUND)
    list(APPEND _libssh2_definitions "LIBSSH2_HAVE_ZLIB")
  endif()
endif()

list(APPEND LIBSSH2_LIBS ${LIBSSH2_LIBS_SOCKET})

# to find generated header
list(APPEND libssh2_INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR})

if(MSVC)
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /Zi /Od")
  set(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS_DEBUG} /DEBUG")
endif()

## Sources

include(GNUInstallDirs)
transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
# Get 'CSOURCES' and 'HHEADERS' variables
include("${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
set(_sources ${CSOURCES} ${HHEADERS})

## Library definition

# Ensure that the static library and import library filenames are different,
# when building both static and shared library. On Windows, with certain
# toolchains (e.g. MSVC) these libraries get the same by default, overwriting
# each other. MinGW is not affected.
if(WIN32 AND BUILD_STATIC_LIBS AND BUILD_SHARED_LIBS AND
   NOT STATIC_LIB_SUFFIX AND NOT IMPORT_LIB_SUFFIX AND
   CMAKE_STATIC_LIBRARY_SUFFIX STREQUAL CMAKE_IMPORT_LIBRARY_SUFFIX)
  set(STATIC_LIB_SUFFIX "_static")
endif()

unset(_libssh2_export)

# we want it to be called libssh2 on all platforms
if(BUILD_STATIC_LIBS)
  list(APPEND _libssh2_export ${LIB_STATIC})
  add_library(${LIB_STATIC} STATIC ${_sources})
  add_library(${PROJECT_NAME}::${LIB_STATIC} ALIAS ${LIB_STATIC})
  target_compile_definitions(${LIB_STATIC} PRIVATE ${PRIVATE_COMPILE_DEFINITIONS} ${_libssh2_definitions})
  target_link_libraries(${LIB_STATIC} PRIVATE ${LIBSSH2_LIBS})
  set_target_properties(${LIB_STATIC} PROPERTIES
    PREFIX "" OUTPUT_NAME "libssh2" SOVERSION "${_libssh2_soversion}" VERSION "${_libssh2_libversion}"
    SUFFIX "${STATIC_LIB_SUFFIX}${CMAKE_STATIC_LIBRARY_SUFFIX}")

  target_include_directories(${LIB_STATIC}
    PRIVATE
      "${PROJECT_SOURCE_DIR}/include"
      ${libssh2_INCLUDE_DIRS}
      ${PRIVATE_INCLUDE_DIRECTORIES}
    PUBLIC
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
      "$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}>")
endif()
if(BUILD_SHARED_LIBS)
  list(APPEND _libssh2_export ${LIB_SHARED})
  add_library(${LIB_SHARED} SHARED ${_sources})
  add_library(${PROJECT_NAME}::${LIB_SHARED} ALIAS ${LIB_SHARED})
  if(WIN32)
    set_property(TARGET ${LIB_SHARED} APPEND PROPERTY SOURCES "libssh2.rc")
  endif()
  target_compile_definitions(${LIB_SHARED} PRIVATE ${PRIVATE_COMPILE_DEFINITIONS} ${_libssh2_definitions} ${LIB_SHARED_DEFINITIONS})
  target_compile_options(${LIB_SHARED} PRIVATE ${LIB_SHARED_C_FLAGS})
  target_link_libraries(${LIB_SHARED} PRIVATE ${LIBSSH2_LIBS})
  set_target_properties(${LIB_SHARED} PROPERTIES
    PREFIX "" OUTPUT_NAME "libssh2" SOVERSION "${_libssh2_soversion}" VERSION "${_libssh2_libversion}"
    IMPORT_PREFIX "" IMPORT_SUFFIX "${IMPORT_LIB_SUFFIX}${CMAKE_IMPORT_LIBRARY_SUFFIX}"
    POSITION_INDEPENDENT_CODE ON)

  target_include_directories(${LIB_SHARED}
    PRIVATE
      "${PROJECT_SOURCE_DIR}/include"
      ${libssh2_INCLUDE_DIRS}
      ${PRIVATE_INCLUDE_DIRECTORIES}
    PUBLIC
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
      "$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}>")
endif()

add_library(${PROJECT_NAME}::${LIB_NAME} ALIAS ${LIB_SELECTED})
add_library(${LIB_NAME} ALIAS ${LIB_SELECTED})
