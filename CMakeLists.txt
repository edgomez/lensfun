# require at least cmake 2.8
CMAKE_MINIMUM_REQUIRED(VERSION 2.8 FATAL_ERROR )

# path for helper modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/build/CMakeModules)
GET_FILENAME_COMPONENT(SOURCE_BASE_DIR ${CMAKE_SOURCE_DIR} PATH CACHE)

#  global setup
project(lensfun)

# version
SET(VERSION_MAJOR 0)
SET(VERSION_MINOR 2)
SET(VERSION_MICRO 5)
SET(VERSION_BUGFIX 1)

# check if some include are available
INCLUDE(CheckIncludeFiles)
CHECK_INCLUDE_FILES(regex.h HAVE_REGEX_H)
CHECK_INCLUDE_FILES(endian.h HAVE_ENDIAN_H)

# set include directories
INCLUDE_DIRECTORIES(${CMAKE_BINARY_DIR})
INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/include/lensfun)
INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/include/auxfun)
IF(NOT HAVE_REGEX_H)
  INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/include/regex)
  INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/libs/regex)
ENDIF()

# options controlling the build process
OPTION(BUILD_STATIC "Build static library" OFF)
OPTION(BUILD_TESTS "Build test programs" ON)
OPTION(BUILD_AUXFUN "Build auxfun library (requires libpng)" OFF)
OPTION(BUILD_FOR_SSE "Build with support for SSE" OFF)
OPTION(BUILD_FOR_SSE2 "Build with support for SSE2" OFF)
OPTION(BUILD_DOC "Build documentation with doxygen" OFF)

IF(BUILD_FOR_SSE)
  SET(VECTORIZATION_SSE 1)
  IF(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse")
  ENDIF()
ENDIF()
IF(BUILD_FOR_SSE2)
  SET(VECTORIZATION_SSE2 1)
  IF(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
  ENDIF()
ENDIF()

IF(WIN32)
  INCLUDE_DIRECTORIES(${CMAKE_SOURCE_DIR}/libs/getopt)
  # base path for searching for glib on windows
  IF(NOT GLIB2_BASE_DIR)
    SET(GLIB2_BASE_DIR "${SOURCE_BASE_DIR}/glib-2.28.1" CACHE STRING "Base path of glib2 dir." FORCE)
  ENDIF()
ENDIF()

# find dependencies
FIND_PACKAGE(GLIB2 REQUIRED)
INCLUDE_DIRECTORIES(${GLIB2_INCLUDE_DIRS})

IF(BUILD_AUXFUN)
  FIND_PACKAGE(PNG REQUIRED)
  INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
  INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR})
  INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
ENDIF()

IF(BUILD_STATIC)
  ADD_DEFINITIONS(-DCONF_LENSFUN_STATIC)
ENDIF()
 
# define destination dirs (relative to prefix)
SET(BINDIR bin)
SET(LIBDIR lib)
SET(INCLUDEDIR include)
SET(DATADIR share/lensfun)
SET(DOCDIR share/doc/lensfun-${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO}.${VERSION_BUGFIX})

