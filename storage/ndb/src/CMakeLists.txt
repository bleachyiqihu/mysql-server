# Copyright (c) 2008, 2019, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License, version 2.0,
# as published by the Free Software Foundation.
#
# This program is also distributed with certain software (including
# but not limited to OpenSSL) that is licensed under separate terms,
# as designated in a particular file or component or in included license
# documentation.  The authors of MySQL hereby grant you an additional
# permission to link the program and your derivative works with the
# separately licensed software that they have included with MySQL.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License, version 2.0, for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA

INCLUDE(libutils)

# Disable specific types of warnings for current directory
# if the compiler supports the flag
FOREACH(warning
    "unused-but-set-variable"
    "strict-aliasing"
    "unused-parameter"
    "cast-qual"
    # Downgrade -Werror to warning for "may be used uninitialized"
    "error=maybe-uninitialized"
    # Downgrade placement-new as part of Bug #24822203
    "error=placement-new"
    # Downgrade -Werror=deprecated-copy to warning for gcc 9
    "error=deprecated-copy"
    # Downgrade -Werror=stringop-truncation to warning for gcc 8/9
    "error=stringop-truncation"
    )
  MY_CHECK_CXX_COMPILER_WARNING("${warning}" HAS_WARN_FLAG)
  IF(HAS_WARN_FLAG)
    STRING_APPEND(CMAKE_CXX_FLAGS " ${HAS_WARN_FLAG}")
    STRING_APPEND(CMAKE_C_FLAGS " ${HAS_WARN_FLAG}")
  ENDIF()
ENDFOREACH()

##################
# Temporary C++ 8 workarounds:
# (not supported for C, so only set CXX_FLAGS)
FOREACH(warning
    # Workaround for bug#28387760, also failed other places then in NdbEvent*
    "class-memaccess"
    )
  MY_CHECK_CXX_COMPILER_WARNING("${warning}" HAS_WARN_FLAG)
  IF(HAS_WARN_FLAG)
    STRING_APPEND(CMAKE_CXX_FLAGS " ${HAS_WARN_FLAG}")
  ENDIF()
ENDFOREACH()

ADD_SUBDIRECTORY(common)
ADD_SUBDIRECTORY(mgmapi)
ADD_SUBDIRECTORY(ndbapi)

IF(NOT WITHOUT_SERVER)
  ADD_SUBDIRECTORY(kernel)
ENDIF()
ADD_SUBDIRECTORY(mgmclient)
ADD_SUBDIRECTORY(mgmsrv)

IF(BUILD_IS_SINGLE_CONFIG)
  SET(JAVA_SUBDIR)
ELSE()
  set(JAVA_SUBDIR
  $<$<CONFIG:Debug>:Debug>$<$<CONFIG:RelWithDebInfo>:RelWithDebInfo>)
ENDIF()

IF(WIN32)
  SET(JAVA_SEPERATOR ";")
ELSE()
  set(JAVA_SEPERATOR :)
ENDIF()

SET(NDBJTIE_LIB)
IF(HAVE_JDK)
  ADD_SUBDIRECTORY(ndbjtie)
  SET(NDBJTIE_LIB ndbjtie)
  ADD_DEFINITIONS(-DNDB_WITH_NDBJTIE)
ENDIF()

#
# Build static ndbclient library
#
SET(NDBCLIENT_LIBS
  mysys strings
  ${SSL_LIBRARIES} ${ZLIB_LIBRARY}
  ndbapi
  ndbtransport
  ndbtrace
  ndbsignaldata
  ndbmgmapi
  ndbmgmcommon
  ndblogger
  ndbportlib
  ndbgeneral
  ndbconf)

SET(LIBS_TO_MERGE
  mysys
  mytime
  ndbapi
  ndbconf
  ndbgeneral
  ndblogger
  ndbmgmapi
  ndbmgmcommon
  ndbportlib
  ndbsignaldata
  ndbtrace
  ndbtransport
  strings
  )

IF(WITH_ZLIB STREQUAL "bundled")
  LIST(APPEND LIBS_TO_MERGE ${ZLIB_LIBRARY})
