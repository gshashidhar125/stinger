include(ExternalProject)

###############################################
# Add library dependencies here as dep_<folder>
SET(libdep_default string int_hm_seq stinger_core stinger_utils)

IF(APPLE)
  SET(libdep_server ${CMAKE_BINARY_DIR}/lib/libprotobuf.dylib stinger_net)
  SET(libdep_server_alg_tmp ${CMAKE_BINARY_DIR}/lib/libprotobuf.dylib stinger_net)
ELSEIF(CYGWIN OR WIN32)
  SET(libdep_server ${CMAKE_BINARY_DIR}/lib/libprotobuf.dll.a stinger_net)
  SET(libdep_server_alg_tmp ${CMAKE_BINARY_DIR}/lib/libprotobuf.dll.a stinger_net)
ELSE()
  SET(libdep_server ${CMAKE_BINARY_DIR}/lib/libprotobuf.so stinger_net)
  SET(libdep_server_alg_tmp ${CMAKE_BINARY_DIR}/lib/libprotobuf.so stinger_net)
ENDIF()

SET(tgtdep_default string int_hm_seq stinger_core stinger_utils rt)
SET(tgtdep_server rapidjson protobuf stinger_net)
SET(tgtdep_server_alg_tmp rapidjson protobuf stinger_net)

#################################################################
# Iterate over each library and build appropriately based on type
FILE(GLOB subs RELATIVE ${CMAKE_CURRENT_LIST_DIR} ${CMAKE_CURRENT_LIST_DIR}/*)
FOREACH(subdir ${subs})
  IF(IS_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/${subdir})
    IF(EXISTS ${CMAKE_CURRENT_LIST_DIR}/${subdir}/CMakeLists.txt)
      MESSAGE("BIN: CMAKE SUB FOUND ${subdir}")
      ADD_SUBDIRECTORY(${subdir})
    ELSEIF(EXISTS ${CMAKE_CURRENT_LIST_DIR}/${subdir}/configure)
      MESSAGE("BIN: AUTOTOOLS SUB FOUND ${subdir}")
      ExternalProject_Add(
	"${subdir}"
	DOWNLOAD_COMMAND ""
	SOURCE_DIR "${subdir}"
	CONFIGURE_COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/${subdir}/configure --prefix=${CMAKE_BINARY_DIR}
	BUILD_COMMAND ${MAKE}
      )
    ELSE()
      MESSAGE("BIN: DEFAULT SUBDIR FOUND ${subdir}")

      # Glob and compile and protobuf sources
      SET(realprotosrc "")
      SET(protohead "")
      IF(EXISTS ${CMAKE_CURRENT_LIST_DIR}/${subdir}/proto)
	IF(IS_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/${subdir}/proto)
	  FILE(GLOB protos ABSOLUTE ${CMAKE_CURRENT_LIST_DIR}/${subdir}/proto/*.proto)
	  CUST_PROTOBUF_GENERATE_CPP(protosrc protohead ${protos})
	  SET(realprotosrc)
	  FOREACH(protoout ${protosrc})
	    GET_FILENAME_COMPONENT(protofile ${protoout} NAME)
	    LIST(APPEND realprotosrc ${CMAKE_CURRENT_BINARY_DIR}/${subdir}/proto/${protofile})
	  ENDFOREACH()
	  SET_SOURCE_FILES_PROPERTIES(${realprotosrc} PROPERTIES GENERATED TRUE)
	  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
	  INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR}/${subdir})
	ENDIF()
      ENDIF()

      # Glob the c++ source
      FILE(GLOB srcs RELATIVE ${CMAKE_CURRENT_LIST_DIR} ${CMAKE_CURRENT_LIST_DIR}/${subdir}/src/*)
      INCLUDE_DIRECTORIES(${subdir}/inc)
      ADD_EXECUTABLE(${subdir} ${srcs} ${realprotosrc} ${protohead})

      ADD_DEPENDENCIES(${subdir} ${tgtdep_default})
      TARGET_LINK_LIBRARIES(${subdir} ${libdep_default})

      FOREACH(dep_target ${tgtdep_${subdir}})
	ADD_DEPENDENCIES(${subdir} ${dep_target})
      ENDFOREACH()
      TARGET_LINK_LIBRARIES(${subdir} ${libdep_${subdir}})

    ENDIF()
  ENDIF()
ENDFOREACH()
