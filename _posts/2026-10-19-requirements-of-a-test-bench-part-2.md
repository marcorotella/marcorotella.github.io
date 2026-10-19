---
title: "What a test bench really requires (Part 2)"
date: 2026-10-19 18:30:00 +0200
categories: [Posts]
tags: [testbench, requirements, realtime, determinism, ethercat, fsoe]
image:
  path: /assets/img/rt-part-02-requirements-of-a-test-bench-social.png
  width: 1200
  height: 630
  alt: "What a test bench really requires"
---

Before writing a single line of code for the new real-time Linux platform, I spent time on the question that decides everything else: what does a powertrain test bench actually need from its software? Not features  guarantees.

After years of product ownership on TwinCAT-based systems, these are the requirements I wrote down, each as a measurable statement:

▪ A deterministic cycle. The control loop runs every 250 µs (4 kHz). On the target hardware, the wake-up latency of the real-time thread must stay within 20 µs over a long run  not on average, in the worst case.

▪ One clock for everything. Every sample  EtherCAT inputs, instruments, CAN frames, log records  carries the same time base: EtherCAT Distributed Clocks time, as 64-bit nanoseconds. DC sampling jitter across the I/O must stay within 1 µs.

▪ Never act on stale data. When a test sequence commands a new setpoint and then reads a measurement, it must get a value requested after the command took effect. This "request-instant barrier" turned out to be one of the most important rules of the whole design.

▪ Real-time never waits. When a consumer is slow, the real-time side drops the newest data and counts the loss; it never blocks, never allocates memory, never takes a lock.

▪ Every instrument at its own pace. A climatic chamber answers at 0.5 Hz, a power analyzer at 1 kHz. Buffers, timeouts and staleness limits are derived from each instrument's own rate.

▪ Data survives a crash. A power cut may cost at most 5 seconds of recorded data, and the files that remain must be complete and verifiable.

▪ Safety per function. Each safety function gets its own Performance Level / SIL; the safety logic runs on a certified FSoE controller, with Linux as a "black channel" that cannot compromise it.

The requirements translate directly into the architecture:

▪ Layer 1: the EtherCAT cycle on an isolated CPU core with SCHED_FIFO priority 98.
▪ Layer 2: statistics and streaming on a second isolated core.
▪ Services (Symbol Service, instruments, CAN, storage) on their own cores, housekeeping and Linux itself confined to two cores, interrupts routed away from the real-time cores.

The boundary between real-time and everything else is a set of lock-free shared-memory structures: nothing else crosses it.

The method matters as much as the content. Every requirement is a tracked issue, every architectural choice is recorded in a decision log with the alternatives that were rejected, and every module has a verification plan in two tiers: a functional virtual machine for development and sanitizer builds, and a bare-metal host for the real timing figures. AI-assisted engineering makes this level of documentation affordable: the specification, the code and the tests evolve together instead of drifting apart.

In the next part: the zero-copy data plane, and the techniques borrowed from high-frequency trading, robotics and big-data analytics to build it.
