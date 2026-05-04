# STM32 Bare-Metal I2C Bridge & Display Driver

A from-scratch, register-level implementation of I2C Master and Slave communication between two STM32 microcontrollers, featuring a custom protocol bridge to a TM1637 7-Segment display.

## Overview
This project demonstrates how to build a multi-MCU architecture without relying on high-level abstractions like the STM32 HAL. By interacting directly with the CMSIS hardware registers, this project showcases the fundamental mechanics of the I2C protocol, interrupt-driven state machines, and hardware-level debugging techniques.

The system consists of an **I2C Master** that transmits data payloads, an **I2C Slave** that catches the data via hardware interrupts, and a **Protocol Bridge** that translates the received I2C byte into a bit-banged GPIO stream to drive a 5V digital display.

## System Architecture

### 1. The Master (Nucleo-F446RE)
* **Role:** Dictates the clock, initiates communication, and transmits numerical data.
* **Peripheral:** `I2C1` (PB8/PB9) configured for Standard Mode (100 kHz).
* **Key Feature:** Includes a custom Software NACK Detector that immediately aborts and frees the bus if the Slave device fails to acknowledge its address, preventing hardware lockups.

### 2. The Slave / Coprocessor (Discovery-F407VG)
* **Role:** Listens passively on the bus and processes incoming commands without blocking the main CPU loop.
* **Peripheral:** `I2C2` (PB10/PB11) to avoid onboard audio-chip conflicts.
* **Key Feature:** Fully interrupt-driven (`I2C2_EV_IRQn`). The CPU sleeps in its main loop and only wakes up to handle `ADDR`, `RXNE`, and `STOPF` hardware flags.

### 3. The Display Driver (TM1637 Protocol)
* **Role:** Visually outputs the data received over I2C.
* **Implementation:** Custom bit-banging driver on 5V-tolerant pins (`PC0`/`PC1`) configured as Open-Drain.

## Hardware Setup & Wiring

**Critical Requirement:** Both boards must share a common ground, and the I2C bus requires external pull-up resistors.

| Connection | Master (Nucleo) | Slave (Discovery) | TM1637 Display | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **Ground** | `GND` | `GND` | `GND` | Common reference required |
| **Power** | `3.3V` (For Resistors) | - | `5V` | Display runs on 5V logic |
| **I2C Clock** | `PB8` | `PB10` | - | 4.7kΩ Pull-up to 3.3V |
| **I2C Data** | `PB9` | `PB11` | - | 4.7kΩ Pull-up to 3.3V |
| **TM1637 CLK**| - | `PC0` | `CLK` | Open-Drain, 5V Tolerant |
| **TM1637 DIO**| - | `PC1` | `DIO` | Open-Drain, 5V Tolerant |

## Build and Configuration Notes
* **Clock Configuration:** Both microcontrollers are configured to bypass the PLL and run directly off their internal 16 MHz HSI oscillators to ensure perfect software delay timing and I2C baud rate synchronization. (`SystemClock_Config()` must be disabled).
* **Pull-ups:** Internal weak pull-ups are enabled in software as a backup, but external 4.7kΩ resistors on the breadboard are required for bus stability.

## Key Learning Outcomes
1. **Register-Level Configuration:** Direct manipulation of `CR1`, `CR2`, `CCR`, and `TRISE` registers to sculpt physical waveforms.
2. **Silicon Errata Workarounds:** Identifying peripheral conflicts (Discovery board audio chip on I2C1) and safely migrating buses.
3. **Interrupt Service Routines (ISR):** Building non-blocking state machines that react to hardware flags instantly.
4. **Protocol Translation:** Bridging a standard protocol (I2C) to a proprietary serial protocol (TM1637) via GPIO bit-banging.