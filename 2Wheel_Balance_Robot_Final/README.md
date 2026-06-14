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

This report covers a two-wheel self-balancing robot built around an STM32F401RBT6 microcontroller, designed entirely from scratch for ENCE 3231. Unlike reference designs, we created our own PCB, chassis, and firmware from the ground up. The full embedded stack includes a custom KiCad PCB with the MCU, motor driver, IMU, magnetic encoders, and a WiFi module that lets a phone drive the robot and read back live telemetry. The chassis was designed in OnShape and 3D printed. The robot started as a 3-wheel platform; the ball wheel was later removed to transition into a 2-wheel self-balancing configuration, making balance control the core challenge.

---

## 2. Introduction

A self-balancing robot is the classic inverted-pendulum problem in mobile form: two wheels on one axis, a body that wants to tip over, and a controller that drives the wheels just enough — and in the right direction — to keep it upright. It is a strong project for an embedded systems course because every layer of the stack has to work together. The sensors must be read quickly and fused well, the motors must react with low latency, and the control loop must run at a steady rate or the robot falls over.

Our goal was to build the entire system ourselves — no reference PCB, no prefab chassis — and then add an extra challenge on top of the balancing system once the base is working.

---

## 3. System Overview

The design is built around the STM32F401RBT6. Each sensor and actuator hangs off one of its peripherals. The ESP8266 Wi-Fi module sits on a UART and works like a wireless cable, so a phone or laptop can drive the robot and watch telemetry from a browser.

The main parts are:

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

The board uses an STM32F401RBT6. It has enough timers, two I²C buses, a UART, and PWM outputs, which is what we need to talk to two separate sensors, drive a timing-sensitive motor driver, and run a UART link to the ESP8266 without bit-banging anything.

The pins were set up in STM32CubeMX so every peripheral maps onto the F401's resources without conflicts: the two I²C buses, the motor PWM and direction lines, the UART to the ESP8266, and SWD/USB.

### 4.2 Schematic

The schematic was designed in KiCad from scratch. It breaks the system into power input and regulation, the MCU and its crystal, the TB6612 motor driver, the two I²C sensor headers, and the ESP8266 connector.

The full-resolution schematic is included as `figures/schematic.pdf`, and the editable KiCad source is under `hardware/schematic/`.

### 4.3 PCB Layout and 3D Renders

The PCB layout and routing were the main hands-on part of this project. We placed the components and routed the board in KiCad. The editable PCB project, gerbers, and drill files are under `PCB Layout/`.

### 4.4 Bill of Materials

There is an interactive HTML BOM so you can cross-check component placement against the layout:

- [`STM32 PCB/BOM/ibom.html`](STM32%20PCB/BOM/ibom.html) (open in a browser)

### 4.5 Fabrication and Assembly

The bare boards were fabricated externally. We hand-soldered the board in the lab. The STM32F401RBT6 in its fine-pitch 64-pin LQFP package was the most difficult part to solder by hand.

### 4.6 Chassis

The chassis was designed entirely in OnShape. It is a custom structure that everything mounts to: the PCB, the two gear motors, and the battery holder. All dimensions were measured to line up with the mounting points on the hardware. The chassis CAD files are under `Assembely/`.

---

## 5. Firmware Design

The firmware is an STM32CubeIDE project (`Program/`). CubeMX generates the peripheral init, and the application code goes in the `USER CODE` sections. The main files:

- `Program/Core/Src/main.c` — scheduler, telemetry, command handling
- `Program/Core/Src/motor_driver.c` — TB6612 motor driver
- `Program/Core/Src/encoder.c` — magnetic encoder
- `Program/Core/Src/imu.c` — MPU6050 + complementary filter
- `Program/Core/Src/wifi.c` — ESP8266 UART communication

### 5.1 Architecture

The application uses a cooperative super-loop with a simple tick-based scheduler built on `HAL_GetTick()`. Each task runs on its own interval, which keeps timing predictable without needing an RTOS:

| Task | Rate | What it does |
|---|---|---|
| Heartbeat LED | 100 ms | Toggles the user LED so the loop is visibly alive |
| IMU + complementary filter | 10 ms (100 Hz) | Reads the MPU6050 and updates the fused angle |
| Telemetry | 100 ms | Packs speed/angle and ships it over UART |
| Command service | every loop | Acts on the latest received drive command |

### 5.2 Sensor Fusion

A raw accelerometer angle is noisy when the robot moves, and a raw gyro angle drifts over time. The firmware combines them with a **complementary filter** using a coefficient of `0.995`:

```c
angle = 0.995 * (angle + gyro_rate * dt) + 0.005 * accel_angle
```

The gyro term handles fast changes and the accelerometer slowly pulls out the drift. The filter runs at 100 Hz and produces the fused tilt angle that the balance loop runs on.

---

## 6. Communication & Remote Control

The ESP8266 is connected to USART1 on the STM32. It runs as a transparent serial bridge — the STM32 sends and receives plain text commands and telemetry frames over UART, and the ESP8266 exposes that as a WiFi access point. A phone or laptop connects to the robot's WiFi network and opens a browser-based interface to drive the robot and watch live telemetry.

---

## 7. Balancing Control

The balancing controller was designed around the fused tilt angle from the complementary filter, targeting a 100 Hz PID control loop to drive the motors and keep the tilt angle at zero. The PID was planned and partially implemented, however the robot was unable to achieve stable balancing during the course of the project. Hardware issues encountered during bring-up, including problems with the power rail reaching the MCU, prevented full end-to-end testing of the control loop. The design and logic are complete and documented here as the intended implementation.

---

## 8. Results

The robot was fully designed and assembled — custom PCB, custom chassis, and firmware — but was ultimately unable to balance. The following was completed and verified:

- [x] Custom PCB designed and fabricated
- [x] Chassis designed in OnShape and assembled
- [x] STM32 firmware compiling and flashing successfully
- [x] IMU reading and complementary filter implemented
- [x] Motor driver responding to PWM commands
- [x] ESP8266 WiFi link established
- [ ] Balancing PID — implemented but not tuned; robot did not balance
- [ ] Additional challenge layer — not reached

Despite not achieving balance, the project successfully demonstrated the full embedded design process from schematic to PCB to firmware, and all individual subsystems were brought up and tested independently.

---

## 9. Future Work

- Resolve hardware power rail issue to enable full MCU bring-up
- Tune the PID balancing controller with a working board
- Add a browser-based live telemetry dashboard
- Implement the additional challenge layer once balancing is stable
