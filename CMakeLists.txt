cmake_minimum_required(VERSION 2.8)

project("kms-jsonrpc")
message(STATUS "CMake project: ${PROJECT_NAME}")

# Test configuration
set(GENERATE_TESTS FALSE CACHE BOOL "Always build tests: add `make check_build` to normal `make` calls")
set(DISABLE_TESTS FALSE CACHE BOOL "Enable running `make check` during the building process")
set(VALGRIND_NUM_CALLERS 20 CACHE STRING "Valgrind option: maximum number of entries shown in stack traces")

message("If KurentoHelpers is not found, you need to install 'kms-cmake-utils' from the Kurento repository")
find_package(KurentoHelpers REQUIRED)

include(KurentoGitHelpers)
install_git_hook(pre-commit ${CMAKE_CURRENT_SOURCE_DIR}/hooks/pre-commit.hook)
get_git_version(PROJECT_VERSION 1.2.0-dev)
message(STATUS "Project version: ${PROJECT_NAME}-${PROJECT_VERSION}")

# Set GCC flags common to all projects
# Note: Flags added from dpkg-buildflags (if installed):
# -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2
include(DpkgBuildFlags)
get_dpkg_buildflags_c(DPKG_BUILDFLAGS_C)
get_dpkg_buildflags_cxx(DPKG_BUILDFLAGS_CXX)
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   ${DPKG_BUILDFLAGS_C}   -std=c11   -Wall -Werror -pthread")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${DPKG_BUILDFLAGS_CXX} -std=c++11 -Wall -Werror -pthread")
set(CMAKE_C_FLAGS_DEBUG   "-g -O0") # FIXME Ideal is '-Og' but a bug in GCC prevents this,
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0") # causing "may be used uninitialized" errors: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=58455
set(CMAKE_C_FLAGS_RELEASE   "-DNDEBUG -O2") # CMake adds '-O3' by default
set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -O2") # but currently only '-O2' is tested
set(CMAKE_POSITION_INDEPENDENT_CODE ON) # Use "-fPIC" for all targets, including static libs

find_package(PkgConfig)
pkg_check_modules(JSONCPP REQUIRED kmsjsoncpp>=0.6.0)

find_package(Boost REQUIRED COMPONENTS unit_test_framework)

include(GNUInstallDirs)
set(CUSTOM_PREFIX "kurento")
set(INCLUDE_PREFIX "${CMAKE_INSTALL_INCLUDEDIR}/${CUSTOM_PREFIX}/jsonrpc")

if(NOT DEFINED SYSCONF_INSTALL_DIR)
  set(SYSCONF_INSTALL_DIR etc CACHE PATH
      "Install directory for system-wide configuration files")
endif(NOT DEFINED SYSCONF_INSTALL_DIR)

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}/${CUSTOM_PREFIX}")
set(requires "kmsjsoncpp")

configure_file(libjsonrpc.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libjsonrpc.pc @ONLY)
configure_file(FindKmsJsonRpc.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/FindKmsJsonRpc.cmake @ONLY)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/libjsonrpc.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  )

set(CMAKE_MODULES_INSTALL_DIR
  ${CMAKE_INSTALL_DATAROOTDIR}/cmake-${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}/Modules
  CACHE STRING
  "Destination (relative to CMAKE_INSTALL_PREFIX) for cmake modules files"
)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/FindKmsJsonRpc.cmake
  DESTINATION ${CMAKE_MODULES_INSTALL_DIR}
)

enable_testing()

add_subdirectory(src)
add_subdirectory(tests)
