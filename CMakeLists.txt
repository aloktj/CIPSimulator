cmake_minimum_required(VERSION 3.16)
project(CIPSimulator VERSION 0.1.0 LANGUAGES C CXX)

# --------- Toolchain / standards ----------
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# --------- Your sources (placeholder if empty) ----------
file(GLOB_RECURSE CIPSIM_SRC CONFIGURE_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp
)
if(CIPSIM_SRC STREQUAL "")
  message(WARNING "No sources found in ${CMAKE_CURRENT_SOURCE_DIR}/src — creating a temporary placeholder main.")
  set(_AUTOGEN_MAIN ${CMAKE_CURRENT_BINARY_DIR}/autogen_main.c)
  file(WRITE ${_AUTOGEN_MAIN}
"#include <stdio.h>\nint main(void){ puts(\"CIPSimulator placeholder. Add sources under ./src/\"); return 0; }\n")
  add_executable(CIPSimulatorExe ${_AUTOGEN_MAIN})
else()
  add_executable(CIPSimulatorExe ${CIPSIM_SRC})
endif()

target_include_directories(CIPSimulatorExe PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
if (CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
  target_compile_options(CIPSimulatorExe PRIVATE -Wall -Wextra -Wpedantic)
endif()

# --------- Build OpENer (POSIX) same as setup_posix.sh ----------
include(ExternalProject)
set(OPENER_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/import/OpENer/source")
set(OPENER_BIN_DIR "${CMAKE_CURRENT_BINARY_DIR}/opener_posix")   # build folder for OpENer

ExternalProject_Add(opener_posix
  SOURCE_DIR        "${OPENER_SRC_DIR}"
  BINARY_DIR        "${OPENER_BIN_DIR}"
  CMAKE_GENERATOR   "${CMAKE_GENERATOR}"
  CMAKE_ARGS
    -DCMAKE_C_COMPILER=gcc
    -DOpENer_PLATFORM:STRING=POSIX
    -DCMAKE_BUILD_TYPE:STRING=
    -DBUILD_SHARED_LIBS:BOOL=OFF
    -DOPENER_INSTALL_AS_LIB:BOOL=ON
  BUILD_ALWAYS      OFF
  INSTALL_COMMAND   ""     # no install
  TEST_COMMAND      ""     # skip tests
)

# --------- Stage OpENer libs into known paths with explicit build rules ----------
set(OPENER_STAGE_DIR "${CMAKE_CURRENT_BINARY_DIR}/opener_stage")
set(OPENER_STAGE_LIB "${OPENER_STAGE_DIR}/lib")

# OpENer libs produced by the external build (adjust if their names ever change)
set(_OPENER_SRC_LIBS
  "${OPENER_BIN_DIR}/src/enet_encap/libENET_ENCAP.a"
  "${OPENER_BIN_DIR}/src/cip/libCIP.a"
  "${OPENER_BIN_DIR}/src/utils/libUtils.a"
  "${OPENER_BIN_DIR}/src/ports/libPLATFORM_GENERIC.a"
  "${OPENER_BIN_DIR}/src/ports/POSIX/libPOSIXPLATFORM.a"
  "${OPENER_BIN_DIR}/src/ports/nvdata/libNVDATA.a"
)

# Staged outputs we will actually link against
set(_OPENER_STAGED_LIBS)
foreach(_lib IN LISTS _OPENER_SRC_LIBS)
  get_filename_component(_name "${_lib}" NAME)
  list(APPEND _OPENER_STAGED_LIBS "${OPENER_STAGE_LIB}/${_name}")
endforeach()

# Custom rule: copy the libs after the external project builds them
add_custom_command(
  OUTPUT ${_OPENER_STAGED_LIBS}
  COMMAND ${CMAKE_COMMAND} -E make_directory "${OPENER_STAGE_LIB}"
  # copy all libs (copy_if_different avoids needless relinks)
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${_OPENER_SRC_LIBS} "${OPENER_STAGE_LIB}/"
  DEPENDS opener_posix            # ensure OpENer is built first
  COMMENT "Staging OpENer static libraries into ${OPENER_STAGE_LIB}"
)

# A target that represents the staged libs
add_custom_target(opener_stage ALL DEPENDS ${_OPENER_STAGED_LIBS})

# Ensure your exe waits for staged libs to exist
add_dependencies(CIPSimulatorExe opener_stage)

# --------- Link against the staged libs ----------
target_link_libraries(CIPSimulatorExe PRIVATE
  ${_OPENER_STAGED_LIBS}
)

# Headers from OpENer (so you can include their APIs)
target_include_directories(CIPSimulatorExe PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/import/OpENer/source/src
)

# libcap is commonly needed on POSIX builds
find_library(LIBCAP cap)
if(LIBCAP)
  target_link_libraries(CIPSimulatorExe PRIVATE ${LIBCAP})
endif()

message(STATUS "OpENer will be configured in: ${OPENER_BIN_DIR}")
message(STATUS "OpENer source used:        ${OPENER_SRC_DIR}")
message(STATUS "OpENer libs staged to:     ${OPENER_STAGE_LIB}")
