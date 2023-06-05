# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2019-2022 Second State INC

# Try to get libbpf use the following order
# - PkgConfig
# - ${LIBBPF_SOURCE_DIR}
# - FetchContent

option(WASMEDGE_PLUGIN_WASM_BPF_BUILD_LIBBPF_WITH_PKG_CONF "Build libbpf without pkg-config. Requires libelf and libz being placed at correct place." YES)

message(STATUS "Trying to get libbpf..")
message(STATUS "Build libbpf with pkg-config: ${WASMEDGE_PLUGIN_WASM_BPF_BUILD_LIBBPF_WITH_PKG_CONF}")
set(LIBBPF_FOUND FALSE)

# A wrapper function to add libbpf located at a local path as a dependency
function(AddLibbpfAsExternal SOURCE_ROOT WITH_PKG_CONF)
  include(ExternalProject)
  set(LIBBPF_SO_PATH ${SOURCE_ROOT}/src/build/libbpf.so)
  set(LIBBPF_INCLUDE_DIRS_LOCAL "${SOURCE_ROOT}/src/root/usr/include" "${SOURCE_ROOT}/include/uapi" "${SOURCE_ROOT}/include")
  set(LIBBPF_INCLUDE_DIRS ${LIBBPF_INCLUDE_DIRS_LOCAL} PARENT_SCOPE)

  set(LIBBPF_LIBRARIES ${LIBBPF_SO_PATH} PARENT_SCOPE)
  set(LIBBPF_LIBRARIES_STATIC ${SOURCE_ROOT}/src/build/libbpf.a PARENT_SCOPE)

  if(${WITH_PKG_CONF})
    set(PKGCONF_PREFIX "")
  else()
    set(PKGCONF_PREFIX "NO_PKG_CONFIG=1")
  endif()

  ExternalProject_Add(libbpf
    PREFIX libbpf
    SOURCE_DIR ${SOURCE_ROOT}
    CONFIGURE_COMMAND "mkdir" "build" "root"
    BUILD_COMMAND "${PKGCONF_PREFIX}" "OBJDIR=${SOURCE_ROOT}/src/build" "DESTDIR=${SOURCE_ROOT}/src/root" "make" "-C" "${SOURCE_ROOT}/src" "install"
    INSTALL_COMMAND "cp" "${LIBBPF_SO_PATH}" "${CMAKE_CURRENT_BINARY_DIR}/libbpf.so"
    BUILD_IN_SOURCE TRUE
  )

  set(LIBBPF_TARGET_NAME libbpf PARENT_SCOPE)
endfunction()

# Try PkgConfig
if(NOT ${LIBBPF_FOUND})
  find_package(PkgConfig)

  if(PkgConfig_FOUND)
    message(STATUS "Try to get libbpf through PkgConfig")

    # It will set LIBBPF_FOUND for us
    pkg_check_modules(LIBBPF libbpf>=1111.2.0 IMPORTED_TARGET)
    set(LIBBPF_TARGET_NAME "PkgConfig::LIBBPF")
    message(STATUS "LIBBPF_FOUND=${LIBBPF_FOUND}")

    if(${LIBBPF_FOUND})
      SET(LIBBPF_FOUND TRUE)
    else()
      SET(LIBBPF_FOUND FALSE)
    endif()

    if(${LIBBPF_FOUND})
      message(STATUS "libbpf found using PkgConfig")
    else()
      message(STATUS "libbpf not found using pkgconfig")
    endif()
  else()
    message(STATUS "PkgConfig not found")
  endif()
endif()

# Try LIBBPF_SOURCE_DIR
if(NOT ${LIBBPF_FOUND})
  message(STATUS "Try to get libbpf through the pre-defined LIBBPF_SOURCE_DIR")

  if(DEFINED LIBBPF_SOURCE_DIR)
    AddLibbpfAsExternal(${LIBBPF_SOURCE_DIR} ${WASMEDGE_PLUGIN_WASM_BPF_BUILD_LIBBPF_WITH_PKG_CONF})
    set(LIBBPF_FOUND TRUE)
    message(STATUS "libbpf found using LIBBPF_SOURCE_DIR")
  else()
    message(STATUS "LIBBPF_SOURCE_DIR not defined")
  endif()
