---
title: "From TwinCAT to real-time Linux: a new experiment (Part 1)"
date: 2026-10-16 18:30:00 +0200
categories: [Posts]
tags: [twincat, linux, realtime, ethercat, preempt-rt, testbench, automation]
image:
  path: /assets/img/rt-part-01-from-twincat-to-realtime-linux-social.png
  width: 1200
  height: 630
  alt: "From TwinCAT to real-time Linux"
---

Since 2018 I have conceived, and been the product owner of, two automation software platforms for test benches: components, battery packs, internal combustion engines and electric motors. I built both on Beckhoff TwinCAT, and both have given me — and their users — a lot of satisfaction. TwinCAT was the right choice then, and it is still a very good one.

But the technology landscape has changed.

Linux now ships real-time out of the box: since kernel 6.12, PREEMPT_RT is part of the mainline kernel. Hard real-time no longer means maintaining a patched kernel.

And AI-assisted development has made complex software — including low-level, embedded and real-time code — accessible to far more people. A small team can now specify, implement and verify systems that used to require a large group of specialists.

So I started an experiment: a high-performance, Linux-based real-time platform for test benches. What the specification covers so far:

▪ Real-time core on PREEMPT_RT with native EtherCAT (IgH master) and Distributed Clocks: 250 µs cycle, one DC time base (int64 ns) for every sample in the system
▪ Zero-copy data plane: lock-free shared-memory rings between the RT core and the services, iceoryx2 zero-copy local bus, Apache Arrow columnar batches
▪ A Symbol Service replacing ADS, with a Python client for test sequences: cycle-tagged writes and a "fresh data" barrier, so a sequence never evaluates a value measured before its own command took effect
▪ Asynchronous instruments — climatic chambers, chillers, power supplies, power analyzers — each with its own sampling rate, from 0.5 Hz to 1 kHz
▪ Crash-safe recording: Parquet with a write-ahead journal, SQLite run catalogue, ASAM MDF4 export (data lost after a real power cut: about 0.3 s)
▪ CAN and CAN FD: a device abstraction layer (Kvaser first, PEAK, USB, PCIe and Thunderbolt adapters, EtherCAT gateways such as the Beckhoff EL6751), DBC import with multiplexing, J1939 29-bit identifiers, AUTOSAR E2E checksums and counters, cyclic and event-driven transmission, UDS diagnostics over ISO-TP, raw-frame recording
▪ Next: CANopen (CiA 301 / CiA 402), functional safety over FSoE with Linux as a black channel, qualification on real hardware

Every component is cross-checked against independent tools — for example the DBC decoder against cantools on more than 10,000 frames, and the E2E profiles against an independent AUTOSAR implementation — and the real-time paths run under RealtimeSanitizer. Today everything runs on a functional test VM; the next step is bare-metal hardware, where the real timing figures will come from.

This is the first of a series of posts about this journey: the architecture decisions, what worked and what did not, and what AI-assisted engineering really changes in industrial software.

If you work with test benches, EtherCAT or real-time Linux, I would be glad to compare notes.

In the next part: what a powertrain test bench really requires from its software—not features, but guarantees.
