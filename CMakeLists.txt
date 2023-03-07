cmake_minimum_required(VERSION 3.20)

set(DEFAULT_BUILD_TYPE "Release")

if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' as none was specified.")
    set(CMAKE_BUILD_TYPE  "${DEFAULT_BUILD_TYPE}")
endif()

if(NOT ATPACK_DIR)
    message(FATAL_ERROR "ATPACK_DIR is not defined, please define the path to the ATPACK directory by issuing 'cmake -DATPACK_DIR=<dir> ..'")
endif()

if(NOT MCU)
    message(FATAL_ERROR "MCU is not defined, please define by 'cmake -DMCU=<mcu> ..'")
endif()

if(NOT F_CPU)
    message(FATAL_ERROR "F_CPU is not defined, please define clock frequency by 'cmake -DF_CPU=<clock frequency> ..'")
endif()


set(CMAKE_VERBOSE_MAKEFILE ON)


# --------------------------------- Toolchain ---------------------------------

find_program(AVR_CC avr-gcc REQUIRED)
find_program(AVR_CXX avr-g++ REQUIRED)
find_program(AVR_OBJCOPY avr-objcopy REQUIRED)
find_program(AVR_SIZE_TOOL avr-size REQUIRED)
find_program(AVR_OBJDUMP avr-objdump REQUIRED)
find_program(PYMCUPROG pymcuprog REQUIRED)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR avr)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_C_COMPILER ${AVR_CC})
set(CMAKE_CXX_COMPILER ${AVR_CXX})
set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED TRUE)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

enable_language(C CXX ASM)

# Helper variables for the device related directories in the ATPACK
set(MCU_DEVICE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/${ATPACK_DIR}/gcc/dev/${MCU})
set(MCU_INCLUDE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/${ATPACK_DIR}/include)

# ----------------------------------- Target -----------------------------------

project(avr)

set(TARGET main.elf)

file(GLOB SOURCES 
    src/*.c
    src/*.cpp
)

set(INCLUDE_DIRECTORIES
    src
    include
)



# ------------------- Definitions and options for compiler ---------------------

set(COMPILE_DEFINITIONS
    # Define the clock speed of the MCU 
    -DF_CPU=${F_CPU}
)

set(ERROR_FLAGS 
    -Wall 
    -Wundef
    -Wextra 
    -Wpedantic 
    -Wshadow 
    -Wno-vla 
)

set(COMPILE_OPTIONS
    -mmcu=${MCU}

    ${ERROR_FLAGS}

    -funsigned-char 
    -funsigned-bitfields
    -fpack-struct
    -fshort-enums
    -ffunction-sections
    -fdata-sections
    -fno-split-wide-types
    -fno-tree-scev-cprop

    # Include the AVR header files from the ATPACK
    -I${MCU_INCLUDE_DIRECTORY} 
    -B${MCU_DEVICE_DIRECTORY}

    # Vary optimization and debug level based on if
    # CMake is configured for release or debug 
    # (builds for e.g. debug with 
    #  cmake -DCMAKE_BUILD_TYPE=Debug ..)
    $<$<CONFIG:Debug>:-O0 -g3>
    $<$<CONFIG:Release>:-O3>
)



# ----------------------------- Options for linker ----------------------------

set(LINK_OPTIONS 
    -mmcu=${MCU}

    -Wl,-Map=${TARGET}.map

    -Wl,--print-memory-usage
    -Wl,--gc-section
    -Wl,--sort-section=alignment
    -Wl,--cref

    -B${MCU_DEVICE_DIRECTORY}
)


# -------------------------------- Executable --------------------------------

add_executable(${TARGET} ${SOURCES})

target_include_directories(${TARGET} PRIVATE ${INCLUDE_DIRECTORIES})
target_compile_definitions(${TARGET} PRIVATE ${COMPILE_DEFINITIONS})
target_compile_options(${TARGET} PRIVATE ${COMPILE_OPTIONS})
target_link_options(${TARGET} PRIVATE ${LINK_OPTIONS})


# Add command to convert the elf file to a hex file after compiling 
add_custom_command(TARGET ${TARGET}
    POST_BUILD
    COMMAND ${AVR_OBJCOPY} -R .eeprom -R .fuse -R .lock -R .signature -O ihex ${TARGET} ${TARGET}.hex
)



# ----------------------------------- Flashing -------------------------------

add_custom_target(flash
    DEPENDS ${TARGET} 
    COMMAND ${PYMCUPROG} write -d {MCU} -f {TARGET}.hex --erase
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
