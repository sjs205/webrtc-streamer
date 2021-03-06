cmake_minimum_required (VERSION 3.0.0)
project (webrtc-streamer)
include(CMakeParseArguments)

# Function to execute a processes and exit CMake on error
function (exec_proc_and_die_on_error)
  cmake_parse_arguments(PARAM "" "WORKING_DIR" "COMMAND" ${ARGN})

  EXECUTE_PROCESS(WORKING_DIRECTORY ${PARAM_WORKING_DIR}
                  COMMAND ${PARAM_COMMAND}
                  RESULT_VARIABLE cmd_result
                  ERROR_VARIABLE cmd_error)

  if (NOT cmd_result STREQUAL "0")
    string (REPLACE ";" " " command_pretty_str "${PARAM_COMMAND}")
    string(CONCAT display_msg
           "Failed to execute command \"${command_pretty_str}\"\n"
           "Workding dir = \"${PARAM_WORKING_DIR}\"\n"
           "Result       = \"${cmd_result}\"\n"
           "Error        = \"${cmd_error}\".")
    message(FATAL_ERROR ${display_msg})
  endif()
endfunction()

# Function to find a program end exit CMake if it cannot be found
function (require_program program_name hint)
  find_program(PROG_${program_name} ${program_name})
  if (PROG_${program_name} MATCHES "PROG_${program_name}-NOTFOUND")
     message (FATAL_ERROR "${hint} - ${PROG_${program_name}}")
  else()
     message (STATUS "Program ${program_name} found: ${PROG_${program_name}}")
     set(PROG_${program_name} ${PROG_${program_name}} PARENT_SCOPE)
  endif ()
endfunction()


set (WEBRTCROOT "../webrtc" CACHE STRING "WEBRTC root directory")
set (WEBRTCBUILD "Release" CACHE STRING "WEBRTC build type")
set (WEBRTCDESKTOPCAPTURE "ON" CACHE STRING "WEBRTC Desktop capture")

MESSAGE("WEBRTCBUILD = ${WEBRTCBUILD}\nWEBRTCROOT = ${WEBRTCROOT}\nWEBRTCDESKTOPCAPTURE= ${WEBRTCDESKTOPCAPTURE}")

set (CMAKE_BUILD_TYPE ${WEBRTCBUILD}) 
set (CMAKE_POSITION_INDEPENDENT_CODE ON)
set (CMAKE_CXX_STANDARD 11)
set (USE_X11 OFF)

