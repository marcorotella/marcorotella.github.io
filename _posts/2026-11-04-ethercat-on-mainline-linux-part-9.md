---
title: "EtherCAT on mainline Linux (Part 9)"
date: 2026-11-04 18:30:00 +0100
categories: [Posts]
tags: [ethercat, igh, linux, preempt-rt, distributed-clocks, pll]
image:
  path: /assets/img/rt-part-09-ethercat-on-mainline-linux-social.png
  width: 1200
  height: 630
  alt: "EtherCAT on mainline Linux"
---

EtherCAT is the backbone of the bench: drives, I/O, measurement terminals and the safety controller all hang on it. On TwinCAT, the EtherCAT master is part of the product. On Linux, the platform uses the open-source IgH EtherCAT master and builds the real-time loop around it.

▪ The cycle. One thread on an isolated CPU core, with the highest real-time priority, runs every 250 µs. It wakes on an absolute timer, receives the frame of the previous cycle, runs the cell logic, the safety frame routing and the data extraction, queues the outputs and sends the next frame. Nothing in this thread allocates memory, takes a lock or waits for another process.

▪ A native driver, polled. The IgH master drives the network card with its own native driver. The real-time thread polls it directly, so no network interrupt and no Linux network stack sit between the loop and the wire.

▪ A timing budget, calculated before measuring. A small model computes, for a given topology, when the frame leaves, when the outputs reach the last slave and when the frame is back. For the reference topology, about 1 kB of process data (200 measurement channels, control and safety data) takes 93.4 µs on the wire, slightly more than the first estimate, which had left out the datagram headers. The model also showed that the output instant (SYNC0) planned at 180 µs left too little margin for some device types; 195 µs is the recommended value.

▪ Locking onto the bus clock, borrowed from control theory. The Distributed Clocks of the slaves define the time of the bus. The master does not impose its own clock: it follows the reference slave with a phase-locked loop, adjusting its wake-up phase and rate every cycle, with a large-step mode at start-up and a slew limit afterwards. In simulation, with clock drifts between -80 and +50 ppm, the loop locks within 2000 cycles (half a second) and holds the phase within 2.5 µs. Writing the tests also showed a subtle point: the cable delay to the reference slave cannot be learned from the loop itself, because it looks exactly like a clock offset. It is a constant measured at commissioning.

▪ The rest of the machine stays out of the way. Linux housekeeping and all interrupts are confined to two cores, the real-time threads have their own cores, and the kernel's real-time throttling is disabled. Standard tools (cyclictest, rtla) measure the latency on the target machine; the requirement is 20 µs in the worst case over a long run.

▪ The same time everywhere. The loop publishes the relation between the Linux clock and the bus clock, so every other process (instruments, CAN, recorder) stamps its data in EtherCAT DC time, as described in part 4.

The figures above come from the timing model and from simulation on a functional test VM. The latency figures of the real machine will come from the hardware qualification.

In the next part: replacing ADS. How test sequences read and write variables by name, from C++ to Python.
