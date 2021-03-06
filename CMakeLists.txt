CMAKE_MINIMUM_REQUIRED(VERSION 3.8)
PROJECT(SEGS)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

IF(MSVC)
    add_definitions( -D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_WARNINGS -DNOMINMAX -D_USE_MATH_DEFINES)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj")
ELSEIF ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
   SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic -Wall -Wextra -Wuseless-cast -std=c++14")
   
   if(MINGW)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wa,-mbig-obj")
   endif()
ELSEIF("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic -Wall -Wextra -Wuseless-cast -std=c++14 -Wno-nested-anon-types -Wno-gnu-anonymous-struct -Wno-gnu-zero-variadic-macro-arguments")
endif()
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(AUTOGEN_TARGETS_FOLDER ON)

add_definitions(-D__STDC_FORMAT_MACROS)

SET(MAIN_INCLUDE_PATH ${PROJECT_SOURCE_DIR}/3rd_party/include CACHE PATH "Includes")
SET(MAIN_LIB_PATH ${PROJECT_SOURCE_DIR}/3rd_party/lib CACHE PATH "Libraries")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/out)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/out)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/out)
SET(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/out)
SET(CMAKE_INCLUDE_PATH ${MAIN_INCLUDE_PATH})
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/CMakeScripts;${CMAKE_MODULE_PATH})
SET(UNICODE TRUE)
set(INSTALLED_TARGETS)
OPTION(ENABLE_SCRIPTING_ENGINE "Build experimental scripting engine ?" ON)
OPTION(ENABLE_TESTS "Enable testing unit and functional tests" OFF)
OPTION(BUILD_COX_MAP_VIEWER "Build CoX map file viewer ?" OFF)

########################################################################################
# Prerequisites
########################################################################################
find_package(Threads REQUIRED)

include(3rdparty_support)
set(ThirdParty_Install_Dir ${CMAKE_BINARY_DIR}/3rd_party/built) # ERICEDIT: Changed to CMAKE_BINARY_DIR so that this variable always points to the top level of the build directory.
add_subdirectory(3rd_party)
link_directories(${PROJECT_SOURCE_DIR}/lib) # ERICEDIT: Changed to lowercase.

include(LocateQt5)
find_package(Qt5 COMPONENTS Core Sql REQUIRED)

if(ENABLE_TESTS)
    enable_testing()
    find_package(Qt5 COMPONENTS Test REQUIRED)
    # Event registration support functionality
    macro(add_segs_test testname) # followed by a list of directories to scan
        add_executable(${testname} ${testname}.cpp)
        add_test(${testname} ${testname})

        target_link_libraries(${testname}
            Qt5::Test
            ${ARGN}
        )
    endmacro()
endif()

link_directories(${ThirdParty_Install_Dir}/lib)
include_directories(SYSTEM ${ThirdParty_Install_Dir}/include)
include_directories(SYSTEM ${PROJECT_SOURCE_DIR}/3rd_party)
include_directories(${PROJECT_SOURCE_DIR}/include)

include_directories(Components)

# this has to be visited before scan_event_types macro below has been used
add_subdirectory(Utilities)
# this function is here to solve the env paths issue
function(to_path_list var path1)
    if("${CMAKE_HOST_SYSTEM}" MATCHES ".*Windows.*")
        set(sep "\\;")
    else()
        set(sep ":")
    endif()
    set(result "${path1}") # First element doesn't require separator at all...
    foreach(path ${ARGN})
        set(result "${result}${sep}${path}") # .. but other elements do.
    endforeach()
    set(${var} "${result}" PARENT_SCOPE)
endfunction()
##############################################
# Event registration support functionality
macro(scan_event_types group) # followed by a list of directories to scan
    set(all_headers)
    foreach(path ${ARGN})
        message("Scanning ${path} for events")
        file(GLOB headers ${path}/*.h)
        list(APPEND all_headers ${headers})
    endforeach()
    to_path_list(path_with_qt5 $ENV{PATH} ${Qt5_BIN_DIR} )
    # for event_gen we first add qt5 binary path to allow it to run during build.
    add_custom_command(
        COMMAND ${CMAKE_COMMAND} -E env "PATH=${path_with_qt5}" $<SHELL_PATH:$<TARGET_FILE:event_gen>> scan_events ${group} ${CMAKE_CURRENT_BINARY_DIR}/ ${ARGN}
        OUTPUT  ${CMAKE_CURRENT_BINARY_DIR}/${group}.gen.cpp
        DEPENDS event_gen ${all_headers}
        COMMENT "Generating ${group} implementation source"
    )
endmacro()
macro(add_event_sources group tgt_var)
    list(APPEND ${tgt_var} ${CMAKE_CURRENT_BINARY_DIR}/${group}.gen.cpp)
    message("adding ${group} to sources")
endmacro()
macro(add_messages_library name )
    #find handler's events
    scan_event_types(${name}Events ${CMAKE_CURRENT_SOURCE_DIR})
    add_event_sources(${name}Events ${name}_SOURCES)
    add_library(${name}_interface STATIC ${${name}_SOURCES} ${ARGN})
    # allow all headers to refer to events by their dir/name.h
    target_include_directories(${name}_interface PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/..)
    target_link_libraries(${name}_interface PUBLIC Qt5::Core)
    # all interfaces use stuff from Common, so should depend/link to it
    target_link_libraries(${name}_interface PUBLIC SEGS_Components)
	source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${ARGN}) 
	set_target_properties (${name}_interface PROPERTIES
		FOLDER Projects/CoX/Common/Messages
	)
endmacro()

macro(add_handler name)
    # TODO: consider add_subdirectory(../../Common/Messages/${name}) to force existence/building of interface lib
    # add the implementation library
    add_library(${name} STATIC ${ARGN})
    target_link_libraries(${name} PUBLIC ${name}_interface)
endmacro()
##############################################

add_subdirectory(Components)
add_subdirectory(Projects)

file(MAKE_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/data)

set(MAPNAMES
  City_00_01
  City_01_01 City_01_02 City_01_03
  City_02_01 City_02_02
  City_03_01 City_03_02
  City_04_01 City_04_02
  City_05_01
  Hazard_01_01
  Hazard_02_01
  Hazard_03_01
  Hazard_04_01 Hazard_04_02
  Hazard_05_01
  Trial_01_01 Trial_01_02
  Trial_02_01
  Trial_03_01
  Trial_04_01 Trial_04_02
  Trial_05_01
  )
foreach(MAPDIR ${MAPNAMES})
  file(MAKE_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/DefaultMapInstances/${MAPDIR})
endforeach(MAPDIR)
