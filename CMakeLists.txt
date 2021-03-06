project(ayatana-indicator-power C CXX)
cmake_minimum_required(VERSION 3.13)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    SET(CMAKE_INSTALL_PREFIX "/usr" CACHE PATH "..." FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

set(PROJECT_VERSION "2.1.2")
set(PACKAGE ${CMAKE_PROJECT_NAME})

# Options
option(ENABLE_TESTS "Enable all tests and checks" OFF)
option(ENABLE_COVERAGE "Enable coverage reports (includes enabling all tests and checks)" OFF)

if(ENABLE_COVERAGE)
    set(ENABLE_TESTS ON)
    set(CMAKE_BUILD_TYPE "Coverage")
else()
    set(CMAKE_BUILD_TYPE "Release")
endif()

##
##  GNU standard installation directories
##

include (GNUInstallDirs)
set (CMAKE_INSTALL_PKGLIBEXECDIR "${CMAKE_INSTALL_LIBEXECDIR}/${CMAKE_PROJECT_NAME}")
set (CMAKE_INSTALL_FULL_PKGLIBEXECDIR "${CMAKE_INSTALL_FULL_LIBEXECDIR}/${CMAKE_PROJECT_NAME}")

##
## Gettext
##

set(GETTEXT_PACKAGE "ayatana-indicator-power")
add_definitions (-DGETTEXT_PACKAGE="${GETTEXT_PACKAGE}"
                 -DLOCALEDIR="${CMAKE_INSTALL_FULL_LOCALEDIR}")

##
##  Check for prerequisites
##

find_package (PkgConfig REQUIRED)
include (CheckIncludeFile)
include (FindPkgConfig)

pkg_check_modules(SERVICE_DEPS REQUIRED
                  glib-2.0>=2.36
                  gio-2.0>=2.36
                  gio-unix-2.0>=2.36
                  libnotify>=0.7.6
                  libayatana-common>=0.9.1)

include_directories (SYSTEM ${SERVICE_DEPS_INCLUDE_DIRS})

set(URL_DISPATCHER_REQUIRED_VERSION 0)
pkg_check_modules(
  URLDISPATCHER
  lomiri-url-dispatcher>=${URL_DISPATCHER_REQUIRED_VERSION}
)
include_directories(${URLDISPATCHER_INCLUDE_DIRS})

##
##  custom targets
##

set (ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${PROJECT_VERSION})
add_custom_target (dist
                   COMMAND bzr export --root=${ARCHIVE_NAME} ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.gz
                   WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

add_custom_target (cppcheck COMMAND cppcheck --enable=all -q --error-exitcode=2 --inline-suppr
                   ${CMAKE_SOURCE_DIR}/src
                   ${CMAKE_SOURCE_DIR}/tests)

##
##  Actual building
##

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(C_WARNING_ARGS "${C_WARNING_ARGS} -Weverything")
  set(C_WARNING_ARGS "${C_WARNING_ARGS} -Wno-c++98-compat -Wno-padded") # these are annoying
  set(C_WARNING_ARGS "${C_WARNING_ARGS} -Wno-documentation") # gtk-doc != doxygen
else()
  set(C_WARNING_ARGS "${C_WARNING_ARGS} -Wall -Wformat=2") # Use -Wextra and -Wpedantic explicitly if needed
endif()
set(C_WARNING_ARGS "${C_WARNING_ARGS} -Wno-missing-field-initializers") # GActionEntry


include_directories (${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories (${CMAKE_CURRENT_BINARY_DIR}/include)

# actually build things
add_subdirectory(src)
add_subdirectory(data)
add_subdirectory(po)


# testing & coverage
if (ENABLE_TESTS)
    enable_testing ()
    add_subdirectory(tests)
    if (ENABLE_COVERAGE)
        find_package(CoverageReport)
        ENABLE_COVERAGE_REPORT(
            TARGETS ayatanaindicatorpowerservice ayatana-indicator-power-service
            TESTS ${COVERAGE_TEST_TARGETS}
            FILTER /usr/include ${CMAKE_BINARY_DIR}/*
        )
    endif ()
endif ()

# Display config info

message(STATUS "Install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "Unit tests: ${ENABLE_TESTS}")
