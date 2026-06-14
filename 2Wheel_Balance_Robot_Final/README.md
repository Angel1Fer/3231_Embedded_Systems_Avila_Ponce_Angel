# Two-Wheel Self-Balancing Robot: Final Report

**Authors:** Avila & Ponce &nbsp;|&nbsp; **Course:** ENCE 3231: Embedded Systems &nbsp;|&nbsp; **Year:** 2026

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Introduction](#2-introduction)
3. [System Overview](#3-system-overview)
4. [Hardware Design](#4-hardware-design)
5. [Firmware Design](#5-firmware-design)
6. [Communication & Remote Control](#6-communication--remote-control)
7. [Balancing Control](#7-balancing-control)
8. [Results](#8-results)
9. [Future Work](#9-future-work)

---

## 1. Abstract

For our final project in ENCE 3231 we built a two-wheel self-balancing robot using an STM32F401RBT6 microcontroller. Everything was made from scratch — the PCB, the chassis, and the firmware. The goal was to control the robot over WiFi from a phone and eventually turn it into a self-balancing system. We were able to get the WiFi connection established and verified that the motors work independently using PWM signals, but we were not able to get the robot driving as a full system by the end of the semester.

---

## 2. Introduction

A two-wheel balancing robot is basically an inverted pendulum on wheels. It naturally wants to fall over, and the job of the controller is to keep spinning the wheels fast enough in the right direction to stay upright. What makes it a good embedded systems project is that everything has to work at the same time — reading sensors, driving motors, and running a control loop, all without any one task slowing the others down.

We wanted to build ours completely from scratch instead of using a kit or a reference board. That meant designing the PCB ourselves, modeling and printing the chassis in OnShape, and writing the firmware from the ground up using STM32CubeMX and CubeIDE.

---

## 3. System Overview

The whole system runs off the STM32F401RBT6. Each part of the robot connects to one of its peripherals. The ESP8266 handles the WiFi side and connects over UART, so a phone can send drive commands and receive telemetry from the robot.

Here are the main components:

| Block | Part | Interface to MCU |
|---|---|---|
| Microcontroller | STM32F401RBT6 (64-pin LQFP) | n/a |
| Motor driver | TB6612FNG (dual H-bridge) | PWM + direction GPIO |
| IMU | MPU6050 (3-axis accel + gyro) | I²C |
| Wheel angle sensor | Magnetic encoder | I²C / Timer |
| Wireless link | ESP8266 | UART |
| Motors | 2x DC gear motors | via TB6612 |

---

## 4. Hardware Design

### 4.1 Microcontroller and Peripheral Mapping

We used the STM32F401RBT6 because it has two I²C buses, enough timers for PWM and encoder reading, and a UART for the ESP8266 — everything we needed without having to bit-bang anything. We mapped out all the pins in STM32CubeMX before starting the schematic to make sure nothing conflicted.

### 4.2 Schematic

The schematic was made in KiCad. We broke it into sections: power regulation, the MCU and crystal, the TB6612 motor driver, the IMU and encoder headers, and the ESP8266 connector. Designing it from scratch meant we had to look up every datasheet and figure out the support circuitry ourselves.

The schematic PDF is saved under `PCB Phase/` and the KiCad source files are there too.

### 4.3 PCB Layout

We laid out and routed the PCB in KiCad. It took a few tries to get the traces clean and avoid crossing signals. The final board has all the blocks labeled and fits everything onto one compact board. Gerber files are included under `PCB Phase/`.

### 4.4 Bill of Materials

We made an interactive BOM using KiCad so you can see where each component sits on the board:

- [`STM32 PCB/BOM/ibom.html`](STM32%20PCB/BOM/ibom.html) (open in a browser)

### 4.5 Fabrication and Assembly

The boards were ordered from an online fab house. We soldered everything by hand in the lab. The STM32 was the hardest part — 64 pins in an LQFP package with very fine pitch. Soldering that by hand is where most assembly issues tend to come from.

### 4.6 Chassis

The chassis was designed in OnShape. We made it to fit the PCB, two gear motors, and a battery holder, with mounting holes lined up to the actual hardware dimensions. The files are under `Assembely Phase/`.

---

## 5. Firmware Design

The firmware was written in STM32CubeIDE. CubeMX handled the peripheral setup and we wrote the application code in the USER CODE sections. Main source files:

- `main.c` — main loop, scheduler, command handling
- `motor_driver.c` — PWM control for the TB6612
- `encoder.c` — reading wheel position
- `imu.c` — MPU6050 driver and complementary filter
- `wifi.c` — sending and receiving data over UART to the ESP8266

### 5.1 Architecture

We used a simple super-loop scheduler based on `HAL_GetTick()`. Each task checks if its interval has passed before running, so everything stays on its own timing without needing an RTOS:

| Task | Rate | What it does |
|---|---|---|
| Heartbeat LED | 100 ms | Blinks to show the loop is running |
| IMU + filter | 10 ms (100 Hz) | Reads sensor and updates tilt angle |
| Telemetry | 100 ms | Sends speed and angle over UART |
| Command handler | every loop | Processes latest drive command |

### 5.2 Sensor Fusion

The gyroscope drifts over time and the accelerometer is noisy during motion, so we fused them with a complementary filter:

```c
angle = 0.995 * (angle + gyro_rate * dt) + 0.005 * accel_angle
```

The gyro handles the fast changes and the accelerometer corrects the drift slowly. Running at 100 Hz this gives a stable tilt angle to feed into the balance controller.

---

## 6. Communication & Remote Control

The ESP8266 connects to USART1 on the STM32. It acts as a WiFi bridge — the robot creates its own access point and a phone or laptop can connect to it and open a browser page to send commands and watch live telemetry. We were able to successfully connect to the robot over WiFi, however we could not get the robot to respond to drive commands through that connection. The communication link was established but full end-to-end control was not achieved.

---

## 7. Balancing Control

Since we were not able to get the robot driving as a complete system over WiFi, we did not reach the balancing stage of the project. The two-wheel self-balancing portion was never attempted. The motors were tested individually and confirmed to work correctly when driven directly with a PWM signal, so the hardware itself is functional. The balancing controller was not implemented.

---

## 8. Results

We completed the full hardware design and build — custom PCB, custom chassis, and firmware — but the robot did not operate as a full system. Here is where things stood at the end:

- [x] PCB designed, fabricated, and assembled
- [x] Chassis designed in OnShape and built
- [x] Firmware compiling and flashing to the board
- [x] IMU reading and complementary filter running
- [x] Motors verified working individually with PWM signals
- [x] ESP8266 WiFi connection established
- [ ] Robot responding to drive commands over WiFi — not achieved
- [ ] Two-wheel balancing — not attempted

The individual subsystems were brought up and tested on their own. The gap was integrating them into a working full system.

---

## 9. Future Work

- Debug why the robot was not responding to WiFi commands despite the connection working
- Get the motors running through the full firmware stack, not just direct PWM
- Attempt the two-wheel balancing once drive control is working
- Tune a PID controller for balancing once the base system is functional
