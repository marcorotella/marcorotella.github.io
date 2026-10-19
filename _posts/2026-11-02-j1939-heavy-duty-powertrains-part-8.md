---
title: "SAE J1939 for heavy-duty powertrains (Part 8)"
date: 2026-11-02 18:30:00 +0100
categories: [Posts]
tags: [j1939, heavy-duty, commercial-vehicles, pgn, powertrain]
image:
  path: /assets/img/rt-part-08-j1939-heavy-duty-social.png
  width: 1200
  height: 630
  alt: "SAE J1939 for heavy-duty powertrains"
---

Passenger cars mostly use OEM-specific CAN matrices, described in DBC files. Heavy-duty vehicles are different: trucks, buses, off-highway machines, marine gensets and the new electric and hydrogen commercial powertrains all speak a common standard, SAE J1939. A test bench for these engines, transmissions, battery packs and axle drives must speak it too.

What the platform does today: J1939 messages use 29-bit CAN identifiers, and the CAN subsystem of part 5 already decodes them from a DBC file, with the same timestamps, recording and checks as any other message. What follows is the J1939 layer we have specified on top of it, the next step of the CAN work.

▪ Identifiers with meaning. In J1939 the 29-bit identifier is structured: priority, parameter group number (PGN) and source address. The platform decodes the PGN and the address of every frame, so a channel can be bound to "engine speed from the engine controller" rather than to a raw identifier, and the same engine can be tested whatever address it has.

▪ Standard parameters out of the box. The common powertrain parameter groups (EEC1 with engine speed and torque, EEC2, ET1 with coolant and oil temperatures, ETC1 for the transmission) are decoded with their standard scaling. A test can start without first writing a message database.

▪ Messages longer than 8 bytes. J1939 carries up to 1785 bytes with its transport protocol: broadcast announcements (BAM) or a peer-to-peer connection with request-to-send and clear-to-send. Diagnostic trouble code lists, version strings and calibration data travel this way.

▪ Address claiming. J1939 nodes negotiate their source addresses at start-up with a 64-bit NAME. The bench must take part correctly, or the ECU may refuse to talk to it.

▪ Diagnostics. Active trouble codes (DM1) are decoded into suspect parameter number, failure mode and occurrence count, monitored during the test and recorded with the run.

The interesting part is how little of this has to be written from scratch. Since version 5.4 the Linux kernel has its own J1939 implementation: the kernel assembles multi-packet messages, handles address claiming and filters by PGN. The application receives a complete message with one read call. As with ISO-TP for UDS diagnostics in part 5, the platform relies on the mainline kernel and puts its own effort into integration, timing and verification.

In the next part: EtherCAT on mainline Linux, and how the real-time loop locks onto the Distributed Clocks of the bus.
