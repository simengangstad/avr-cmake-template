cmake_minimum_required(VERSION 3.20)

# ---------------------------- Target Configuration ----------------------------

# Specify the target/project name
set(TARGET test_project)

set(MCU atmega4809)
set(F_CPU 3333333)

set(AVRDUDE_MCU m4809)
set(AVRDUDE_PROGRAMMER_ID pkobn_updi)

# Specify the desired language standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 20)

# Include the toolchain. If a device pack is needed, keep this value to true.
# The device pack should be in a folder name 'device_pack'
set(TOOLCHAIN_NEED_DEVICE_PACK true)
include(../cmake/Toolchain.cmake)

# ----------------------------------- Target -----------------------------------

project(test_project)
file(GLOB SOURCES src/*.c src/*.cpp)

add_avr_executable(${TARGET} ${SOURCES})

# Specify own definitions, include directories and libraries here. The
# AVRToolChain.cmake file already specifies a bunch of standard compile options
# and definitions

# target_compile_definitions(${TARGET} ...)
target_include_directories(${TARGET} PRIVATE include src)
target_link_libraries(${TARGET} PRIVATE m)
