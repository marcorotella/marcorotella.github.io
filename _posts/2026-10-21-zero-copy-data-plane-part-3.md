---
title: "A zero-copy data plane, borrowed from other industries (Part 3)"
date: 2026-10-21 18:30:00 +0200
categories: [Posts]
tags: [zerocopy, arrow, parquet, iceoryx2, performance, realtime]
image:
  path: /assets/img/rt-part-03-zero-copy-data-plane-social.png
  width: 1200
  height: 630
  alt: "Zero-copy data plane architecture"
---

A powertrain test bench produces a steady stream: 200 channels at 4 kHz, plus instruments and CAN buses. The data has to leave the real-time loop without disturbing it, and reach statistics, storage, the HMI and Python test scripts without being copied over and over.

Industrial automation has rarely had to solve this problem at this scale. Other industries have, so the new platform borrows their solutions.

▪ From high-frequency trading: wait-free single-producer/single-consumer rings. The real-time thread writes each cycle's frame into a ring in shared memory, in place, without locks, allocation or system calls. Each index sits on its own cache line, so the producer and consumer never contend. When the ring is full, the newest frame is dropped and counted: the real-time side never waits.

▪ From operating-system kernels and game engines: seqlocks and triple buffers. The clock relation and the process snapshot (inputs, setpoints, status: about 6 KB per cycle) are published so that the writer never waits and the reader never sees a half-written state. Publishing the snapshot costs about 50 ns.

▪ From robotics and automotive middleware: iceoryx2, a zero-copy publish/subscribe bus in shared memory (Eclipse, Rust core). Subscribers receive a reference to the data, not a copy, regardless of message size.

▪ From big-data analytics: Apache Arrow. A streaming bridge transposes the row-oriented cycle frames into column-oriented batches (400 rows = 100 ms per batch), in cache-sized chunks with 64-byte aligned columns. The batch is laid out exactly as Arrow expects, so Python (pyarrow, NumPy) and the Parquet recorder read it without converting or copying it.

▪ From compilers and databases: minimal perfect hashing. The Symbol Service (our replacement for Beckhoff ADS) resolves any variable name in constant time with one string comparison, from a table that is fixed once the configuration is loaded.

Borrowing also means checking what you borrow. A RealtimeSanitizer test showed that publishing on iceoryx2 0.10 takes a mutex and allocates memory. That is acceptable in a service, but not on a real-time core. The fix is a wait-free handoff: the bridge passes a finished batch to a non-real-time thread that publishes it, at a cost of about 4 µs per 333 KB batch outside the real-time path. Every real-time function in the code base is annotated and checked automatically by RealtimeSanitizer: any allocation, lock or blocking call on these paths fails the tests.

The result is a pipeline where a measured value is written once by the real-time loop and then read in place by every consumer, from the statistics engine to a Python script analysing the run.

All figures above were measured on a functional test VM. The bare-metal numbers will come with the hardware qualification.

In the next part: data you can trust  one clock for the whole system, protection against stale data, and storage that survives a power cut.
