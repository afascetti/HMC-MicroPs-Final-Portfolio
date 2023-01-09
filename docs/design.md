---
layout: page
title: Design
permalink: /design/
---

This system involves four independent systems working together: the RFID Reader, the MCU, the FPGA, and the speaker system. All must be designed independently, and then integrated together.

As the design of the speaker system is analog, it is not described in detail here. It simply involves a LM386 Audio Amplifier and an 8Ω speaker. The breadboard schematic of this system is shown in the [Documentation](https://joseph-q-zales.github.io/HMC-MicroPs-Final-Portfolio/doc/). 

Descriptions of the implementation and design of the MCU, RFID Reader, and FPGA follow.

## MCU Design

The MCU role is threefold:
1. Control the I2C communication with the PN532
1. Extract and process the ID information that was sent over I2C
1. Control the SPI communication with the FPGA

Each of these functions are detailed below.

# I2C Device Driver

Much of this development work began with understanding the I2C protocol. The benefit of I2C is that it only uses two wires to communicate and the addition of a new peripheral to the ‘bus’ (the set of two wires, see Figure below) is easy and straightforward. Note that while this website uses the terminology of ‘controller’ and ‘peripheral,’ the previous nomenclature was ‘master’ and ‘slave’ (as is in the illustration below). 

<div style="text-align: center">
  <img src="../assets/img/I2Cbus.png" alt="I2C Bus" width="700" />
</div>
Example of an I2C Bus.[^1]

As in the figure, the two wires have a shared clock and data line. To allow for easy transmission of data, these lines are pulled high (by the Rp resistor) and are ‘open drain’ on the microcontroller and peripherals (meaning they can only be pulled low). By only allowing the devices to pull the lines low, it stops contention on the bus. 

For the peripherals on the bus to know when the data being transmitted across the data line is for them, there is an addressing system. Each peripheral is assigned a 7-bit address (or more recently, a 10-bit address). When a peripheral’s address is called on the bus, it pays attention to what data comes next. With this addressing system, 128 peripherals can be added to one bus thus allowing for easy expansion. The two downsides of using I2C is that it is slower than SPI in that there has to be the peripheral address bit for every interaction and that the peripherals can’t tell the controller that it has data to send. 

As our peripheral requires that the MCU knows when someone has tapped a RFID device, the peripheral uses a ‘handshake’ method. Basically, this is a third wire, which the PN532 calls the IRQ, that is pulled low when an event (tapping the RFID device) takes place. When this signal is drawn low, the MCU knows to poll the PN532 for the RFID number.

We created a custom I2C device driver to enable this I2C functionality on the MCU. To implement I2C on the MCU, we created separate C and header files to control the initialization and sending/receiving of data between the MCU and a peripheral device. To determine all of the appropriate configuration settings, the Reference Manual for the MCU was used.[^2] From looking at the documentation, we determined that we should operate the MCU in “controller receiver” mode and the RFID board in “peripheral transmitter” mode.

The most difficult part of creating this driver was trying to send and receive multiple bytes. To send or receive a byte, the shift register for the data coming in and out of the communication link must be empty or full, respectively. For this to happen, the peripheral must send an ACK byte, telling the MCU that it has acknowledged the data. The ACK byte for the PN532 peripheral is standardized and is shown below.

<div style="text-align: center">
  <img src="../assets/img/ACKframe.png" alt="ACK Frame" width="700" />
</div>
ACK Frame as Defined by the PN532 User Manual.[^3]

Because this ACK byte is so important, we created a special function to determine whether an ACK was read or not. This serves as troubleshooting in the sense that we know not to advance through our code if the peripheral hasn’t sent an ACK back from a command.

# ID Data Processing

The ID collected from the I2C communication is stored in a receive array. From the PN532 documentation, as described below, we know what form to expect this information in. We therefore know that the 4 bytes corresponding to the unique ID of the card correspond to bytes 14, 15, 16, and 17 of this receive array.[^3] The below diagram shows how these bytes of data are mapped to 5 bytes of unique signal data. The FPGA algorithm, described below, explains how these signal data bytes are then transformed into unique tones.

<div style="text-align: center">
  <img src="../assets/img/MCUalg.png" alt="MCU Algorithm" width="700" />
</div>
Creation of 5 Bytes of Signal Data from 4 Bytes of Unique ID Data.

# SPI with FPGA

The MCU serves as the controller in a SPI communication link with the FPGA. Even though SPI involves 4 wires usually (SCL, SDO, SDI, CE), we only require 3 because we never read data in from the FPGA. 

To perform this communication, the MCU asserts chip enable, sends all five bytes of signal data using our SPI device driver, and then lowers chip enable again. 

Shown below is a snapshot of the logic analyzer as 5 bytes of signal data are sent from the MCU to the FPGA. 

<div style="text-align: center">
  <img src="../assets/img/spiDataSent.png" alt="SPI on LA" width="700" />
</div>
Logic Analyzer Snapshot of the MCU Sending 5 Bytes of Signal Data to the FPGA over SPI.

Because of how the MCU performs its algorithm (as just described), these first four bytes (0x4A, 0x55, 0x0C, and 0x40) actually correspond to the ID number for the card that we tested when taking this photo!

# Cumulative MCU Routine

The MCU routine consists of turning on the RF Field for the PN532, giving the PN532 the command to listen for cards, waiting for the PN532 to say that a card has been detected, reading the detected information, creating 5 bytes of signal data from the ID of the card, and sending these bytes to the FPGA over an SPI link. Phew.

## New Hardware: RFID Reader

For this project, we bought a PN532 RFID Reader because of its thorough documentation available online.[^4] This board has 3 supported host interfaces (I2C, SPI, and UART) and 6 different operating modes.[^5] We use this board as an I2C peripheral to the MCU in MIFARE Classic 1K mode. This means that when configured, any card that operates at 13.56 MHz (corresponding to MIFARE) that is held close to the board will be detected and the ID information associated with the card will be transmitted over I2C from the board to the MCU.

As detailed in the implementation of the I2C on the MCU above, 3 lines are used in this communication: SDA, SCL, and IRQ. Normal I2C communication only uses SDA and SCL. The IRQ line in this application serves as a handshake mechanism. Since the PN532 board is the peripheral in this exchange, it cannot take command of the I2C line. Instead, it uses the IRQ bit to alert the MCU that a card has been detected. Once this IRQ line drops, the MCU can then begin the exchange so the PN532 can send back the ID data information that it collected. An example of this exchange is shown below. Once the MCU senses the falling edge of the IRQ line (D2), it opens the I2C channel with the initiation of the SCL (D0). The SDA line (D1) then fills with the output data to be sent back to the MCU.

<div style="text-align: center">
  <img src="../assets/img/IDdetected.png" alt="LA showing ID Detected" width="700" />
</div>
Logic Analyzer Snapshot of the Moment an ID is Detected by the PN532.

The PN532 user manual details the commands supported for configuration of the board.[^3] Commands supported over I2C must adhere to the normal information frame shown below.

<div style="text-align: center">
  <img src="../assets/img/normalInfoFrame.png" alt="Information Frame" width="700" />
</div>
The Normal Information Frame as Defined by the PN532 User Manual.[^3]

All I2C communication begins with the peripheral address (followed by a 0x01 if the information is going from the peripheral to the controller). The address for the PN532 is 0x48. The information frame shown above then follows. As can be seen when comparing the information frame to the logic analyzer snapshot, the data being sent back to the MCU aligns with the information frame as expected. Note that the entire exchange does not fit on one screen of the logic analyzer, and thus just the beginning of the exchange is shown.

In this case, we configure the PN532 for collecting 1 ID card of MIFARE 106 kbps type A. To do this, we use the InListPassiveTarget command. This command configures the board to detect RFID signals in passive mode. The figure below illustrates the information flow for the InListPassiveTarget command. As can be seen, the MCU must first send the command out, and receive an ACK back. Then, once a MIFARE card is detected, the PN532 can send this output data back.

<div style="text-align: center">
  <img src="../assets/img/ILPT_command_exchange.png" alt="Data Exchange ILPT" width="700" />
</div>
An Example of the InListPassiveTarget Command Information Flow as Defined by the PN532 User Manual.[^3]

The expected output data for this command is shown below. This output includes the ID information collected from the card that was detected. As can be seen, this expected data output corresponds to what is shown on the logic analyzer snapshot above when an ID is detected and the data is sent back to the MCU. 

<div style="text-align: center">
  <img src="../assets/img/dataOutILPT.png" alt="UM Output ILPT" width="700" />
</div>
The Output Data of the InListPassiveTarget Command as Defined by the PN532 User Manual.[^3]

All outputs from the PN532 have a TFI byte (see Information Frame) of D5 while all input commands have a TFI byte of D4. The next data byte corresponds to the command being sent or received. The following bytes of data are unique to the command being sent: some have none while some have many. The snapshot below shows the moment the InListPassiveTarget command is sent to the PN532 board. The address is sent followed by the information frame corresponding to the sent command.

<div style="text-align: center">
  <img src="../assets/img/ILPTcommandSent.png" alt="LA showing ILPT Command" width="700" />
</div>
Logic Analyzer Snapshot of the InListPassiveTarget Command being Sent to the PN532.

To be able to detect a RFID card in passive mode, the RF Field of the PN532 must be on. To turn it on, we use the RFRegulationTest command. There is no output for this command.

## FPGA Design

At a high level, the FPGA is sent a set of data from the MCU, creates a set of frequencies and durations from that data, and outputs the corresponding pulse width modulation (PWM) signal. When that PWM signal is connected to a speaker, a set of notes are played. 

To do this, the FPGA code is divided into three cascading sections: data received, data decomposed into frequencies, and a PWM driver output. A fourth section, the enabler, uses signals from other sections to determine when to pass the signals out of SPI and when to start the doorbell.

<div style="text-align: center">
  <img src="../assets/img/fpgaBlock.png" alt="fpga block diagram" width="700" />
</div>

Block Diagram Illustrating a High-level View of the FPGA Code. 

# Data Received from MCU
The first section, data received, is based on the SPI code from Lab 7 (implementing AES on an FPGA).[^6] For lab 7, 128-bits were sent to the FPGA via SPI, and after the FPGA had completed encrypting the data, it sent the encrypted data back. However, as the output of the FPGA goes to a speaker system instead of back to the MCU, the FPGA does not need to have the capability to send anything back via SPI to the MCU. From the MCU, the FPGA receives a 40-bit signal from which to decompose frequencies, durations, and the number of times to repeat.

At its core, SPI is a shared shift register. This means that every signal input (a 1 or 0) shifts the previous inputs left by 1. Therefore, to create an SPI module on the FPGA, on the rising edge of the sck clock line from the MCU, a new bit of data was shifted into the register on the FPGA. At the end of 40 clock cycles (enough to push in the 40 bits), the chip enable (CE) goes from high to low signaling that the data can now be decomposed. 

# MCU Data Decomposed
The second section takes in the 5 8-bit signals and decomposes them. First, it bit swizzles the 40-bit signal into the frequencies, durations, and the number of times the tune repeats. See figure below for more details.

<div style="text-align: center">
  <img src="../assets/img/Decomp.png" alt="Decomposition of Input Signal" width="700" />
</div>

Decomposition of Signals MCU Outputs to Frequencies and Durations Generated by the FPGA.

From these sets of 4-bit signals, the frequencies must be chosen according to the input. For example, if tone0 is 0b0111, the corresponding output will be 440Hz (the note A4). However, the PWM driver takes thresholds as inputs, and so the frequency is mapped to such a threshold (more on this in the section below).

# Doorbell Creation
The last section, and heart of the program, is the creation of the doorbell sound. The doorbell tune creation is configured with a frequency generator, a duration counter, and a finite state machine (FSM) that controls the first two. All three sections must come together to create a PWM signal. In comparison to using the PWM peripheral on a microcontroller, creating one from scratch on an FPGA is not a trivial task. 

### Frequency Generator
To create a signal that oscillates at a known frequency, we used a strobe clock to allow for a fully synchronous design. As described earlier, the frequency generator takes a threshold as an input. This threshold is the number that the strobe clock counts to before the square wave switches. When the clock strobe goes high, the output toggles.
For example, in the figure below, the threshold is 3 meaning that every 4 clock cycles (as the counter starts at 0), the output toggles.

<div style="text-align: center">
  <img src="../assets/img/freqGen.png" alt="Frequency Generator Wave forms." width="700" />
</div>
Example of Frequency Generator's Use of a Clock Strobe.

To create the correct threshold for each frequency, the following algorithm is used:

<div style="text-align: center">
  <img src="../assets/img/freqThresholdEq.png" alt="frequency threshold equation" width="300" />
</div>


The desired frequency is multiplied by two to allow for a 50% duty cycle in the PWM. Additionally, 1 is subtracted as the counter starts at 0. Our clock speed is 24MHz. The table below shows the relationship between note, frequency, and the clock strobe’s threshold.

| Note | Frequency (Hz) | Clock Strobe Threshold |
| ---- | ---- | ------ |
| A3 | 220 | 54,544 | 
| B3 | 247 | 48,582 | 
| C4 | 262 | 45,801 | 
| D4 | 294 | 40,815 | 
| E4 | 330 | 36,363 | 
| F4 | 349 | 34,383 | 
| G4 | 392 | 30,611 | 
| A4 | 440 | 27,272 | 
| B4 | 494 | 24,290 | 
| C5 | 523 | 22,944 | 
| D5 | 587 | 20,442 | 
| E5 | 659 | 18,208 | 
| F5 | 698 | 17,191 | 
| G5 | 784 | 15,305 | 
| A5 | 880 | 13,635 | 

### Duration Counter
Very similar to how the frequency generator works, the duration counter also uses a clock strobe to determine when the expected duration has elapsed. However, as the duration counter is used to determine a time instead of a frequency, the threshold calculation looks slightly different. In the calculation for the duration threshold shown below, the clock speed is again 24MHz and the duration is given in ms.

<div style="text-align: center">
  <img src="../assets/img/durThresholdEq.png" alt="duration threshold equation" width="300" />
</div>


The table below shows a list of durations used and the corresponding clock strobe thresholds.
 
| Duration (ms) | Clock Strobe Threshold |
| ---- | ---- | 
| 1000 | 2.4x10<sup>10</sup> |
| 500 | 1.2x10<sup>10</sup> |
| 250 | 6x10<sup>9</sup> |


### Main FSM
The main FSM combines the frequency generator and the duration counter with the correct durations and frequencies required to create the doorbell tune. Each doorbell tune is made up of 4 notes (each with its own duration), and a number of times the 4 note sequence is repeated. To accomplish this task, we used a finite state machine. The image below shows this FSM.

<div style="text-align: center">
  <img src="../assets/img/fsm.png" alt="fsm" width="500" />
</div>
Finite State Machine for the FPGA PWM Driver Logic.

The FPGA spends most of its time in the idle state, waiting for a new signal to arrive from the microcontroller. Once a signal arrives, if the doorbell isn’t already making music (and the makingMusic output signal to the LED is high), then the start flag goes high, and the FPGA moves to note0. Note that the start flag is controlled by the enabler. In note0, the makingMusic flag is set high to indicate to the user that the doorbell is playing. Additionally, the stopCountFlag is set to 0 (more on this when note3 is discussed). At the same time, the frequency and duration thresholds for note0 are passed to the frequency generator and duration counter allowing them to play the right note. Once the duration counter has reached its threshold, a done signal will go high, thus moving the FPGA into the note1 state. In this state, the frequency and duration thresholds for note1 are passed into the frequency generator and duration counter. Once done goes high, the FPGA moves into state note2 in which the same occurs. 

In note3, the frequency and duration thresholds are passed into the frequency generator and duration counter with the note3 thresholds. Every time the FPGA gets into note3, it adds 1 to a counter and then determines — once done goes high — if the next state will be back to note0 and it will repeat the tune or if it will go to the complete state. To do this, when the FPGA moves into note3, a counter increases (and the stopCountFlag is set until the FSM comes back to note0). If the counter is equal to the threshold set by the repeated byte sent from the FPGA then the FPGA moves into the complete state. In the complete state, the makingMusic flag is set to 0 (and the music stops and LED turns off).

This cycle then repeats the next time a card is tapped to the RFID reader, and the MCU sends the appropriate signals to the FPGA over SPI.

—--

[^1]: J. Valdez and J. Becker, “Understanding the I2C Bus,” p. 8, 2015.
[^2]: “STM32L4 Reference Manual.” ST Microelectronics, Oct. 2018.
[^3]: “PN532 User Manual.” NXP Semiconductors, 2007. [Online]. Available: [https://www.nxp.com/docs/en/user-guide/141520.pdf](https://www.nxp.com/docs/en/user-guide/141520.pdf).
[^4]: PaintYourDragon and LadyAda, “Adafruit-PN532.” Adafruit Industries, Nov. 28, 2022. Accessed: Dec. 08, 2022. [Online]. Available: [https://github.com/adafruit/Adafruit-PN532](https://github.com/adafruit/Adafruit-PN532).
[^5]: “PN532 Datasheet.” NXP Semiconductors, 2017. [Online]. Available: [https://www.nxp.com/docs/en/nxp/data-sheets/PN532_C1.pdf](https://www.nxp.com/docs/en/nxp/data-sheets/PN532_C1.pdf).
[^6]: J. Brake, “E155 Lab 7: The Advanced Encryption Standard.” Harvey Mudd College, 2022.
