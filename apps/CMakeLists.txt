IF(WIN32)
  SET(COMMON_LIBS getopt)
ENDIF()

IF(BUILD_LENSTOOL)
    FIND_PACKAGE(PNG REQUIRED)
    INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
    INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR})
    INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
    ADD_EXECUTABLE(lenstool lenstool/lenstool.cpp lenstool/image.cpp)
    TARGET_LINK_LIBRARIES(lenstool lensfun ${COMMON_LIBS} ${PNG_LIBRARY} ${ZLIB_LIBRARY})
    INSTALL(TARGETS lenstool DESTINATION ${LENSFUN_FULL_BINDIR})
ENDIF()


IF(INSTALL_HELPER_SCRIPTS)
  INSTALL(FILES g-lensfun-update-data lensfun-add-adapter lensfun-update-data
          PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ GROUP_EXECUTE GROUP_READ WORLD_EXECUTE WORLD_READ
          DESTINATION ${LENSFUN_FULL_BINDIR})
ENDIF(INSTALL_HELPER_SCRIPTS)
