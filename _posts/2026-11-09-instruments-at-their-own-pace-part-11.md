---
title: "Climatic chambers, chillers and power analyzers at their own pace (Part 11)"
date: 2026-11-09 18:30:00 +0100
categories: [Posts]
tags: [instruments, scpi, modbus, asynchronous, testbench, daq]
image:
  path: /assets/img/rt-part-11-instruments-at-their-own-pace-social.png
  width: 1200
  height: 630
  alt: "Asynchronous bench instruments"
---

A test bench is not only EtherCAT. Climatic chambers, chillers, battery cyclers, power supplies, power analyzers and precision multimeters are connected over SCPI, Modbus TCP or CAN, and they all have their own rhythm. A climatic chamber answers every two seconds, a power analyzer a thousand times per second. Treating them as if they ran at the cycle rate is one of the classic sources of wrong test results.

The new platform treats every instrument as what it is: an asynchronous device with its own rate.

▪ The rate is a parameter of each instrument. Each instrument is configured with its own sampling rate, from 0.5 Hz to 1 kHz, and its typical reply latency. Everything else derives from these two values.

▪ Buffers sized from the rate. Each instrument gets its own wait-free ring, sized to hold ten seconds of data: 16 entries for a 0.5 Hz chamber, 16384 for a 1 kHz analyzer. A slow device does not waste memory, a fast one does not overflow.

▪ Timeouts and staleness from the rate. How long a test sequence may wait for a fresh reading, and when a value is considered stale, are computed from the instrument's period and latency: about four seconds for the chamber, under three milliseconds for the analyzer. No global timeout that is too short for one device and too long for another.

▪ Two timestamps per reading. Every reading carries the DC time at which the request was sent and the time at which the reply was received, in the same clock as the EtherCAT data. A sequence can tell exactly whether a reading was requested before or after its command.

▪ The fresh-sample barrier. After a command, the platform returns the earliest reading requested after the command took effect. A reading requested before is pre-stimulus data and is never returned. If no fresh reading arrives in time, the error says how long it waited, how many readings arrived and how many of them were too old.

▪ No resampling at recording time. Instruments are recorded with their own timestamps and rate, in the same run as the 4 kHz channels. Aligning them onto a common grid is an analysis decision, not something the recorder imposes.

▪ A simulated bench. Every instrument has a simulator with a configurable rate, reply latency and a simple physical model, so sequences, barriers and recording can be tested without the hardware. The simulated bench runs five instruments from 0.5 Hz to 1 kHz together with the simulated EtherCAT master.

The SCPI and Modbus drivers for real devices use the same rings, clock and messages; they are the next step for this part of the platform.

In the next part: AI-assisted engineering in practice. What the AI did on this project, what stayed with the engineer, and what the numbers say.
