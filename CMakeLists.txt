cmake_minimum_required(VERSION 3.20)

set(DEFAULT_BUILD_TYPE "Release")

if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' as none was specified.")
    set(CMAKE_BUILD_TYPE  "${DEFAULT_BUILD_TYPE}")
endif()

if(NOT MCU OR NOT F_CPU)
    message(FATAL_ERROR "MCU or F_CPU is not defined, please define by 'cmake -DMCU=<mcu> -DF_CPU=<clock frequency> ..'")
endif()

set(DEVICE_PACK_DIR ${CMAKE_CURRENT_SOURCE_DIR}/device_pack)

if(NOT EXISTS ${DEVICE_PACK_DIR})
    message(FATAL_ERROR "Device pack not found in root of project directory. The device pack (ATPACK/DFP) should be in a folder named 'device_pack'.")
endif()



# --------------------------------- Toolchain ---------------------------------

find_program(AVR_CC avr-gcc REQUIRED)
find_program(AVR_CXX avr-g++ REQUIRED)
find_program(AVR_OBJCOPY avr-objcopy REQUIRED)
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

# Helper variables for the device related directories in the ATPACK and the 
# toolchain
set(MCU_DEVICE_DIRECTORY ${DEVICE_PACK_DIR}/gcc/dev/${MCU})
set(MCU_INCLUDE_DIRECTORY ${DEVICE_PACK_DIR}/include)

if(NOT EXISTS ${MCU_DEVICE_DIRECTORY}/device-specs/specs-${MCU})
    message(FATAL_ERROR "Could not find device pack for ${MCU}, tried looking in directory ${MCU_DEVICE_DIRECTORY}/device-specs/specs-${MCU}. The device name might be misspelled or the device pack might be incorrect for this device.")
endif()

# Find the device library name by going through the device specs for the device 
file(READ ${MCU_DEVICE_DIRECTORY}/device-specs/specs-${MCU} SPECS_FILE_CONTENT)
string(
    REGEX MATCH 
    "-D__AVR_DEVICE_NAME__=${MCU} -D__AVR_DEV_LIB_NAME__=([a-zA-Z0-9]*)" 
    _ 
    ${SPECS_FILE_CONTENT}
)
set(MCU_DEV_LIB_NAME ${CMAKE_MATCH_1})



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
    -D__AVR_DEV_LIB_NAME__=${MCU_DEV_LIB_NAME}
)

set(ERROR_FLAGS 
    -Wall 
    -Wextra 
    -pedantic 
    -Wshadow 
    -Wno-array-bounds
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

    # Notify the compiler about the device specs
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

    # Notify the compiler about the device specs
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
    COMMAND ${PYMCUPROG} write -d ${MCU} -f ${TARGET}.hex --erase
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
