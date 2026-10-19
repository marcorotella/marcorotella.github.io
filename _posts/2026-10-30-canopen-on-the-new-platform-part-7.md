---
title: "CANopen on the new platform (Part 7)"
date: 2026-10-30 18:30:00 +0100
categories: [Posts]
tags: [canopen, cia301, cia402, drives, sensors, industrial-automation]
image:
  path: /assets/img/rt-part-07-canopen-social.png
  width: 1200
  height: 630
  alt: "CANopen on the real-time Linux platform"
---

A powertrain test bench is not only the device under test. Around it there are auxiliary drives, actuators, power supplies, angle and position encoders, temperature and pressure sensor clusters. Many of them speak CANopen, the CAN in Automation standard that industrial automation has used for decades.

This part describes the architecture we have specified for CANopen. It is the next subsystem to be implemented, so what follows is design, not yet running code.

▪ The standards. The platform acts as CANopen master according to CiA 301 (object dictionary, SDO transfers, PDO process data, NMT network management, SYNC, heartbeat and emergency messages) and CiA 402 for drives (the drive state machine from "switch on disabled" to "operation enabled", profile velocity and torque modes, cyclic synchronous position, velocity and torque). CiA 305 layer setting services assign node IDs and bit rates to unconfigured devices, and CiA 306 device description files (EDS, DCF) describe each device.

▪ Two physical paths, one model. CANopen devices are reached either over native CAN interfaces (the same Kvaser and PEAK devices as in part 5, or a Beckhoff EL6751 terminal) or as CANopen-over-EtherCAT (CoE) slaves through the EtherCAT master. In both cases the objects of a device appear in the same way to the rest of the system.

▪ A deterministic master engine. The master runs on the fieldbus core of the machine, over SocketCAN with non-blocking I/O and timers, without dynamic memory after start-up. It produces the SYNC telegram, packs and unpacks PDOs into pre-allocated memory, and supervises the heartbeats of the slaves: a device that drops off the bus raises a fault event for the cell state machine within one heartbeat period. We will build on an existing open-source stack (CANopenNode or Lely CANopen) instead of writing a new one.

▪ Objects become named channels. Statusword, actual speed, actual torque of a drive become symbols of the Symbol Service, with names such as "Dyno/Drive/ActualSpeed". They can be read, written, recorded and plotted like any EtherCAT or CAN signal, with the same DC timestamps.

▪ Python for configuration and tests. Test scripts can use the widely used python-canopen library on the same interfaces: load an EDS file, bring a node to operational, read and write SDO parameters by name. This is well suited to calibration and commissioning steps that do not belong in the real-time loop.

The approach is the same as for CAN and CAN FD: reuse standards and proven open-source components, put them behind the platform's common interfaces, and verify them against independent implementations.

In the next part: SAE J1939, the standard of heavy-duty powertrains, from 29-bit identifiers to multi-packet messages and diagnostics.