IF(WIN32)
  SET(PLATFORM_WINDOWS 1)
  # install into place in build-dir
  SET(CMAKE_INSTALL_PREFIX "${SOURCE_BASE_DIR}/lensfun" CACHE PATH "Install prefix for lensfun" FORCE)

  IF (MSVC)
    IF(BUILD_STATIC)
      # compile everything for the static C runtime
      FOREACH(TYPE C CXX)
        # makefiles use blank configuration
        FOREACH(CFG "_DEBUG" "_MINSIZEREL" "_RELEASE" "_RELWITHDEBINFO")
          SET(NEW_FLAGS "${CMAKE_${TYPE}_FLAGS${CFG}}")
          # fix up static libc flags
          STRING(REPLACE "/MD" "/MT" NEW_FLAGS "${NEW_FLAGS}")
          # *FORCE* to override whats already placed into the cache
          SET(CMAKE_${TYPE}_FLAGS${CFG} "${NEW_FLAGS}" CACHE STRING
            "CMAKE_${TYPE}_FLAGS${CFG} (overwritten to ensure static build)" FORCE)
        ENDFOREACH(CFG)
      ENDFOREACH(TYPE)
      # link only with the static C runtime !! THIS IS A MUST !!
      SET(NEW_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
      # ensure that the flags are not duplicaated on subsequent runs
      STRING(REPLACE " /NODEFAULTLIB:\"MSVCRT.lib\" /NODEFAULTLIB:\"MSVCRTd.lib\"" "" NEW_FLAGS "${NEW_FLAGS}")
      # note that flag names (NODEFAULTLIB) etc.) are case sensitive
      SET(CMAKE_EXE_LINKER_FLAGS
          "${NEW_FLAGS} /NODEFAULTLIB:\"MSVCRT.lib\" /NODEFAULTLIB:\"MSVCRTd.lib\""
          CACHE STRING "for MSVC" FORCE
      )
    ENDIF()
  ENDIF(MSVC)
ENDIF(WIN32)

# create config.h file
CONFIGURE_FILE(include/lensfun/config.h.in.cmake ${CMAKE_BINARY_DIR}/config.h)
# create lensfun.h file
CONFIGURE_FILE(include/lensfun/lensfun.h.in ${CMAKE_BINARY_DIR}/lensfun.h @ONLY)

# include libs
ADD_SUBDIRECTORY(libs)

# include example programs
IF(BUILD_TESTS)
ADD_SUBDIRECTORY(tests)
ENDIF()

# install include files
INSTALL(FILES ${CMAKE_BINARY_DIR}/lensfun.h DESTINATION ${INCLUDEDIR}/lensfun)
IF(NOT HAVE_REGEX_H)
  INSTALL(FILES include/regex/regex.h DESTINATION ${INCLUDEDIR}/regex)
ENDIF()
IF(BUILD_AUXFUN)
  INSTALL(FILES include/auxfun/image.h include/auxfun/rgbpixel.h DESTINATION ${INCLUDEDIR}/auxfun)
ENDIF()

# install db files
FILE(GLOB DATABASE_FILES data/db/*.xml)
INSTALL(FILES ${DATABASE_FILES} DESTINATION ${DATADIR})

# build documentation
IF(BUILD_DOC)
  ADD_SUBDIRECTORY(docs)
ENDIF()

IF(WIN32 AND NOT BUILD_STATIC)
FIND_FILE(GLIB2_DLL 
          NAMES glib-2.dll glib-2-vs9.dll
          PATHS "${GLIB2_BASE_DIR}/bin"
          NO_SYSTEM_ENVIRONMENT_PATH)
INSTALL(FILES ${GLIB2_DLL} DESTINATION ${BINDIR})
ENDIF()

# print some information
MESSAGE(STATUS "Building lensfun ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO}.${VERSION_BUGFIX}")
MESSAGE(STATUS "-------------------------------------------------------------------------------------")
IF(BUILD_STATIC)
  MESSAGE(STATUS "Build as static library")
ELSE()
  MESSAGE(STATUS "Build as dynamic/shared library")
ENDIF()
MESSAGE(STATUS "Build doc: ${BUILD_DOC}")
MESSAGE(STATUS "Build test programs ${BUILD_TESTS}")
IF(BUILD_AUXFUN)
  MESSAGE(STATUS "Building auxfun library (with Image class)")
ENDIF()
MESSAGE(STATUS "Build with support for SSE: ${BUILD_FOR_SSE}")
MESSAGE(STATUS "Build with support for SSE2: ${BUILD_FOR_SSE2}")
MESSAGE(STATUS "\nUsing: ")
MESSAGE(STATUS "GLIB2 include dir: ${GLIB2_INCLUDE_DIRS}")
MESSAGE(STATUS "GLIB2 library: ${GLIB2_LIBRARIES}")
IF(BUILD_AUXFUN)
MESSAGE(STATUS "libpng include dir: ${PNG_INCLUDE_DIR}")
MESSAGE(STATUS "libpng lib: ${PNG_LIBRARY}")
MESSAGE(STATUS "zlib include dir: ${ZLIB_INCLUDE_DIR}")
MESSAGE(STATUS "zlib lib: ${ZLIB_LIBRARY}")
ENDIF()

# Package
# This needs some testing, in particular for right package naming for different
# package managers and distributions. Debian works fine.
SET(CPACK_PACKAGE_NAME "lensfun")
SET(CPACK_PACKAGE_VERSION_MAJOR "${VERSION_MAJOR}")
SET(CPACK_PACKAGE_VERSION_MINOR "${VERSION_MINOR}")
SET(CPACK_PACKAGE_VERSION_PATCH "${VERSION_MICRO}.${VERSION_BUGFIX}")
SET(CPACK_PACKAGE_INSTALL_DIRECTORY "CMake ${V_MAJOR}.${V_MINOR}")
SET(CPACK_SOURCE_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO}.${VERSION_BUGFIX}")
SET(CPACK_SOURCE_GENERATOR "TGZ;TBZ2")
SET(CPACK_PACKAGE_CONTACT "zap@homelink.ru")
SET(CPACK_SET_DESTDIR "ON")
IF(BUILD_AUXFUN)
SET(CPACK_DEBIAN_PACKAGE_DEPENDS "libglib2.0-0 (>=2.0.0), libpng (>=1.0), zlib (>=1.0)")
ELSE()
SET(CPACK_DEBIAN_PACKAGE_DEPENDS "libglib2.0-0 (>=2.0.0)")
ENDIF()

IF(CPACK_BINARY_DEB)
    # workaround to set correct debian package name
    SET(CPACK_DEBIAN_PACKAGE_NAME "liblensfun${VERSION_MAJOR}")
    string(TOLOWER "${CPACK_DEBIAN_PACKAGE_NAME}" CPACK_PACKAGE_NAME_LOWERCASE)
    find_program(DPKG_PROGRAM dpkg DOC "dpkg program of Debian-based systems")
    if(DPKG_PROGRAM)
        execute_process(
            COMMAND ${DPKG_PROGRAM} --print-architecture
            OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME_LOWERCASE}_${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO}.${VERSION_BUGFIX}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}")
    else(DPKG_PROGRAM)
        set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME_LOWERCASE}_${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO}.${VERSION_BUGFIX}_${CMAKE_SYSTEM_NAME}")
    endif(DPKG_PROGRAM)
ENDIF(CPACK_BINARY_DEB)

INCLUDE(CPack)