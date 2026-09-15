cmake_minimum_required(VERSION 3.16)

project(fastacl-plugin)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

find_package(VPP REQUIRED)

include_directories(${CMAKE_SOURCE_DIR}/src/plugins)

add_subdirectory(src/plugins/fastacl)
