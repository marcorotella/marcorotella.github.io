---
title: "CAN and CAN FD as a software product (Part 5)"
date: 2026-10-26 18:30:00 +0100
categories: [Posts]
tags: [can, can-fd, dbc, autosar, kvaser, peak, automotive]
image:
  path: /assets/img/rt-part-05-can-canfd-software-stack-social.png
  width: 1200
  height: 630
  alt: "CAN and CAN FD software stack"
---

On a powertrain test bench, CAN is where the device under test speaks: the battery management system, the inverter, the engine control unit. For the new real-time Linux platform, the goal was not "a CAN driver" but a CAN subsystem that a test engineer can configure from a DBC file and trust without reading the code.

▪ One abstraction for every device. Kvaser is the first supported vendor, with PEAK and others alongside. USB, PCIe, M.2 and Thunderbolt adapters as well as EtherCAT gateways (Beckhoff EL6751, esd CAN-EtherCAT/2-FD) sit behind the same channel interface. Devices are selected by driver, serial number and channel, so a USB adapter keeps its role when the operating system renumbers it. Building it turned up a surprise: the Debian real-time kernel does not include the driver for Kvaser's PCIe CAN FD cards. A provisioning script now builds it from the matching kernel source; the platform kernel will enable it in the long run.

▪ The DBC file is the single source of truth. A C++ parser compiles every signal  Intel or Motorola byte order, signed, float, simple and nested multiplexing, J1939 29-bit identifiers, CAN FD frames up to 64 bytes  into a fixed decoding routine that allocates no memory. The configuration screen offers the DBC signals in a drop-down list with a filter; the engineer maps them to channel names, and the same parser validates the mapping.

▪ Safety mechanisms borrowed from automotive. ECUs protect their messages with counters and checksums. The platform implements the AUTOSAR end-to-end profiles (P01, P02, P04, P05, P11, P22) and custom CRC, XOR and sum schemes, for Classic CAN and CAN FD. A frame with a wrong checksum marks its values invalid; a skipped counter marks possible data loss.

▪ A write is confirmed on the wire. When a test sequence changes a value sent on CAN, the confirmation carries the time the frame actually left the controller, taken from the transmit echo of the frame that carried the value.

▪ Everything is recorded: decoded signals with their timestamps, and every raw frame, for later analysis.

▪ Diagnostics: UDS requests over the Linux kernel's ISO-TP transport, on Classic CAN and CAN FD.

What I find most valuable is how the subsystem was verified. Each part was compared against an independent implementation:

▪ The DBC decoder against cantools on more than 10,000 random frames, including 82 test files from the cantools project: no differences.
▪ The end-to-end profiles against an independent AUTOSAR E2E library on 583 test vectors: all identical.
▪ A simulated ECU, built only from those independent tools, checked the whole chain from the CAN interface to the Python test script.
▪ A stress test sent up to eight times the frame rate of an 80 %-loaded CAN FD bus without losing a single frame.

The figures come from a virtual machine with virtual CAN interfaces. Real hardware, real bus load and real timestamps are the next step.

In the next part: CANopen, and what moving from TwinCAT to Linux means for the people who build and run test benches.
