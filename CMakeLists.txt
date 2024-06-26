cmake_minimum_required(VERSION 3.5)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_CONFIGURATION_TYPES "Release;Debug;RelWithDebInfo" CACHE STRING INTERNAL FORCE)
set(CMAKE_SKIP_INSTALL_RULES ON)
set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY ON)
set(CMAKE_SUPPRESS_REGENERATION ON)
include(CheckCXXCompilerFlag)

enable_testing()
configure_file(${CMAKE_SOURCE_DIR}/CTestCustom.cmake ${CMAKE_BINARY_DIR})

project(Dynex)

include_directories(include src external "${CMAKE_BINARY_DIR}/version")

if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64" OR ${CMAKE_SYSTEM_PROCESSOR} STREQUAL "arm64")
  set(ARM64 ON)
else()
  set(ARM64 OFF)
endif()

set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads REQUIRED)

if(MSVC)
  include_directories(src/Platform/Windows)
  set(STATIC_DEFAULT ON)
elseif(APPLE)
  include_directories(src/Platform/OSX)
  include_directories(src/Platform/Posix)
  set(STATIC_DEFAULT ON)
else()
  include_directories(src/Platform/Linux)
  include_directories(src/Platform/Posix)
  set(STATIC_DEFAULT OFF)
endif()

option(STATIC "Link libraries statically" ${STATIC_DEFAULT})

if(MSVC)
  add_compile_options("/std:c++17")
  add_definitions("/bigobj /MP /W3 /GS- /D_CRT_SECURE_NO_WARNINGS /wd4996 /wd4345 /D_WIN32_WINNT=0x0600 /DWIN32_LEAN_AND_MEAN /DGTEST_HAS_TR1_TUPLE=0 /D_VARIADIC_MAX=8 /D__SSE4_1__ /d2archSSE42")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:10485760")
  if(STATIC)
    foreach(VAR CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_RELWITHDEBINFO CMAKE_CXX_FLAGS_RELWITHDEBINFO CMAKE_C_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELEASE)
      string(REPLACE "/MD" "/MT" ${VAR} "${${VAR}}")
    endforeach()
  endif()
  include_directories(SYSTEM src/platform/msc)
else()
  if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    # This option has no effect in glibc version less than 2.20. 
    # Since glibc 2.20 _BSD_SOURCE is deprecated, this macro is recomended instead
    add_definitions("-D_DEFAULT_SOURCE -D_GNU_SOURCE")
  endif()
  set(ARCH native CACHE STRING "CPU to build for: -march value or default")
  # Apple clang is slow with -march=native
  if(("${ARCH}" STREQUAL "default") OR (APPLE AND CMAKE_C_COMPILER_ID MATCHES "Clang"))
    set(ARCH_FLAG "")
  else()
    set(ARCH_FLAG "-march=${ARCH}")
  endif()
  set(WARNINGS "-Wall -Wextra -Wpointer-arith -Wundef -Wvla -Wwrite-strings -Wno-error=extra -Wno-error=deprecated-declarations -Wno-error=sign-compare -Wno-error=strict-aliasing -Wno-error=type-limits -Wno-error=undef -Wno-unused-parameter -Wno-undef -Wno-unused-const-variable -Wno-unused-variable -Wno-write-strings -Wno-unused-function ")
  if(CMAKE_C_COMPILER_ID MATCHES "Clang")
    set(WARNINGS "${WARNINGS} -Wno-error=mismatched-tags -Wno-error=null-conversion -Wno-overloaded-shift-op-parentheses -Wno-error=shift-count-overflow -Wno-error=tautological-constant-out-of-range-compare -Wno-error=unneeded-internal-declaration -Wno-unused-private-field")
  else()
    set(WARNINGS "${WARNINGS} -Wlogical-op -Wno-error=maybe-uninitialized -Wno-error=clobbered -Wno-error=unused-but-set-variable")
  endif()
  if(MINGW)
    set(WARNINGS "${WARNINGS} -Wno-error=unused-value")
    set(MINGW_FLAG "-DWIN32_LEAN_AND_MEAN")
    include_directories(SYSTEM src/platform/mingw)
  else()
    set(MINGW_FLAG "")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 5.1))
    set(WARNINGS "${WARNINGS} -Wno-error=odr")
  endif()
  set(C_WARNINGS "-Waggregate-return -Wnested-externs -Wold-style-definition -Wstrict-prototypes")
  set(CXX_WARNINGS "-Wno-reorder -Wno-missing-field-initializers")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c17 ${MINGW_FLAG} ${WARNINGS} ${C_WARNINGS} ${ARCH_FLAG}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 ${MINGW_FLAG} ${WARNINGS} ${CXX_WARNINGS} ${ARCH_FLAG}")
  if(UNIX)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_THREAD_LIBS_INIT}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_THREAD_LIBS_INIT}")
  endif()
  if(NOT ARM64)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -maes")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes")
  endif()
  #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGTEST_HAS_TR1_TUPLE=0")

  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.8))
    set(DEBUG_FLAGS "-g3 -Og")
  else()
    set(DEBUG_FLAGS "-g3 -O0")
  endif()

  set(RELEASE_FLAGS "-Ofast -DNDEBUG -Wno-unused-variable -fno-strict-aliasing")

  if(NOT APPLE)
    # There is a clang bug that does not allow to compile code that uses AES-NI intrinsics if -flto is enabled
    if (CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_SYSTEM_NAME STREQUAL "Linux" 
        AND CMAKE_BUILD_TYPE STREQUAL "Release" AND ((CMAKE_C_COMPILER_VERSION VERSION_GREATER 4.9) OR (CMAKE_C_COMPILER_VERSION VERSION_EQUAL 4.9)))
      # On linux, to build in lto mode, check that ld.gold linker is used: 'update-alternatives --install /usr/bin/ld ld /usr/bin/ld.gold HIGHEST_PRIORITY'
      set(CMAKE_AR gcc-ar)
      set(CMAKE_RANLIB gcc-ranlib)
    endif()
    set(RELEASE_FLAGS "${RELEASE_FLAGS} -flto")
  endif()
  #if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT MINGW)
  #  set(RELEASE_FLAGS "${RELEASE_FLAGS} -fno-fat-lto-objects")
  #endif()
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  if(STATIC AND NOT APPLE)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++ ")
  endif()
