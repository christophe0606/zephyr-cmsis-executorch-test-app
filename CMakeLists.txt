# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.20.0)

# Local CMSIS-ExecuTorch module; west supplies the upstream source dependency.
list(APPEND EXTRA_ZEPHYR_MODULES
  "${CMAKE_CURRENT_SOURCE_DIR}/../../../CMSIS-Executorch"
)

list(APPEND EXTRA_CONF_FILE "${CMAKE_CURRENT_SOURCE_DIR}/operators.conf")

# Include the NPU node even when an older build cache has an empty overlay list.
if(NOT DTC_OVERLAY_FILE)
  set(DTC_OVERLAY_FILE "${CMAKE_CURRENT_SOURCE_DIR}/app.overlay")
endif()

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)

target_sources(app PRIVATE
  src/main.c
  src/executorch_test.cpp
  src/model_pte.c
  # The local module supplies the runtime/operators; this model also delegates
  # to the upstream Ethos-U backend through Zephyr's initialized core driver.
  "${ZEPHYR_EXECUTORCH_MODULE_DIR}/backends/arm/runtime/EthosUBackend.cpp"
  "${ZEPHYR_EXECUTORCH_MODULE_DIR}/backends/arm/runtime/EthosUBackend_Cortex_M.cpp"
  "${ZEPHYR_EXECUTORCH_MODULE_DIR}/backends/arm/runtime/EthosUBackend_IoMemcpy.cpp"
  "${ZEPHYR_EXECUTORCH_MODULE_DIR}/backends/arm/runtime/VelaBinStream.cpp"
)
