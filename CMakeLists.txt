cmake_minimum_required(VERSION 2.8)
project(ebusd)

file(STRINGS "VERSION" VERSION)
set(PACKAGE ${CMAKE_PROJECT_NAME})
set(PACKAGE_NAME ${CMAKE_PROJECT_NAME})
set(PACKAGE_TARNAME ${CMAKE_PROJECT_NAME})
set(PACKAGE_VERSION ${VERSION})
set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(PACKAGE_BUGREPORT "ebusd@ebusd.eu")
set(PACKAGE_URL "ebusd@ebusd.eu")
set(PACKAGE_PIDFILE "${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/run/${PACKAGE}.pid")
set(PACKAGE_LOGFILE "${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/log/${PACKAGE}.log")
set(PACKAGE_CONFIGPATH "${CMAKE_INSTALL_FULL_SYSCONFDIR}/${PACKAGE}")
execute_process(COMMAND sed -e "s/^\\([0-9]*\\.[0-9]*\\).*/\\1/" -e "s/\\.\\([0-9]\\)\$/0\\1/" -e "s/\\.//" VERSION
                OUTPUT_VARIABLE SCAN_VERSION)
execute_process(COMMAND git describe --always
                OUTPUT_VARIABLE REVISION
                OUTPUT_STRIP_TRAILING_WHITESPACE)
if(NOT REVISION)
  execute_process(COMMAND date +p%Y%m%d
                  OUTPUT_VARIABLE REVISION
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
endif(NOT REVISION)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_INSTALL_PREFIX "/")
include(GNUInstallDirs)
include(CheckFunctionExists)
include(CheckCXXSourceRuns)
include(CheckIncludeFile)

add_compile_options(-fpic -Wall -Wextra)

check_include_file(arpa/inet.h HAVE_ARPA_INET_H)
check_include_file(dirent.h HAVE_DIRENT_H)
check_include_file(fcntl.h HAVE_FCNTL_H)
check_include_file(netdb.h HAVE_NETDB_H)
check_include_file(poll.h HAVE_POLL_H)
check_include_file(pthread.h HAVE_PTHREAD_H)
check_include_file(sys/ioctl.h HAVE_SYS_IOCTL_H)
check_include_file(sys/select.h HAVE_SYS_SELECT_H)
check_include_file(sys/time.h HAVE_SYS_TIME_H)
check_include_file(time.h HAVE_TIME_H)
check_include_file(termios.h HAVE_TERMIOS_H)

check_function_exists(pthread_setname_np HAVE_PTHREAD_SETNAME_NP)
find_library(RTLIB rt)

check_function_exists(pselect HAVE_PSELECT)
check_function_exists(ppoll HAVE_PPOLL)

option(coverage "enable code coverage tracking." OFF)
if(NOT coverage STREQUAL OFF)
  add_compile_options(-coverage -O0)
  message(STATUS "coverage enabled")
endif(NOT coverage STREQUAL OFF)
option(contrib "disable inclusion of contributed sources." ON)
if(contrib STREQUAL ON)
  set(HAVE_CONTRIB ON)
  message(STATUS "contrib enabled")
endif(contrib STREQUAL ON)
find_library(HAVE_MQTT mosquitto)
if(HAVE_MQTT)
  option(mqtt "disable support for MQTT handling." ON)
  if(mqtt STREQUAL ON)
    message(STATUS "MQTT enabled")
  else(mqtt STREQUAL ON)
    unset(HAVE_MQTT)
  endif(mqtt STREQUAL ON)
endif(HAVE_MQTT) 

check_cxx_source_runs("
#include <stdint.h>
int main() {
  union {
    uint32_t i;
    float f;
  } test;
  test.f = 0.15;
  return test.i == 0x3e19999a ? 0 : 1;
}
" HAVE_DIRECT_FLOAT_FORMAT)
if(NOT HAVE_DIRECT_FLOAT_FORMAT)
  check_cxx_source_runs("
#include <stdint.h>
int main() {
  union {
    uint32_t i;
    float f;
  } test;
  test.f = 0.15;
  return test.i == 0x9a99193e ? 0 : 1;
}
  " HAVE_DIRECT_FLOAT_FORMAT_REV)
  if(HAVE_DIRECT_FLOAT_FORMAT_REV)
    set(HAVE_DIRECT_FLOAT_FORMAT 2)
  endif(HAVE_DIRECT_FLOAT_FORMAT_REV)
endif(NOT HAVE_DIRECT_FLOAT_FORMAT)

add_definitions(-DHAVE_CONFIG_H -DSYSCONFDIR="${CMAKE_INSTALL_FULL_SYSCONFDIR}" -DLOCALSTATEDIR="${CMAKE_INSTALL_FULL_LOCALSTATEDIR}")
configure_file(config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

add_subdirectory(src/ebusd)
add_subdirectory(src/lib/utils)
add_subdirectory(src/lib/ebus)
add_subdirectory(src/tools)