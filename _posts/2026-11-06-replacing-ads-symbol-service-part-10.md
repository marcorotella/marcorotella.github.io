---
title: "Replacing ADS: variables by name, from C++ to Python (Part 10)"
date: 2026-11-06 18:30:00 +0100
categories: [Posts]
tags: [ads, symbol-service, python, cpp, ipc, automation]
image:
  path: /assets/img/rt-part-10-replacing-ads-symbol-service-social.png
  width: 1200
  height: 630
  alt: "Replacing Beckhoff ADS with Symbol Service"
---

Anyone who has built software around TwinCAT knows ADS: the protocol through which an HMI, a test sequence or an analysis tool reads and writes PLC variables by name. Leaving TwinCAT means providing the same service. On the new platform it is the Symbol Service.

▪ A catalogue of everything. Every variable of the bench (EtherCAT inputs and outputs, setpoints, status, measurement channels, instruments, CAN signals) is a symbol with a name, a type, a unit, a scaling, limits and access rights. The catalogue is loaded from JSON files at start-up and then fixed.

▪ Constant-time lookup, borrowed from compilers and databases. Names are resolved with a minimal perfect hash: one hash, one string comparison, no search. On the test VM, finding a symbol among 8000 takes well under a microsecond.

▪ Reading without touching the real-time loop. Every cycle, the real-time loop publishes a snapshot of its state (inputs, setpoints, status) into a triple buffer in shared memory, at a cost of about 50 ns. The Symbol Service is its only reader and serves all clients from it. A hundred clients reading at once do not cost the real-time loop a single extra nanosecond.

▪ Writes with a cycle number. A write is a command with the cycle in which it must take effect. The real-time loop applies it in that exact cycle and acknowledges it with the cycle number and the DC time. Several variables written together change in the same cycle, like an ADS sum write. The acknowledgement is precise enough to know which measurement is the first one that can reflect the change.

▪ "Write and await response". The most used function in a test sequence: set a value, then wait for the first measurement that can show its effect. The service knows the input latency of the bus (the cycles between an output and the first input that can see it) and returns the first sample after it. In the end-to-end test, 20 setpoint steps out of 20 returned the new value, never a stale one.

▪ Python as a first-class client. The Python client uses only the standard library. Reading, writing, waiting for fresh data, streaming subscriptions delivered as pyarrow or NumPy arrays and events (state changes, alarms) are each a few lines. A read-only HTTP interface lets any tool or browser inspect the catalogue and current values.

▪ Device channels too. A CAN signal that the bench transmits is a writable symbol as well: the write is confirmed when the frame carrying the new value has left the controller.

In short: the convenience of ADS, with explicit timing guarantees that a test sequence can rely on.

All figures were measured on a functional test VM.

In the next part: climatic chambers, chillers and power analyzers, each working at its own pace.