ENDIF()


MESSAGE(STATUS "LIBS_TO_MERGE ${LIBS_TO_MERGE}")
MESSAGE(STATUS "NDBCLIENT_LIBS ${NDBCLIENT_LIBS}")

MERGE_CONVENIENCE_LIBRARIES(
  ndbclient_static ${LIBS_TO_MERGE} COMPONENT Development)
TARGET_LINK_LIBRARIES(ndbclient_static PRIVATE ${SSL_LIBRARIES})

# Build test program to check linking against ndclient_static
ADD_EXECUTABLE(ndbclient_static_link_test ndbclient_link_test.cpp)
TARGET_LINK_LIBRARIES(ndbclient_static_link_test ndbclient_static)

#
# Build shared ndbclient library
#
SET(NDBCLIENT_SO_LIBS ${LIBS_TO_MERGE} ${NDBJTIE_LIB})
SET(NDB_SHARED_LIB_VERSION "6.1.0")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libndbclient.ver.in
               ${CMAKE_CURRENT_BINARY_DIR}/libndbclient.ver)

ADD_LIBRARY(ndbclient_so SHARED ndbclient_exports.cpp)

# Collect all dynamic libraries in the same directory
SET_TARGET_PROPERTIES(ndbclient_so PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/library_output_directory)
IF(LINUX_INSTALL_RPATH_ORIGIN)
  SET_PROPERTY(TARGET ndbclient_so PROPERTY INSTALL_RPATH "\$ORIGIN/")
ENDIF()
IF(WIN32_CLANG AND WITH_ASAN)
  TARGET_LINK_LIBRARIES(ndbclient_so
    "${ASAN_LIB_DIR}/clang_rt.asan_dll_thunk-x86_64.lib")
ENDIF()

IF(WIN32)
  # This must be a cmake bug on windows ...
  # Anyways, with this the .dll ends up in the desired directory.
  SET_TARGET_PROPERTIES(ndbclient_so PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/library_output_directory)
ENDIF()

TARGET_LINK_LIBRARIES(ndbclient_so ${NDBCLIENT_SO_LIBS})

IF(APPLE AND HAVE_CRYPTO_DYLIB AND HAVE_OPENSSL_DYLIB)
  ADD_CUSTOM_COMMAND(TARGET ndbclient_so POST_BUILD
    COMMAND install_name_tool -change
    "${CRYPTO_VERSION}" "@loader_path/${CRYPTO_VERSION}"
    $<TARGET_SONAME_FILE:ndbclient_so>
    COMMAND install_name_tool -change
    "${OPENSSL_VERSION}" "@loader_path/${OPENSSL_VERSION}"
    $<TARGET_SONAME_FILE:ndbclient_so>
    )
  # All executables have dependencies:  "@loader_path/../lib/xxx.dylib
  # Create a symlink so that this works for Xcode also.
  IF(NOT BUILD_IS_SINGLE_CONFIG)
    ADD_CUSTOM_COMMAND(TARGET ndbclient_so POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E create_symlink
      $<TARGET_SONAME_FILE_DIR:ndbclient_so> lib
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/runtime_output_directory
      )
  ENDIF()
ENDIF()


IF(APPLE)
  SET_TARGET_PROPERTIES(ndbclient_so PROPERTIES
    MACOSX_RPATH ON
  )
ENDIF()

