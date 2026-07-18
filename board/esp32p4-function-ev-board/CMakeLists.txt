# ##############################################################################
# vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/CMakeLists.txt
#
# Vendor board CMakeLists that references the NuttX board sources.
#
# ##############################################################################

# Include the NuttX board src
set(NUTTX_ESP32P4_BOARD_DIR ${NUTTX_DIR}/boards/risc-v/esp32p4/esp32p4-function-ev-board)

add_subdirectory(${NUTTX_ESP32P4_BOARD_DIR}/src
                 ${CMAKE_CURRENT_BINARY_DIR}/board_src)
