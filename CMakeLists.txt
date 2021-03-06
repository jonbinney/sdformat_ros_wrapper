cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

if(COMMAND CMAKE_POLICY)
  CMAKE_POLICY(SET CMP0003 NEW)
  CMAKE_POLICY(SET CMP0004 NEW)
endif(COMMAND CMAKE_POLICY)
enable_testing()

# with -fPIC
if(UNIX AND NOT WIN32)
  set (CMAKE_INSTALL_PREFIX "/usr" CACHE STRING "Install Prefix")
 find_program(CMAKE_UNAME uname /bin /usr/bin /usr/local/bin )
 if(CMAKE_UNAME)
   exec_program(uname ARGS -m OUTPUT_VARIABLE CMAKE_SYSTEM_PROCESSOR)
   set(CMAKE_SYSTEM_PROCESSOR ${CMAKE_SYSTEM_PROCESSOR} CACHE INTERNAL
     "processor type (i386 and x86_64)")
   if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
     ADD_DEFINITIONS(-fPIC)
   endif(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
 endif(CMAKE_UNAME)
endif()

project (SDFormat)
string (TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)

set (SDF_MAJOR_VERSION 1)
set (SDF_MINOR_VERSION 4)
set (SDF_PATCH_VERSION 11)

set (SDF_VERSION ${SDF_MAJOR_VERSION}.${SDF_MINOR_VERSION})
set (SDF_VERSION_FULL ${SDF_MAJOR_VERSION}.${SDF_MINOR_VERSION}.${SDF_PATCH_VERSION})

message (STATUS "${PROJECT_NAME} version ${SDF_VERSION_FULL}")
set (CMAKE_INCLUDE_DIRECTORIES_PROJECT_BEFORE ON)

# Include GNUInstallDirs to get canonical paths
include(GNUInstallDirs)

# Default test type for test all over source
set(TEST_TYPE "UNIT")

# developer's option to cache PKG_CONFIG_PATH and
# LD_LIBRARY_PATH for local installs
if(PKG_CONFIG_PATH)
  set (ENV{PKG_CONFIG_PATH} ${PKG_CONFIG_PATH}:$ENV{PKG_CONFIG_PATH})
endif()
if(LD_LIBRARY_PATH)
  set (ENV{LD_LIBRARY_PATH} ${LD_LIBRARY_PATH}:$ENV{LD_LIBRARY_PATH})
endif()

set (INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_INCLUDEDIR}/sdformat-${SDF_VERSION}/sdf") 
set (LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR} CACHE STRING "Installation directory for libraries (relative to CMAKE_INSTALL_PREFIX)") 
set (BIN_INSTALL_DIR ${CMAKE_INSTALL_BINDIR} CACHE STRING "Installation directory for binaries (relative to CMAKE_INSTALL_PREFIX)") 

set (USE_FULL_RPATH OFF CACHE BOOL "Set to true to enable full rpath")


if (USE_FULL_RPATH)
  # use, i.e. don't skip the full RPATH for the build tree
  set(CMAKE_SKIP_BUILD_RPATH  FALSE)
  
  # when building, don't use the install RPATH already
  # (but later on when installing)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 
  
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}")
  
  # add the automatically determined parts of the RPATH
  # which point to directories outside the build tree to the install RPATH
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
  
  # the RPATH to be used when installing, but only if its not a system directory
  list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}" isSystemDir)
  if("${isSystemDir}" STREQUAL "-1")
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}")
  endif("${isSystemDir}" STREQUAL "-1")
endif()

set (BUILD_SDF ON CACHE INTERNAL "Build SDF" FORCE)
set (build_errors "" CACHE INTERNAL "build errors" FORCE)
set (build_warnings "" CACHE INTERNAL "build warnings" FORCE)

set (sdf_cmake_dir ${PROJECT_SOURCE_DIR}/cmake CACHE PATH
     "Location of CMake scripts")

#####################################
# Build type cflags
set (CMAKE_C_FLAGS_ALL " -Wall -Wextra -Wno-long-long -Wno-unused-value -Wno-unused-value -Wno-unused-value -Wno-unused-value -Wfloat-equal -Wshadow -Winit-self -Wswitch-default -Wmissing-include-dirs -pedantic" CACHE INTERNAL "General C Flags")

#####################################
# Build type link flags
set (CMAKE_LINK_FLAGS_RELEASE " " CACHE INTERNAL "Link flags fro release" FORCE)
set (CMAKE_LINK_FLAGS_DEBUG " " CACHE INTERNAL "Link flags for debug" FORCE)

set (CMAKE_C_FLAGS_RELEASE "")
if (NOT APPLE)
  # -s doesn't work with default osx compiler clang, alternative:
  # http://stackoverflow.com/questions/6085491/gcc-vs-clang-symbol-strippingu
  set (CMAKE_C_FLAGS_RELEASE " -s ")
endif()
set (CMAKE_C_FLAGS_RELEASE " ${CMAKE_C_FLAGS_RELEASE} -O2 -DNDEBUG ${CMAKE_C_FLAGS_ALL}"
  CACHE INTERNAL "C Flags for release" FORCE)
set (CMAKE_CXX_FLAGS_RELEASE ${CMAKE_C_FLAGS_RELEASE})

