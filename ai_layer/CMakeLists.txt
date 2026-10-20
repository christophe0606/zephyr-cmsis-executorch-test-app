# SPDX-License-Identifier: Apache-2.0

# Include this directory before find_package(Zephyr): Kconfig is processed there.
if(TARGET zephyr_interface)
  message(FATAL_ERROR
    "add_subdirectory(ai_layer) must precede find_package(Zephyr) so the AI layer configuration is merged."
  )
endif()

# Select the model's shared components, logging policy, and CPU operators.
list(APPEND EXTRA_CONF_FILE
  "${CMAKE_CURRENT_SOURCE_DIR}/components.conf"
  "${CMAKE_CURRENT_SOURCE_DIR}/operators.conf"
)
set(EXTRA_CONF_FILE "${EXTRA_CONF_FILE}" PARENT_SCOPE)

# An interface target carries the model sources and headers into the app target.
# A generator can list additional model arrays here, each with unique C symbols.
add_library(ai_layer INTERFACE)
target_sources(ai_layer INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/model_pte.c"
)
target_include_directories(ai_layer INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}")

# Zephyr creates app during find_package(), after this directory is processed.
# Link at the end of the enclosing CMakeLists, keeping integration to one call.
get_directory_property(_ai_layer_parent PARENT_DIRECTORY)
cmake_language(DEFER DIRECTORY "${_ai_layer_parent}"
  CALL target_link_libraries app PRIVATE ai_layer
)
