# README
This readme contains an overview of the source code files contained in this folder.

The code base for this project is broken into three sections:

* MCU code, written in C
* FPGA code, written in SystemVerilog
* FPGA testbenches, also written in SystemVerilog

# MCU Code

Our microcontroller code includes a multiple library files, using the CMCIS structures, and a main.c file in which all of the library files are included. The library files are named STM32L432KC_x.c where x is the name of the peripheral driven by that library. For example, the I2C is used to communicate with the PN532 RFID/NCF Controller. The I2C driver is named STM32L432KC_I2C.c.

The main.c file calls functions from the various libraries and allows the MCU to act as the controller for both I2C (to the PN532) and SPI (to the FPGA).

# FPGA Code

The code to program the FPGA resides in one SystemVerilog file titled main.sv. The first module in this file, top, controls the FGPA and calls the other modules as needed.

The module entitled tune is the main finite state machine (FSM) for the FPGA.

# FPGA Testbenches

A series of testbenches were created to test the FPGA prior to the integration with the MCU.

These testbenches are listed as follows:

* SPI
* Main FSM (the tune module)
* Frequency generator
* Duration counter
* Decomposition of the MCU output
* Enables for the system
* Top module
