---
title: "From TwinCAT to Linux: what changes for the people (Part 6)"
date: 2026-10-28 18:30:00 +0100
categories: [Posts]
tags: [culture, cpp, python, devops, git, automation, team]
image:
  path: /assets/img/rt-part-06-people-from-twincat-to-linux-social.png
  width: 1200
  height: 630
  alt: "What changes for the team moving to Linux"
---

The first five parts were about technology. This one is about the people who build and run test benches, because a platform change is first of all a change in how a team works.

On TwinCAT, a bench team works inside one integrated environment: PLC code in Structured Text, the I/O configuration, the HMI connection through ADS, all in one project. It is productive and well known. Moving to Linux changes several habits at once.

▪ From Structured Text to C++20. We decided not to migrate the existing ST logic, neither by hand nor with a transpiler. The cell logic is written again as explicit C++20 state machines, with a strict real-time subset: no exceptions, no dynamic memory after start-up, no locks on the real-time path. This is a bigger step than changing syntax: it needs engineers who are comfortable with C++, memory and concurrency.

▪ TwinCAT stays where it is irreplaceable. The safety logic runs on a certified Beckhoff safety controller (EL6910). It is still designed and verified in TwinCAT XAE on Windows, and only for that. The safety configuration is then deployed over EtherCAT from Linux. No TwinCAT runtime licence is needed on the bench.

▪ Test sequences in Python. Test engineers write sequences in Python against the Symbol Service: read and write variables by name, wait for fresh measurements, stream data into NumPy or pyarrow. The language most test engineers already use for analysis is now also the language of the sequence.

▪ Configuration as text. Bus configurations, symbol catalogues, instrument settings and CAN mappings are JSON files validated by the same code that runs on the bench. They can be compared, reviewed and versioned like source code.

▪ Version control and reviews instead of project archives. Every change goes through Git. Requirements are tracked issues, architectural choices are recorded in a decision log with the rejected alternatives, and every merge runs the automatic tests, including sanitizer builds that catch real-time violations.

▪ Linux skills become bench skills. Kernel configuration, CPU isolation, interrupt affinity and drivers are now part of commissioning. Provisioning scripts and validation checks make this repeatable, but the team must understand what they do.

What does a team need? In my view: one or two engineers strong in C++ and Linux real-time for the core, test engineers fluent in Python, and a safety engineer who keeps working in TwinCAT XAE. The PLC programmer's knowledge of the machine remains the most valuable asset: it moves from ST code into requirements, state machines and test cases.

AI-assisted engineering lowers the entry barrier considerably: it writes and explains low-level code, keeps the documentation in step with it and runs the verification. It does not remove the need to understand the machine, the timing and the safety concept.

In the next part: CANopen on the new platform, over native CAN interfaces and over EtherCAT.