# build version identifier
find_package(Git)
if(GIT_FOUND)
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} submodule update --init)
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} describe --tags --always --dirty OUTPUT_VARIABLE PROJECTVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${PROJECTVERSION}")
	
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C civetweb describe --tags --always --dirty OUTPUT_VARIABLE CIVETVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} civetweb@${CIVETVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C ${WEBRTCROOT}/src describe --tags --always --dirty OUTPUT_VARIABLE WEBRTCVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} webrtc@${WEBRTCVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C live555helper describe --tags --always --dirty OUTPUT_VARIABLE LIVEVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} live555helper@${LIVEVERSION}")
endif()
add_definitions(-DVERSION=\"${VERSION}\")

# target
FILE(GLOB SOURCE src/*.cpp)
add_executable (${PROJECT_NAME} ${SOURCE})
target_include_directories(${PROJECT_NAME} PRIVATE inc) 

# thread
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package (Threads)
target_link_libraries (${PROJECT_NAME} Threads::Threads) 

# civetweb
add_library (civetweb STATIC civetweb/src/civetweb.c civetweb/src/CivetServer.cpp)	
target_link_libraries (${PROJECT_NAME} civetweb) 
target_include_directories(civetweb PUBLIC civetweb/include) 

# webrtc
if (NOT EXISTS ${WEBRTCROOT}/src/out/${WEBRTCBUILD}/args.gn)
  message ("Attempting to rebuild WebRTC.")
  set (WEBRTCARGS rtc_include_tests=false\nrtc_enable_protobuf=false\nuse_custom_libcxx=false\nrtc_include_pulse_audio=false\nrtc_build_examples=false\nrtc_build_tools=false\nis_clang=false\ntreat_warnings_as_errors=false\n)
  if (MSVC)
	  set (WEBRTCARGS rtc_use_h264=false\n${WEBRTCARGS})
  else ()
	  set (WEBRTCARGS rtc_use_h264=true\nffmpeg_branding="Chrome"\n${WEBRTCARGS})
  endif()
  if(WEBRTCBUILD STREQUAL "Release")
    set (WEBRTCARGS is_debug=false\n${WEBRTCARGS})
  else()
  	set (WEBRTCARGS is_debug=true\n${WEBRTCARGS})
  endif()
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm.*")
    set (WEBRTCARGS target_cpu="arm"\n${WEBRTCARGS})
	set (WEBRTCARGS use_sysroot=true\n${WEBRTCARGS})
  else()
    set (WEBRTCARGS use_sysroot=false\n${WEBRTCARGS})
  endif()

  find_package(PkgConfig QUIET)
  pkg_check_modules(GTK3 QUIET gtk+-3.0)
  MESSAGE("   - GTK_FOUND = ${GTK3_FOUND}")
  if(NOT GTK3_FOUND OR (WEBRTCDESKTOPCAPTURE STREQUAL "OFF"))
 	set (WEBRTCARGS use_ozone=true\nis_desktop_linux=false\n${WEBRTCARGS})
  endif()
  FILE(WRITE ${WEBRTCROOT}/src/out/${WEBRTCBUILD}/args.gn ${WEBRTCARGS})

  if (WIN32)
  	SET (SHELLCOMMAND cmd /c )
  endif(WIN32)

  message ("   - Generating ninja files...")
  require_program(gn "GN meta-build system not found! Have you added the depot_tools to your path?")
  exec_proc_and_die_on_error(WORKING_DIR "${WEBRTCROOT}/src/out/${WEBRTCBUILD}" COMMAND ${SHELLCOMMAND} ${PROG_gn} gen .)

  require_program(ninja "Ninja not installed!")
  message ("   - Doing build: ${WEBRTCROOT}/src/out/${WEBRTCBUILD} COMMAND ${PROG_ninja} ${NINJA_TARGET}")
  SET(NINJA_TARGET webrtc rtc_json jsoncpp builtin_video_decoder_factory builtin_video_encoder_factory peerconnection p2p_server_utils)
  exec_proc_and_die_on_error(WORKING_DIR "${WEBRTCROOT}/src/out/${WEBRTCBUILD}" COMMAND ninja ${NINJA_TARGET})

  message ("   - Finished building WebRTC...")
endif() 

set (WEBRTCINCLUDE ${WEBRTCROOT}/src ${WEBRTCROOT}/src/third_party/abseil-cpp ${WEBRTCROOT}/src/third_party/jsoncpp/source/include ${WEBRTCROOT}/src/third_party/libyuv/include)
target_include_directories(${PROJECT_NAME} PRIVATE ${WEBRTCINCLUDE})

set (WEBRTCOBJS ${WEBRTCROOT}/src/out/${WEBRTCBUILD}/obj)

FILE(GLOB_RECURSE WEBRTBBRCOBJS ${WEBRTCOBJS}/modules/congestion_controller/bbr/*${CMAKE_C_OUTPUT_EXTENSION})
FILE(GLOB_RECURSE WEBRTJSONCPPCOBJS ${WEBRTCOBJS}/third_party/jsoncpp/jsoncpp/*${CMAKE_C_OUTPUT_EXTENSION})
FILE(GLOB_RECURSE WEBRTP2POBJ ${WEBRTCOBJS}/p2p/p2p_server_utils/*${CMAKE_C_OUTPUT_EXTENSION})
SET (WEBRTCEXTRAOBJS ${WEBRTJSONCPPCOBJS} ${WEBRTBBRCOBJS} ${WEBRTP2POBJ} ${WEBRTCOBJS}/rtc_base/rtc_json/json${CMAKE_C_OUTPUT_EXTENSION})	

find_library(WEBRTC_LIBRARY NAMES webrtc PATHS ${WEBRTCOBJS})
target_link_libraries (${PROJECT_NAME} ${WEBRTC_LIBRARY}) 
find_library(WEBRTC_VIDEO_DECODER_LIBRARY NAMES builtin_video_decoder_factory PATHS ${WEBRTCOBJS}/api/video_codecs/)
find_library(WEBRTC_VIDEO_ENCODER_LIBRARY NAMES builtin_video_encoder_factory PATHS ${WEBRTCOBJS}/api/video_codecs/)
find_library(WEBRTC_PEERCONNECTION_LIBRARY NAMES peerconnection PATHS ${WEBRTCOBJS}/pc)
target_link_libraries (${PROJECT_NAME} ${WEBRTC_VIDEO_DECODER_LIBRARY} ${WEBRTC_VIDEO_ENCODER_LIBRARY} ${WEBRTC_PC_LIBRARY} ${WEBRTC_PEERCONNECTION_LIBRARY}) 

add_library (webrtcextra STATIC ${WEBRTCEXTRAOBJS})
set_target_properties(webrtcextra PROPERTIES LINKER_LANGUAGE CXX)		
target_link_libraries (${PROJECT_NAME} webrtcextra) 


# live555helper & live555
if (NOT EXISTS live)
    if (NOT EXISTS live555-latest.tar.gz)
        message ("Downloading Live555 Streaming Media...")
        file (DOWNLOAD http://www.live555.com/liveMedia/public/live555-latest.tar.gz ${CMAKE_SOURCE_DIR}/live555-latest.tar.gz )
    endif()

    EXECUTE_PROCESS(
        COMMAND 7z t live555-latest.tar.gz
        RESULT_VARIABLE p7z_result)
    if(NOT p7z_result STREQUAL "0")
        message(FATAL_ERROR "The live555 archive is corrupt!")
    endif()

    message("Extracting Live 555 Streaming Media...")
    require_program(7z "Utility 7zip not found but is required!")
    EXECUTE_PROCESS(
        COMMAND 7z x live555-latest.tar.gz -so
        COMMAND 7z x -aoa -ttar -si
        RESULT_VARIABLE p7z_result
    )
    if(NOT p7z_result STREQUAL "0")
        message(FATAL_ERROR "Downloading or extracting live555 failed!")
    endif()
endif(NOT EXISTS live) 

FILE(GLOB LIVEHELPERSOURCE live/groupsock/*.c* live/liveMedia/*.c* live/UsageEnvironment/*.c* live/BasicUsageEnvironment/*.c* live555helper/src/*.cpp)
add_library (live555helper STATIC ${LIVEHELPERSOURCE})	
target_include_directories(live555helper PUBLIC live/groupsock/include live/liveMedia/include live/UsageEnvironment/include live/BasicUsageEnvironment/include live555helper/inc)
target_link_libraries (${PROJECT_NAME} live555helper) 
add_definitions(-DHAVE_LIVE555 -DHAVE_JPEG)

# compiler specific
if (MSVC)
	# live555
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	target_link_libraries (${PROJECT_NAME} ws2_32)
	# getopt
	add_library (getopt getopt/getopt.c)
	target_include_directories(getopt PUBLIC getopt) 
	target_link_libraries (${PROJECT_NAME} getopt)
	# webrtc
	add_definitions(-DWEBRTC_WIN -EHsc -DNOMINMAX -D__PRETTY_FUNCTION__=__FUNCTION__ -D_HAS_ITERATOR_DEBUGGING=0)
	set(CompilerFlags CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_C_FLAGS_RELEASE)
	foreach(CompilerFlag ${CompilerFlags})
	  string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
	endforeach()
	if (USE_X11)
		add_definitions(-DUSE_X11)
	endif()
	target_link_libraries (${PROJECT_NAME} secur32 dmoguids wmcodecdspuuid strmiids msdmo winmm dxgi d3d11)
else(MSVC)
	# live555
	add_definitions(-DBSD=1 -DSOCKLEN_T=socklen_t -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE=1)
	# webrtc
	add_definitions(-DWEBRTC_POSIX -fno-rtti)
	target_link_libraries (${PROJECT_NAME} dl -static-libgcc -static-libstdc++)
	if (USE_X11)
		if (EXISTS ${WEBRTCROOT}/src/out/${WEBRTCBUILD}/obj/modules/desktop_capture/desktop_capture.ninja)
			add_definitions(-DUSE_X11)
			target_link_libraries (${PROJECT_NAME} X11 Xext Xdamage Xfixes Xcomposite)
		endif()
	endif()
endif(MSVC)

#cpack
install (TARGETS ${PROJECT_NAME} RUNTIME DESTINATION .)
install (DIRECTORY html DESTINATION .)
install (FILES config.json DESTINATION .)
install (FILES Procfile DESTINATION .)
SET(CPACK_GENERATOR "TGZ")
SET(CPACK_SYSTEM_NAME ${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_BUILD_TYPE})
SET(CPACK_PACKAGE_CONTACT "michel.promonet@free.fr") 
if(PROJECTVERSION)
    SET(CPACK_PACKAGE_VERSION "${PROJECTVERSION}")
endif()
INCLUDE(CPack)
