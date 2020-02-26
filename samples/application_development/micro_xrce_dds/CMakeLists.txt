# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.13.1)
include($ENV{ZEPHYR_BASE}/cmake/app/boilerplate.cmake NO_POLICY_SCOPE)
project(micro_xrce_dds)

target_sources(app PRIVATE src/main.c src/HelloWorld.c src/HelloWorld.h)

# The external static library that we are linking with does not know
# how to build for this platform so we export all the flags used in
# this zephyr build to the external build system.
#
# Other external build systems may be self-contained enough that they
# do not need any build information from zephyr. Or they may be
# incompatible with certain zephyr options and need them to be
# filtered out.
# zephyr_get_include_directories_for_lang_as_string(       C includes)
# zephyr_get_system_include_directories_for_lang_as_string(C system_includes)
# zephyr_get_compile_definitions_for_lang_as_string(       C definitions)
# zephyr_get_compile_options_for_lang_as_string(           C options)

# set(external_project_cflags
#   "${includes} ${definitions} ${options} ${system_includes}"
#   )

include(FetchContent)

set(UCLIENT_SUPERBUILD OFF CACHE BOOL "" FORCE)
set(EXTERNAL_TRANSPORT_HEADER ${CMAKE_CURRENT_SOURCE_DIR}/transport/zephyr_serial_transport.h CACHE PATH "" FORCE)
set(EXTERNAL_TRANSPORT_SRC ${CMAKE_CURRENT_SOURCE_DIR}/transport/zephyr_serial_transport.c CACHE PATH "" FORCE)

FetchContent_Declare(
  microcdr
  GIT_REPOSITORY https://github.com/eProsima/Micro-CDR
  GIT_TAG        dashing
)

FetchContent_Declare(
  microxrcedds
  GIT_REPOSITORY https://github.com/eProsima/Micro-XRCE-DDS-Client.git
  GIT_TAG        dashing
)

FetchContent_GetProperties(microcdr)
if(NOT microcdr_POPULATED)
  FetchContent_Populate(microcdr)
  add_subdirectory(${microcdr_SOURCE_DIR} ${microcdr_BINARY_DIR})
endif()

find_package(microcdr REQUIRED)

target_link_libraries(app PUBLIC microcdr)

FetchContent_GetProperties(microxrcedds)
if(NOT microxrcedds_POPULATED)
  FetchContent_Populate(microxrcedds)
  add_subdirectory(${microxrcedds_SOURCE_DIR} ${microxrcedds_BINARY_DIR})
endif()

find_package(microxrcedds_client REQUIRED)

target_link_libraries(app PUBLIC microxrcedds_client)