endif()

# Try FetchContent
if(NOT ${LIBBPF_FOUND})
  message(STATUS "Try to get libbpf through FetchContent")
  include(FetchContent)
  FetchContent_Declare(
    libbpf
    GIT_REPOSITORY https://github.com/libbpf/libbpf
    GIT_TAG cf46d44f0a06aa8b9400691ea3eb86ca4f066d5c
  )
  FetchContent_GetProperties(libbpf)

  if(NOT libbpf_POPULATED)
    message(STATUS "Fetching libbpf..")
    FetchContent_Populate(libbpf)
    message(STATUS "Fetched libbpf")
  endif()

  set(LIBBPF_DOWNLOAD_SOURCE_DIR "${libbpf_SOURCE_DIR}")
  message(DEBUG "libbpf saved at: ${LIBBPF_DOWNLOAD_SOURCE_DIR}")
  AddLibbpfAsExternal(${LIBBPF_DOWNLOAD_SOURCE_DIR} ${WASMEDGE_PLUGIN_WASM_BPF_BUILD_LIBBPF_WITH_PKG_CONF})
  set(LIBBPF_FOUND TRUE)
endif()

# If we cannot find libbpf..
if(NOT ${LIBBPF_FOUND})
  message(FATAL_ERROR "Could not find libbpf")
endif()

message(STATUS "LIBBPF_INCLUDE_DIRS=${LIBBPF_INCLUDE_DIRS}")
message(STATUS "LIBBPF_LIBRARIES=${LIBBPF_LIBRARIES}")
message(STATUS "LIBBPF_TARGET_NAME=${LIBBPF_TARGET_NAME}")

if(WASMEDGE_PLUGIN_WASM_BPF_BUILD_LIBBPF_WITH_PKG_CONF)
  # Find the dependencies `libelf` and `libz` of libbpf
  find_package(PkgConfig)

  pkg_check_modules(LIBBPF_DEP REQUIRED libelf zlib)

  message(DEBUG "LIBBPF_DEP_LIBRARIES=${LIBBPF_DEP_LIBRARIES}")
endif()

wasmedge_add_library(wasmedgePluginWasmBpf
  SHARED
  wasm-bpf-module.cpp
  func-load-bpf-object.cpp
  func-close-bpf-object.cpp
  func-attach-bpf-program.cpp
  func-bpf-buffer-poll.cpp
  func-bpf-map-fd-by-name.cpp
  func-bpf-map-operate.cpp
  wasm-bpf.cpp
  util.cpp
)

add_dependencies(wasmedgePluginWasmBpf ${LIBBPF_TARGET_NAME})
target_link_libraries(wasmedgePluginWasmBpf PUBLIC ${LIBBPF_LIBRARIES} ${LIBBPF_DEP_LIBRARIES})
target_include_directories(wasmedgePluginWasmBpf PUBLIC ${LIBBPF_INCLUDE_DIRS})

set_target_properties(wasmedgePluginWasmBpf PROPERTIES
  CXX_STANDARD 17
)

target_compile_options(wasmedgePluginWasmBpf
  PUBLIC
  -DWASMEDGE_PLUGIN
  -fPIC
)

target_include_directories(wasmedgePluginWasmBpf
  PUBLIC
  $<TARGET_PROPERTY:wasmedgePlugin,INCLUDE_DIRECTORIES>
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${LIBBPF_INCLUDE_DIRS}
)

if(WASMEDGE_LINK_PLUGINS_STATIC)
  target_link_libraries(wasmedgePluginWasmBpf
    PRIVATE
    wasmedgeCAPI
    ${LIBBPF_LIBRARIES}
  )
else()
  target_link_libraries(wasmedgePluginWasmBpf
    PRIVATE
    wasmedge_shared
    ${LIBBPF_LIBRARIES}
  )
endif()
