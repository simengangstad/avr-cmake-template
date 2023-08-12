# AVR CMake Template Project

This template project serves as a starting point for using an AVR toolchain with CMake. 

## Setup

1. In order to use the template, some variables have to be defined in the `CMakeList.txt`:

```
set(TARGET <your target/project name>)

set(MCU <mmcu variable, e.g. atmega4809>)
set(F_CPU <CPU frequency>)

set(AVRDUDE_MCU <corresponding MCU identifier for avrdude, e.g. m4809>)
set(AVRDUDE_PROGRAMMER_ID <your programmer identifier, e.g. pkobn_updi>)

# Optional: specify the desired language standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 20)
```

2. Thereafter, the toolchain can be included with: `include(cmake/Toolchain.cmake)`

3. The project needs to be defined with the function `add_avr_executable(${TARGET} <your sources>)`.

Please refer to the example project for the full outline of the `CMakeLists.txt`. Note that for newer devices, a device pack might be needed. The boolean `TOOLCHAIN_NEED_DEVICE_PACK` specifies this. If this variable is set to true, `Toolchain.cmake`will look for a `device_pack` folder at the root of the project and set up the necessary compiler flags for using the device pack with the given MCU specified.

## Defined targets

`Toolchain.cmake` defines the following convenience targets in addition to default make targets:

- `make upload`: Upload using avrdude.
- `make upload_eeprom`: Upload eeprom using avrdude.
- `make disassemble`: Disassemble the .elf file.

## Use

Once the `CMakeLists.txt` has been defined, the project can be compiled and upload by issuing:

- `mkdir build && cd build`
- `cmake ..`
- `make -j`
- `make upload`
