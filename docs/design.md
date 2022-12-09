---
layout: page
title: Design
permalink: /design/
---

This system involves four independent systems working together: the RFID Reader, the MCU, the FPGA, and the speaker system. All must be designed independently, and then integrated together.

As the design of the speaker system is analog, it is not described in detail here. It simply involves a LM386 Audio Amplifier and an 8Ω speaker. The breadboard schematic of this system is shown in the [Documentation](https://joseph-q-zales.github.io/HMC-MicroPs-Final-Portfolio/doc/). 

Descriptions of the implementation and design of the MCU, RFID Reader, and FPGA follow.

## MCU Design

INSERT HERE.

## New Hardware: RFID Reader

For this project, we bought a PN532 RFID Reader because of its thorough documentation available online.[^1] This board has 3 supported host interfaces (I2C, SPI, and UART) and 6 different operating modes.[^2] We use this board as a I2C peripheral to the MCU in MIFARE Classic 1K mode. This means that when configured, any card that operates at 13.56 MHz (corresponding to MIFARE) that is held close to the board will be detected and the ID information associated with the card will be transmitted over I2C from the board to the MCU.

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

All I2C communication begins with the peripheral address followed by a 0x00 or 0x01 (0x01 for “read”). The information frame shown above then follows. As can be seen when comparing the information frame with the logic analyzer snapshot, the data being sent back to the MCU aligns with the information frame as expected. Note that the entire exchange does not fit on one screen of the logic analyzer, and thus just the beginning of the exchange is shown.

In this case, we configure the PN532 for collecting 1 ID card of MIFARE 106 kbps type A. Therefore, the expected output data for the command to collect and return the ID data information is shown below. 

<div style="text-align: center">
  <img src="../assets/img/dataOutILPT.png" alt="UM Output ILPT" width="700" />
</div>
The Output Data of the InListPassiveTarget Command as Defined by the PN532 User Manual.[^3]

All outputs from the PN532 have a TFI byte (see Information Frame) of D5 while all input commands have a TFI byte of D4. The next data byte corresponds to the command being sent or received. The following bytes of data are unique to the command being sent: some have none while some have many. The User Manual for the PN532 det

## FPGA Design

At a high level, the FPGA is sent a set of data from the MCU, creates a set of frequencies and durations from that data ,and outputs the corresponding pulse width modulation (PWM) signal. When that PWM signal is connected to a speaker, a set of notes are played. 

To do this, the FPGA code is divided into three cascading sections: data received, data decomposed into frequencies and a PWM driver output. A fourth section, the enabler, uses signals from other sections to determine when to pass the signals out of SPI and when to start the doorbell.

<div style="text-align: center">
  <img src="../assets/img/fpgaBlock.png" alt="fpga block diagram" width="700" />
</div>

This block diagram illustrates a high-level view of the FPGA code. 

# Data Received from MCU
The first section, data received, is based on the SPI code from Lab 7 (implementing AES on an FPGA). For lab 7, 128-bits were sent to the FPGA via SPI, and after the FPGA had completed encrypting the data, it sent the encrypted data back. However, as the output of the FPGA goes to a speaker system instead of back to the MCU, the FPGA does not need to have the capability to send anything back via SPI to the MCU. From the MCU, the FPGA receives a 40-bit signal from which to decompose frequencies, durations and the number of times to repeat.

At its core, SPI is a shared shift register. This means that every signal input (a 1 or 0) shifts the previous inputs left by 1. Therefore, to create an SPI module on the FPGA, on the rising edge of the sck clock line from the MCU, a new bit of data was shifted into the register on the FPGA. At the end of 40 clock cycles (enough to push in the 40 bits), the chip enable (CE) goes from high to low signaling that the data can now be decomposed. 

# MCU Data Decomposed
The second section takes in the 5 8-bit signals and decomposes them. First, it bit swizzles the 40-bit signal into the frequencies, durations, and the number of times the tune repeats. See figure below for more details.

<div style="text-align: center">
  <img src="../assets/img/Decomp.png" alt="Decomposition of Input Signal" width="700" />
</div>

The decomposition of signals from what the MCU outputs through to the frequencies and durations generated by the FPGA.

From these sets of 4-bit signals, the frequencies must be chosen according to the input. For example, if tone0 is 0b0111, the corresponding output will be 440Hz (the note A4). However, the PWM driver takes thresholds as inputs, and so the frequency is mapped to such a threshold (more on this in the section below).

# Doorbell Creation
The last section, and heart of the program is the creation of the doorbell sound. The doorbell tune creation is configured with a frequency generator, a duration counter and a finite state machine (FSM) which controls the first two. All three sections must come together to create a PWM signal. In comparison to using the PWM peripheral on a microcontroller, creating one from scratch on an FPGA is not a trivial task. 

### Frequency Generator
To create a signal that oscillates at a known frequency, we used a strobe clock to allow for a fully synchronous design. As described earlier, the frequency generator takes a threshold as an input. This threshold is the number that the strobe clock counts to before the square wave switches. When the clock strobe goes high, the output toggles.
For example, in the figure below, the threshold is 3 meaning that every 4 clock cycles (as the counter starts at 0), the output toggles.

<div style="text-align: center">
  <img src="../assets/img/freqGen.png" alt="Frequency Generator Wave forms." width="700" />
</div>
Example of the frequency generators use of a clock strobe.

To create the correct threshold for each frequency, the following algorithm is used:

<div style="text-align: center">
  <img src="../assets/img/freqThresholdEq.png" alt="frequency threshold equation" width="300" />
</div>


The desired frequency is multiplied by two to allow for a 50% duty cycle in the PWM. It is then subtracted by 1 as the counter starts at 0. Our clock speed is 24MHz. The table below shows the relationship between note, frequency and the clock strobe’s threshold.

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
Very similar to how the frequency generator works, the duration counter also uses a clock strobe to determine when the expected duration has elapsed. However, as the duration counter is used to determine a time instead of a frequency, the threshold calculation looks slightly different. The clock speed remains 24MHz and the duration is in ms:

<div style="text-align: center">
  <img src="../assets/img/durThresholdEq.png" alt="duration threshold equation" width="300" />
</div>

For a list of durations used and the corresponding clock strobe thresholds, see the table below:
 
| Duration (ms) | Clock Strobe Threshold |
| ---- | ---- | 
| 1000 | 2.4x10<sup>10<\sup> |
| 500 | 1.2x10<sup>10<\sup> |
| 250 | 6x10<sup>9<\sup> |


### Main FSM
The main FSM combines the frequency generator and the duration counter with the correct durations and frequencies required to create the doorbell tune. Each doorbell tune is made up of 4 notes (each with its own duration), and a number of times the 4 note sequence is repeated. To accomplish this task, we used a finite state machine. The image below shows this fsm.

<div style="text-align: center">
  <img src="../assets/img/fsm.png" alt="fsm" width="500" />
</div>
Finite State Machine for the FPGA PWM Driver Logic.

The FPGA spends most of its time in the idle state, waiting for a new signal to arrive from the microcontroller. Once a signal arrives, if the doorbell isn’t already making music (and the makingMusic output signal to the LED is high), then the start flag goes high, and the FPGA moves to note0. Note that the start flag is controlled by the enabler. In note0, the makingMusic flag is set high to indicate to the user that the doorbell is playing. Additionally, the stopCountFlag is set to 0 (more on this when note3 is discussed). At the same time, the frequency and duration thresholds for note0 are passed to the frequency generator and duration counter allowing them to play the right note. Once the duration counter has reached its threshold, a done signal will go high, thus moving the FPGA into the note1 state. In this state, the frequency and duration thresholds for note1 are passed into the frequency generator and duration counter. Once done goes high, the FPGA moves into state note2 in which the same occurs. 

In note3, the frequency and duration thresholds are passed into the frequency generator and duration counter with the note3 thresholds. Every time the FPGA gets into note3, it adds 1 to a counter and then determines — once done goes high — if the next state will be back to note0 and it will repeat the tune or if it will go to the complete state. To do this, when the FPGA moves into note3, a counter increases (and the stopCountFlag is set until the FSM comes back to note0). If the counter is equal to the threshold set by the repeated byte sent from the FPGA then the FPGA moves into the complete state. In the complete state, the makingMusic flag is set to 0 (and the music stops and LED turns off).

—--

[^1]: INSERT REFERENCE FOR adafruit github for the PN532.
[^2]: INSERT REFERENCE FOR PN532 datasheet.
[^3]: INSERT REFERENCE FOR PN532 user manual.