IF(MSVC)
# All "convenience" libraries are now well-known, we *could* do this:
#  SET(_PLATFORM  x64)
#
#  ADD_CUSTOM_COMMAND(TARGET ndbclient_so PRE_LINK
#    COMMAND cscript ARGS //nologo ${PROJECT_SOURCE_DIR}/win/create_def_file.js
#                         ${_PLATFORM}
#                         "$<TARGET_FILE:lib1>"
#                         "$<TARGET_FILE:lib2>"
#                         ...
#                         > ndbclient_exports.def
#    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
#	
#  # Modify link flags to use the generated module definition file
#  GET_TARGET_PROPERTY(ndbclient_so_link_flags ndbclient_so LINK_FLAGS)
#  SET_TARGET_PROPERTIES(ndbclient_so PROPERTIES LINK_FLAGS
#                        "${ndbclient_link_flags} /DEF:ndbclient_exports.def")
ELSE()
  # Turn off cmake's transitive link behaviour so that the shared
  # ndbclient library contains everything except functions from
  # system libs
  SET(NDBCLIENT_SYSTEM_LIBS)
  FOREACH(lib ${NDBCLIENT_SO_LIBS})
    SET(deps ${${lib}_LIB_DEPENDS})
    FOREACH(dep_lib ${deps})
    SET(ignore_dep_lib)

      # The list of dependent libs contains keywords used for debug vs optimized
      # builds, ignore them 
      IF(dep_lib MATCHES "general" OR
         dep_lib MATCHES "debug" OR
         dep_lib MATCHES "optimized")
        # MESSAGE(STATUS "Ignoring keyword ${dep_lib}")
        SET(ignore_dep_lib 1)
      ENDIF()

      # Ignore libs where location is known(this means it's built locally)
      LIST(FIND KNOWN_CONVENIENCE_LIBRARIES ${dep_lib} FOUNDIT)
      IF(FOUNDIT GREATER -1)
        # MESSAGE(STATUS "Ignoring ${dep_lib} since location is known")
        SET(ignore_dep_lib 1)
      ENDIF()

      IF(NOT ignore_dep_lib)
        LIST(APPEND NDBCLIENT_SYSTEM_LIBS ${dep_lib})
      ENDIF()

    ENDFOREACH()
  ENDFOREACH()
  IF(NDBCLIENT_SYSTEM_LIBS)
    LIST(REMOVE_DUPLICATES NDBCLIENT_SYSTEM_LIBS)
    MESSAGE(STATUS "System libs used by ndbclient_so: ${NDBCLIENT_SYSTEM_LIBS}")
    TARGET_LINK_LIBRARIES(ndbclient_so LINK_INTERFACE_LIBRARIES
      ${NDBCLIENT_SYSTEM_LIBS})
  ELSE()
    MESSAGE(STATUS "System libs used by ndbclient_so: [none]")
  ENDIF()

  # Prepend any special linker flags(like -m64) for shared library
  GET_TARGET_PROPERTY(ndbclient_so_link_flags ndbclient_so LINK_FLAGS)
  IF(NOT ndbclient_so_link_flags)
    # Avoid LINK_FLAGS-NOTFOUND
    SET(ndbclient_so_link_flags)
  ENDIF()
  SET(ndbclient_so_link_flags
    "${CMAKE_SHARED_LIBRARY_C_FLAGS} ${ndbclient_so_link_flags}")
  IF(LINK_FLAG_NO_UNDEFINED)
    SET(ndbclient_so_link_flags
      "${ndbclient_so_link_flags} ${LINK_FLAG_NO_UNDEFINED}")
    SET(ndbclient_so_link_flags
      "${ndbclient_so_link_flags} -Wl,--version-script=${CMAKE_CURRENT_BINARY_DIR}/libndbclient.ver")
  ENDIF()
  IF(SOLARIS)
    SET(ndbclient_so_link_flags
      "${ndbclient_so_link_flags} ${CMAKE_CXX_LINK_FLAGS}")
  ENDIF()
  SET_TARGET_PROPERTIES(ndbclient_so
    PROPERTIES LINK_FLAGS "${ndbclient_so_link_flags}")
ENDIF()


SET_TARGET_PROPERTIES(ndbclient_so PROPERTIES
  OUTPUT_NAME "ndbclient"
  SOVERSION ${NDB_SHARED_LIB_VERSION})
MYSQL_INSTALL_TARGETS(ndbclient_so
  DESTINATION "${INSTALL_LIBDIR}"
  COMPONENT Development)
# Build test program to check linking against ndclient_so
ADD_EXECUTABLE(ndbclient_shared_link_test ndbclient_link_test.cpp)
TARGET_LINK_LIBRARIES(ndbclient_shared_link_test ndbclient_so)
