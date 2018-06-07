cmake_minimum_required(VERSION 2.8)

cmake_policy(SET CMP0048 NEW)

PROJECT(zookeeper
		VERSION 1.0.0
		)
#set(zookeeper_MAJOR_VERSION 1)
#set(zookeeper_MINOR_VERSION 0)
#set(zookeeper_PATCH_VERSION 0)

set(PROJECT_INCLUDE_INSTALL_DIR include/${PROJECT_NAME} CACHE PATH "Install directory for headers")
set(PROJECT_LIBRARY_INSTALL_DIR lib CACHE PATH "Install directory for library")
set(PROJECT_RUNTIME_INSTALL_DIR bin CACHE PATH "Install directory for binaries/dlls")
set(PROJECT_CONFIG_INSTALL_DIR  share/${PROJECT_NAME} CACHE PATH "Install directory for cmake config files")

set(hdrs
   "include/config.h"
   "include/proto.h"
   "include/recordio.h"
   "include/winconfig.h"
   "include/winstdint.h"
   "include/zookeeper.h"
   "include/zookeeper.jute.h"
   "include/zookeeper_log.h"
   "include/zookeeper_version.h"
)

set(src 
"src/hashtable/hashtable_itr.c"
"src/hashtable/hashtable.c"
"src/mt_adaptor.c"
"src/recordio.c"
"src/zk_hashtable.c"
"src/zk_log.c"
"src/zookeeper.c"
)

set(src_generated "generated/zookeeper.jute.c")

ADD_DEFINITIONS(-DTHREADED)
if (NOT WIN32)
	ADD_DEFINITIONS(-fPIC)
endif()

INCLUDE_DIRECTORIES("include")

macro(ms_link_static_runtime FLAG_VAR)
    if(MSVC)
        if(${FLAG_VAR} MATCHES "/MD")
            string(REPLACE "/MD"  "/MT" "${FLAG_VAR}" "${${FLAG_VAR}}")
            #Save persistently
            set(${FLAG_VAR} ${${FLAG_VAR}} CACHE STRING "" FORCE)
        endif()
    endif()
endmacro(ms_link_static_runtime)

macro(ms_replace_compiler_flags REPLACE_OPTION)
    set(SUFFIXES "")
    if((NOT DEFINED CMAKE_CONFIGURATION_TYPES) OR (CMAKE_CONFIGURATION_TYPES STREQUAL ""))
        #set(SUFFIXES "_DEBUG" "_RELEASE" "_MINSIZEREL" "_RELWITHDEBINFO")
        if((DEFINED CMAKE_BUILD_TYPE) AND (NOT (CMAKE_BUILD_TYPE STREQUAL "")))
            string(TOUPPER ${CMAKE_BUILD_TYPE} SUFFIXES)
            set(SUFFIXES "_${SUFFIXES}")
        endif()
    else()
        foreach(SUFFIX ${CMAKE_CONFIGURATION_TYPES})
            string(TOUPPER ${SUFFIX} SUFFIX)
            set(SUFFIXES ${SUFFIXES} "_${SUFFIX}")
        endforeach()
    endif()

    foreach(SUFFIX "" ${SUFFIXES})
        foreach(LANG C CXX)
            set(FLAG_VAR "CMAKE_${LANG}_FLAGS${SUFFIX}")
            if(${REPLACE_OPTION} STREQUAL "STATIC_LINK")
                ms_link_static_runtime(${FLAG_VAR})
            endif()
        endforeach()
        #message(STATUS ${FLAG_VAR} ":" ${${FLAG_VAR}})
    endforeach()
endmacro(ms_replace_compiler_flags)

if(WIN32)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	add_definitions(-DWIN32_LEAN_AND_MEAN)        
	add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
	add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS=1)
	add_definitions(-D_WIN32_WINNT=0x0600)
	add_definitions(-D_UNICODE)
	add_definitions(-DUNICODE)
	add_definitions(-DDLL_EXPORT)
	#add_compile_options(-MP)
	#ms_replace_compiler_flags("STATIC_LINK")
	set(src ${src} "src/winport.c")
	ADD_LIBRARY(zookeeper SHARED ${src} ${src_generated})
	target_link_libraries(zookeeper PRIVATE "ws2_32.lib")	
else()
	ADD_LIBRARY(zookeeper STATIC ${src} ${src_generated})
endif()
		
install(TARGETS ${PROJECT_NAME} 
		DESTINATION lib
        EXPORT ${PROJECT_NAME}-targets
        RUNTIME DESTINATION ${PROJECT_RUNTIME_INSTALL_DIR}
        LIBRARY DESTINATION ${PROJECT_LIBRARY_INSTALL_DIR}
        ARCHIVE DESTINATION ${PROJECT_LIBRARY_INSTALL_DIR}
        )

install(FILES ${hdrs} DESTINATION ${PROJECT_INCLUDE_INSTALL_DIR})

include(CMakePackageConfigHelpers)

set(PROJECT_CONFIG_INSTALL_DIR share/${PROJECT_NAME} )

set(INSTALL_INCLUDE_DIR ${PROJECT_INCLUDE_INSTALL_DIR})
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/package-config.cmakein ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
                              INSTALL_DESTINATION ${PROJECT_CONFIG_INSTALL_DIR}
                              PATH_VARS INSTALL_INCLUDE_DIR
                              )

write_basic_package_version_file( ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion )

install(FILES   ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake 
				${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
        DESTINATION ${PROJECT_CONFIG_INSTALL_DIR} )


install(EXPORT ${PROJECT_NAME}-targets 
		DESTINATION ${PROJECT_CONFIG_INSTALL_DIR})