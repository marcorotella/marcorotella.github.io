---
title: "AI-assisted engineering in practice (Part 12)"
date: 2026-11-11 18:30:00 +0100
categories: [Posts]
tags: [ai, claude-code, agentic-coding, software-engineering, verification]
image:
  path: /assets/img/rt-part-12-ai-assisted-engineering-social.png
  width: 1200
  height: 630
  alt: "AI-assisted engineering in practice"
---

In the first part I wrote that AI has made complex, low-level software accessible to far more people. This part shows what that means on this project, with numbers from the repository and an honest split of the work.

▪ What the AI did. I work with Claude Code, an AI coding agent that operates directly on the repository, a Linux test VM and the issue tracker. From the operational prompts and analyses, it wrote the specifications, the C++20 and Python code, the tests and the provisioning scripts; it built and ran everything on the VM, including sanitizer builds, crash tests and stress tests; it kept the documentation, the decision log and the issue tracker up to date, with atomic commits.

▪ What stayed with me. The requirements and the architecture analyses, written from years of product ownership on test benches. The choice of what to build next and in which order. The approval of every architectural decision: each one is recorded with its alternatives and stays "proposed" until I approve it. And the review of what is published, including these posts.

▪ The numbers after the first days (3 to 6 October 2026):
about 19,600 lines of C++ code and 5,300 lines of C++ tests;
about 3,800 lines of Python;
148 automatic C++ tests and 26 Python tests, passing in normal, RealtimeSanitizer and ThreadSanitizer builds;
35 requirements and 10 architectural decisions, all tracked;
about 130 commits;
6 specifications with verification results and open points.
The AI work used about 4.6 million tokens, logged per phase.

▪ What the tests caught. The value is not that the code is written quickly, it is that it is checked. A few examples from the logs: a phase-locked loop design that locked to a wrong phase when the cable delay was learned at start-up; a middleware call that allocated memory on a real-time path; a barrier that let stale data through when two clocks were compared; a missing Kvaser driver in the distribution kernel; a write acknowledgement that could carry the timestamp of the wrong frame. Each one was found by a test and is documented with its fix.

▪ What did not change. The AI does not know the machine. It does not know which interlock trips first, how a dyno behaves at overspeed, or what a customer will accept at factory acceptance. Timing must still be measured on real hardware, and the safety concept still belongs to a certified controller and a safety engineer. Every number in this series that comes from a virtual machine is labelled as such for that reason.

▪ What it changes. A specification, the code and its tests can evolve together instead of drifting apart. Cross-checks against independent implementations, sanitizer builds and power-cut tests stop being luxuries that a small team skips. And an engineer with domain knowledge can turn that knowledge into working, verified software without a large team around them.

At the end of the project I will publish an estimate of the equivalent effort for a conventional team, with the method and the sources, next to the actual time and AI usage.

In the next part: functional safety with Linux as a black channel.