set (CMAKE_C_FLAGS_DEBUG " -ggdb3 ${CMAKE_C_FLAGS_ALL}"
  CACHE INTERNAL "C Flags for debug" FORCE)
set (CMAKE_CXX_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})

#####################################
# Set the default build type
if (NOT CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE "RELEASE" CACHE STRING 
    "Choose the type of build, options are: debug release profile check" FORCE)
endif (NOT CMAKE_BUILD_TYPE)
string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE)

#####################################
# Set all the global build flags
set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE}}")
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS_${CMAKE_BUILD_TYPE}}")
set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_LINK_FLAGS_${CMAKE_BUILD_TYPE}}")
set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_LINK_FLAGS_${CMAKE_BUILD_TYPE}}")
set (CMAKE_MODULE_LINKER_FLAGS "${CMAKE_LINK_FLAGS_${CMAKE_BUILD_TYPE}}")

if ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  set (BUILD_TYPE_RELEASE TRUE)
else()
  set (BUILD_TYPE_RELEASE FALSE)
endif()

if ( "${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  set (BUILD_TYPE_DEBUG TRUE)
else()
  set (BUILD_TYPE_DEBUG FALSE)
endif()

message (STATUS "\n\n====== Finding 3rd Party Packages ======")
include (${sdf_cmake_dir}/SearchForStuff.cmake)
message (STATUS "----------------------------------------\n")


#################################################
# OS Specific initialization
if (UNIX)
  sdf_setup_unix()
else (WIN32)
  sdf_setup_windows()
else (APPLE)
  sdf_setup_apple()
endif()

#################################################
# Print warnings and errors
if ( build_warnings )
  message(STATUS "BUILD WARNINGS")
  foreach (msg ${build_warnings})
    message(STATUS ${msg})
  endforeach ()
  message(STATUS "END BUILD WARNINGS\n")
endif (build_warnings)

########### Add uninstall target ###############
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)
add_custom_target(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake")

if (build_errors)
  message(STATUS "BUILD ERRORS: These must be resolved before compiling.")
  foreach (msg ${build_errors})
    message(STATUS ${msg})
  endforeach ()
  message(STATUS "END BUILD ERRORS\n")
  message (FATAL_ERROR "Errors encountered in build. Please see the BUILD ERRORS above.")

else (buid_errors)

  ########################################
  # Write the config.h file
  configure_file (${sdf_cmake_dir}/sdf_config.h.in
    ${PROJECT_BINARY_DIR}/sdf/sdf_config.h)
  sdf_install_includes("" ${PROJECT_BINARY_DIR}/sdf/sdf_config.h)

  message (STATUS "C Flags:${CMAKE_CXX_FLAGS}")
  message (STATUS "Build Type: ${CMAKE_BUILD_TYPE}")
  message (STATUS "Install path: ${CMAKE_INSTALL_PREFIX}")
 
  if (BUILD_SDF)
    include_directories(include
      ${PROJECT_BINARY_DIR}
      ${PROJECT_BINARY_DIR}/include
      )

    add_subdirectory(test)
    add_subdirectory(src)
    add_subdirectory(include/sdf)
    add_subdirectory(sdf)
    add_subdirectory(doc)
  endif(BUILD_SDF)

  ########################################
  # Make the package config file
  configure_file(${CMAKE_SOURCE_DIR}/cmake/sdformat_pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake/sdformat.pc @ONLY)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cmake/sdformat.pc DESTINATION
    ${LIB_INSTALL_DIR}/pkgconfig COMPONENT pkgconfig)

  ########################################
  # Make the cmake config files
  set(PKG_NAME ${PROJECT_NAME})
  set(PKG_LIBRARIES sdformat)
  set(PKG_DEPENDS Boost)
  set(cmake_conf_file "cmake/sdf_config.cmake")
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/${cmake_conf_file}.in" "${CMAKE_CURRENT_BINARY_DIR}/SDFormatConfig.cmake" @ONLY)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/SDFormatConfig.cmake DESTINATION ${LIB_INSTALL_DIR}/cmake/${PROJECT_NAME_LOWER}/ COMPONENT cmake)

  ########################################
  # Package Creation: 
  include (${sdf_cmake_dir}/sdf_cpack.cmake)
  set (CPACK_PACKAGE_VERSION "${SDF_VERSION_FULL}")
  set (CPACK_PACKAGE_VERSION_MAJOR "${SDF_MAJOR_VERSION}")
  set (CPACK_PACKAGE_VERSION_MINOR "${SDF_MINOR_VERSION}")
  set (CPACK_PACKAGE_VERSION_PATCH "${SDF_PATCH_VERSION}")

  if (CPACK_GENERATOR)
    message(STATUS "Found CPack generators: ${CPACK_GENERATOR}")

    configure_file("${sdf_cmake_dir}/cpack_options.cmake.in"
                   ${SDF_CPACK_CFG_FILE} @ONLY)
    set(CPACK_PROJECT_CONFIG_FILE ${SDF_CPACK_CFG_FILE})
    include (CPack)
  endif()

  message(STATUS "Configuration successful. Type make to compile sdf")
endif(build_errors)
