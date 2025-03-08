cmake_minimum_required(VERSION 3.20)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

project(usb-sha256 C CXX ASM)

find_package(wolfssl REQUIRED)

# paths and files

set(BOARD_DIR ${PROJECT_SOURCE_DIR}/board)
set(BOARD_SRC_DIR ${BOARD_DIR}/src)
set(BOARD_INC_DIR ${BOARD_DIR}/include)

set(USBSHA256_DIR ${PROJECT_SOURCE_DIR}/usb-sha256)
set(USBSHA256_SRC_DIR ${USBSHA256_DIR}/src)
set(USBSHA256_INC_DIR ${USBSHA256_DIR}/include)

set(LINKER_SCRIPT ${PROJECT_SOURCE_DIR}/linker/STM32F411CEUX_FLASH.ld)

include(${PROJECT_SOURCE_DIR}/cmake/stm32cubef4.cmake)

# startup library

add_subdirectory(${PROJECT_SOURCE_DIR}/startup)

# main ELF file

add_executable(main
    ${CMSIS_TEMPLATES_DIR}/system_stm32f4xx.c
    ${HAL_SRC_DIR}/stm32f4xx_hal.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_cortex.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_gpio.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_pcd.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_pcd_ex.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_rcc.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_tim.c
    ${HAL_SRC_DIR}/stm32f4xx_hal_tim_ex.c
    ${HAL_SRC_DIR}/stm32f4xx_ll_usb.c
    ${BOARD_SRC_DIR}/main.c
    ${USBSHA256_SRC_DIR}/sha256sum.cpp
    ${USBSHA256_SRC_DIR}/shell.cpp
    ${BOARD_SRC_DIR}/stm32f4xx_hal_msp.c
    ${BOARD_SRC_DIR}/stm32f4xx_it.c
    ${BOARD_SRC_DIR}/syscalls.c
    ${BOARD_SRC_DIR}/sysmem.c
    ${BOARD_SRC_DIR}/usbd_cdc_if.c
    ${BOARD_SRC_DIR}/usbd_conf.c
    ${BOARD_SRC_DIR}/usbd_desc.c
    ${BOARD_SRC_DIR}/usb_device.c
    ${USBDEVLIB_CDC_SRC_DIR}/usbd_cdc.c
    ${USBDEVLIB_SRC_DIR}/usbd_core.c
    ${USBDEVLIB_SRC_DIR}/usbd_ctlreq.c
    ${USBDEVLIB_SRC_DIR}/usbd_ioreq.c
)
set_target_properties(main PROPERTIES OUTPUT_NAME main.elf)
target_link_libraries(main PRIVATE startup wolfssl::wolfssl)
target_include_directories(main PRIVATE
    ${BOARD_INC_DIR}
    ${USBSHA256_INC_DIR}
    ${CMSIS_INCLUDE_DIRECTORIES}
    ${HAL_INCLUDE_DIRECTORIES}
    ${USBDEVLIB_INCLUDE_DIRECTORIES}
)
target_link_options(main PRIVATE
    -Wl,-T ${LINKER_SCRIPT}
    -Wl,--fatal-warnings
    -Wl,--gc-sections
    -nostartfiles
)
target_compile_definitions(main PRIVATE STM32F411xE)
target_compile_options(main PRIVATE
    -Wall
    -Wextra
    -Wpedantic
    -Werror
    $<$<COMPILE_LANGUAGE:CXX>:-fno-exceptions>
    -ffunction-sections
    -fdata-sections
)

# strip ELF file

find_program(CMAKE_STRIP arm-none-eabi-strip REQUIRED)
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/stripped.elf
    DEPENDS main
    COMMAND ${CMAKE_STRIP} main.elf -o stripped.elf
)
add_custom_target(stripped DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/stripped.elf)

# make firmware file

find_program(CMAKE_OBJCOPY arm-none-eabi-objcopy REQUIRED)
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/main.bin
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/stripped.elf
    COMMAND ${CMAKE_OBJCOPY} -O binary stripped.elf main.bin
)
add_custom_target(main_bin ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/main.bin)

# flash device

message(CHECK_START "Check whether st-flash is installed")
find_program(CMAKE_ST_FLASH st-flash)
if (CMAKE_ST_FLASH-NOTFOUND)
    message(CHECK_FAIL "not found")
else()
    message(CHECK_PASS "found")
    add_custom_target(
        flash
        COMMAND ${CMAKE_ST_FLASH} --connect-under-reset write main.bin 0x08000000
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/main.bin
    )
endif()