endif()

# CURL is requred:
FIND_PACKAGE(CURL REQUIRED)
IF(CURL_FOUND)
  INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIRS})
ELSE(CURL_FOUND)
  MESSAGE(FATAL_ERROR "Could not find the CURL library and development files.")
ENDIF(CURL_FOUND)

# Boost is required:
if(STATIC)
  MESSAGE(STATUS "LIB BOOST: Static linking")
  set(Boost_USE_STATIC_LIBS ON)
endif()
find_package(Boost 1.55 REQUIRED COMPONENTS filesystem program_options)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
if(MINGW)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};ws2_32;mswsock")
elseif(APPLE)
  set(Boost_LIBRARIES "${Boost_LIBRARIES}")
elseif(NOT MSVC)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};rt")
endif()

set(COMMIT_ID_IN_VERSION ON CACHE BOOL "Include commit ID in version")
file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/version")
if (NOT COMMIT_ID_IN_VERSION)
  set(VERSION "${VERSION}-unknown")
  configure_file("src/version.h.in" "version/version.h")
  add_custom_target(version ALL)
elseif(DEFINED COMMIT)
  string(REPLACE "." "\\." VERSION_RE "${VERSION}")
  if(NOT REFS MATCHES "(\\(|, )tag: v${VERSION_RE}(\\)|, )")
    set(VERSION "${VERSION}-g${COMMIT}")
  endif()
  configure_file("src/version.h.in" "version/version.h")
  add_custom_target(version ALL)
else()
  find_package(Git QUIET)
  if(Git_FOUND OR GIT_FOUND)
    message(STATUS "Found Git: ${GIT_EXECUTABLE}")
    add_custom_target(version ALL "${CMAKE_COMMAND}" "-D" "VERSION=${VERSION}" "-D" "GIT=${GIT_EXECUTABLE}" "-D" "TO=${CMAKE_BINARY_DIR}/version/version.h" "-P" "src/version.cmake" WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")
  else()
    message(STATUS "WARNING: Git was not found!")
    set(VERSION "${VERSION}-unknown")
    configure_file("src/version.h.in" "version/version.h")
    add_custom_target(version ALL)
  endif()
endif()

file(STRINGS "src/version.h.in" CNVER REGEX "#define CN_VER_.*")
foreach(VAL CN_VER_MAJOR CN_VER_MINOR CN_VER_REV CN_VER_WAL CN_VER_BUILD)
  if(CNVER MATCHES "#define ${VAL} ([0-9]+)")
    #message(STATUS "${VAL}=${CMAKE_MATCH_1}")
    set(${VAL} "${CMAKE_MATCH_1}")
  endif()
endforeach()
MESSAGE(STATUS "VERSION: ${CN_VER_MAJOR}.${CN_VER_MINOR}.${CN_VER_REV}(.${CN_VER_WAL})-${CN_VER_BUILD}")

add_subdirectory(external)
add_subdirectory(src)
add_subdirectory(tests)

MESSAGE(STATUS "BUILD: ${CMAKE_BUILD_TYPE}")
