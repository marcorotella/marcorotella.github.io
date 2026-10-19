---
title: "Data you can trust: one clock, fresh values, no lost runs (Part 4)"
date: 2026-10-23 18:30:00 +0200
categories: [Posts]
tags: [data-integrity, distributed-clocks, wal, storage, crash-safe]
image:
  path: /assets/img/rt-part-04-data-you-can-trust-social.png
  width: 1200
  height: 630
  alt: "Data you can trust"
---

A test bench sells one thing in the end: measurements that an engineer can trust. In the new real-time Linux platform, three mechanisms take care of that, and each borrows from a field where getting data right is the whole business.

▪ One clock  from distributed systems.
The EtherCAT Distributed Clocks are the reference for everything: every sample, instrument reading, CAN frame and log record carries DC time as 64-bit nanoseconds since 2000. The real-time loop locks its cycle to the DC reference with a phase-locked loop and publishes the relation between the Linux clock and DC time through a seqlock. Every other process  instrument drivers, the CAN engine, the recorder  converts its own timestamps with it. When the lock is not yet valid, data are flagged "unsynchronized" rather than silently mixed.

▪ Fresh values  from database consistency models.
Databases call it "read-your-writes": after you write, you must read a state that includes your write. A test bench needs the same. When a sequence sets a new speed and then reads the torque, a value measured before the setpoint reached the machine is worse than no value. The platform stamps every command with the cycle in which it was applied, and every instrument reading with the instant it was requested. A read waits  with a timeout derived from the instrument's own rate  until a reading requested after the command is available. In an early simulation, comparing timestamps from two different clocks let pre-stimulus data through. The rule is now explicit: only DC-synchronized samples can satisfy the barrier.

▪ No lost runs  from database storage engines.
Recording uses Parquet files written in rolling 5-second chunks. A chunk is only complete when it is closed, so on its own a crash could lose up to the whole open chunk. Databases solved this decades ago with a write-ahead log. Every open chunk therefore also has an append-only journal (Arrow IPC format), flushed to disk every second. At start-up, any run left open is rebuilt from its journal, every file is checked against its SHA-256 hash and catalogued in SQLite. We tested this by killing the recorder and by cutting the power of the test machine: the data lost were between 0.19 and 0.33 seconds, against a requirement of 5 seconds. Recording 4 kHz × 200 channels costs about 1 % of one CPU core.

Statistics follow the same principle. Running means, variances, minimum and maximum over 100 ms, 1 s and the whole run use numerically stable streaming algorithms (Welford, with Chan's parallel merge). Their windows are measured in DC time, not in sample counts, so a dropped frame or a slow instrument cannot distort them.

For engineers outside the platform, runs export to ASAM MDF4, readable by the usual measurement-analysis tools.

All tests ran on a functional virtual machine; qualification on bare-metal hardware follows.

In the next part: CAN and CAN FD as a software product  device abstraction, DBC decoding, AUTOSAR end-to-end protection and diagnostics.
