---
layout: page
title: Documentation
permalink: /doc/
---

# Schematics
<!-- Include images of the schematics for your system. They should follow best practices for schematic drawings with all parts and pins clearly labeled. You may draw your schematics either with a software tool or neatly by hand. -->

<div style="text-align: center">
  <img src="../assets/img/wiring.png" alt="wiring" width="700" />
</div>


# Source Code Overview
<!-- This section should include information to describe the organization of the code base and highlight how the code connects. -->
The code base for this project is broken into three sections: 

* MCU code, written in C
* FPGA code, written in SystemVerilog
* FPGA testbenches, also written in SystemVerilog

The source code for the project is located in the Github repository [here](https://github.com/Joseph-Q-Zales/HMC-MicroPs-Final-Portfolio/tree/main/src).

## MCU Code
Our microcontroller code includes a multiple library files,  using the CMCIS structures, and a `main.c` file in which all of the library files are included. The library files are named `STM32L432KC_x.c` where x is the name of the peripheral driven by that library. For example, the I2C is used to communicate with the PN532 RFID/NCF Controller. The I2C driver is named `STM32L432KC_I2C.c`.

The `main.c` file calls functions from the various libraries and allows the MCU to act as the controller for both I2C (to the PN532) and SPI (to the FPGA).

### MCU Communication
The Nucleo microcontroller is at the heart of the project. It communicates via I2C with the PN532 NFC/RFID contoller using a set of multi-byte commands. See section **TODO** for details. It also communicates via one-directional SPI to the FPGA. 

From the RFID reader, the MCU extracts the UID from the RFID card, separates it into the bytes needed by the FPGA and sends 5 bytes of data, via SPI, to the FPGA.

## FPGA Code
The code to program the FPGA resides in one SystemVerilog file titled `main.sv`. The first module in this file, top, controlls the FGPA and calls the other modules as needed. 

The module entitled `tune` is the main finite state machine (FSM) for the FPGA. See section **TODO** for details. Below is the full netlist (all of the connections between modules) on the FPGA.


<div style="text-align: center">
  <img src="../assets/img/netListAnalyzer.png" alt="netlist" width="1200" />
</div>
FPGA Netlist.

## FPGA Testbenches
A series of testbenches were created to test the FPGA prior to the integration with the MCU. 

Test benches created

* Testing SPI
* Testing the main FSM
* Testing frequency output
* Testing duration outputs



# Bill of Materials
<!-- The bill of materials should include all the parts used in your project along with the prices and links.  -->

| Item | Purpose | Part Number | Quantity | Unit Price | Vendor/Source |
| ---- | ------- | ----------| ----- | ---- | ---- |
| Adafruit PN532 NFC/RFID Controller / breakout board | RFID Peripheral | 364 | 1 | $39.95 |  [Adafruit](https://www.adafruit.com/product/364) |
| Nucleo STM32L432KC Microcontroller | Interface between RFID peripheral and FPGA, runs algorithm to determine doorbell | NUCLEO-L432KC | 1 | | Lab kit |
| Upduino ICE40 & UP5K FPGA | Creates PWM signal to drive speaker based on RFID ID# | N/A | 1 | | lab kit |
| 8Ω speaker | Doorbell output speaker | N/A | 1 | | Stock room |
| Resistor, 4.7kΩ | Pull-up resistor for I2C connections | N/A | 2 | | Stock room |
| Resistor, 15kΩ | Speaker volume control | N/A | 1 | | Stock room |
| Resistor, 2.7kΩ | Speaker volume control | N/A | 1 | | Stock room |
| LM386 Audio amplifier | Audio amplifier | LM386N-4/NOPB | 1 | | Stock room|
| Slide switch | On/off switch for doorbell speaker | N/A | 1 | | Stock room |
| Wire, various length | Connection between electronics | N/A | variable | | Stock room |

**Total cost: $39.95**
