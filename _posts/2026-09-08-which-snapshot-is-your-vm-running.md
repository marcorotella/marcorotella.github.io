---
title: Which snapshot is your VM actually running?
date: 2026-09-08 20:30:00 +0200
categories: [Posts]
tags: [linux, libvirt, kvm, qemu, qcow2, snapshots, virtualization]     # TAG names should always be lowercase
image:
  path: /assets/img/which-snapshot-is-your-vm-running-social.png
  width: 1200
  height: 630
  alt: A libvirt snapshot chain drawn twice, once as the metadata declares it and once as the filesystem actually holds it, with the good state missing from the second row.
---

A Windows 11 guest on an industrial PC stopped answering on the network. Not crashed, not refusing to boot: it powered on, went through the whole startup, showed a login screen, and then simply was not on the network any more. Yesterday it worked. Today it didn't. Nobody had touched it, which in my experience is always technically true and never actually true.

The machine underneath is the same Beckhoff IPC running TwinCAT Linux RT I wrote about [last time]({% post_url 2026-09-08-firewall-who-is-in-charge %}) — Debian, real-time kernel, KVM/libvirt hosting a single Windows guest next to a handful of containers. And after the firewall episode my first instinct was, of course, that I had broken the firewall again.

I hadn't. The cause turned out to be months older than the symptom, and it was written down nowhere except in the metadata of the hypervisor itself. This post is about reading that.

## Ask what is running before you ask what is wrong

The temptation, when a VM boots but has no network, is to go straight inside the guest: check the adapter, the IP, the routes, the firewall profile. I did some of that. It told me nothing useful, because the guest was configured exactly as a guest that had never been fixed would be configured.

The useful question came from one level down: **which disk is this domain actually booting from today?**

```shell
$ sudo virsh domblklist win11
 Target   Source
--------------------------------------------------
 sda      /var/lib/libvirt/images/win11.pre-network-change
```

That is not a base image. That is an overlay file, named after a snapshot, sitting in the disk slot of a live domain. A file called `pre-network-change` is not something you would deliberately choose as the permanent home of a production guest. Something had rearranged this domain and left the name behind as a fingerprint.

## A file that used to be an overlay

An external libvirt snapshot works by freezing the current disk and creating a new qcow2 file on top of it. The new file holds only the blocks that change from that moment on, and its header carries a pointer — the *backing file* — to the layer underneath. Read a block that was never written into the overlay and QEMU walks down the chain until it finds it.

So the obvious next move was to ask that file about its own header:

```shell
$ sudo qemu-img info -U /var/lib/libvirt/images/win11.pre-network-change
image: /var/lib/libvirt/images/win11.pre-network-change
file format: qcow2
virtual size: 80 GiB (85899345920 bytes)
disk size: 64 GiB
```

No `backing file` line at all. Eighty gigabytes virtual, sixty-four real: this thing was not a thin overlay any more, it was a full, standalone image that happened to still be named like a snapshot. At some point somebody had flattened it — and flattening an overlay is a perfectly legitimate operation, right up until you ask which layer it was flattened *at*.

## What libvirt thought the chain looked like

libvirt keeps its own bookkeeping of snapshots, as XML, entirely separate from the disk files. So I asked it.

First mild annoyance: the libvirt version on this box rejected the `--all` flag that every guide on the internet opens with. Fine. There is a leaner path that works everywhere — get the names, then interrogate each one individually:

```shell
$ sudo virsh snapshot-list win11 --name
pre-network-change
networking_fixed

$ sudo virsh snapshot-info win11 networking_fixed
Name:           networking_fixed
Domain:         win11
Current:        yes
State:          shutoff
Location:       external
Parent:         pre-network-change
Children:       0
```

There it is, in one field. `Current: yes` on `networking_fixed`. According to libvirt, the state this domain is supposed to be sitting on is not the one it is booting from.

`virsh snapshot-dumpxml` fills in the rest, because each snapshot embeds the complete domain XML as it was at the moment the snapshot was taken — including the full disk definition and the backing chain declared underneath it. Put the two dumps side by side and the intended chain is unambiguous:

- `win11.qcow2` — the original base
- `win11.pre-network-change` — first snapshot, no parent, backing file `win11.qcow2`
- `win11.networking_fixed` — second snapshot, parent `pre-network-change`, chaining down to the same base, marked current

## The name lied, and it lied in the most human way possible

Read those two names cold and you will guess, as I did, that `pre-network-change` is the safe state. That is what the word "pre" does to you: it sounds like *before I touched anything*, the state you roll back to when the risky change goes wrong.

It was the opposite. Someone had made a network change, taken a snapshot **before** doing it, then discovered the change was wrong, fixed it, and taken a second snapshot of the working result. `pre-network-change` is the intermediate state — networking half-done, not yet corrected. `networking_fixed` is the good one, which is exactly why libvirt had it flagged as current.

So the file whose name promises safety was the broken one. And the file that was actually good was the one that no longer existed.

## The metadata and the filesystem disagreed

Which brings us to the least glamorous and most important command in this whole investigation:

```shell
$ sudo ls -la /var/lib/libvirt/images/
```

Of the three files in that chain, exactly one was on disk: `win11.pre-network-change`, the middle one, flattened into a standalone image. The original base `win11.qcow2` was gone. `win11.networking_fixed`, the good state, the one libvirt still confidently reported as current — gone too.

![Desktop View](/assets/img/which-snapshot-is-your-vm-running-01.svg)

I cannot tell you exactly how. There are no notes, no shell history worth trusting, no ticket. The most plausible reading is somebody tried to tidy up: consolidate the chain, reclaim some space on a disk that does not have much of it, flatten the overlays back into one file — and did it against the wrong layer, deleting the parent and the good child on the way out. It is not an exotic mistake. `qemu-img commit` and `virsh blockcommit` both do very reasonable things to whichever layer you happen to point them at.

The result was a domain booting, quite happily, from a disk image whose content predates the fix. Which matches the symptom exactly: Windows starts, everything looks normal, and the networking configuration is the broken one from before the correction. The VM was not damaged. It had been quietly time-travelled.

## The backup that was a puzzle with a missing piece

There was a backup. Made the day before, by someone else, presumably as a precaution before whatever went wrong went wrong. For about ninety seconds it felt like a rescue.

```shell
$ sudo qemu-img info -U /path/to/win11-backup.qcow2
image: /path/to/win11-backup.qcow2
file format: qcow2
backing file: /var/lib/libvirt/images/win11.qcow2
```

That is a backup of an **overlay**. It holds the differential blocks and nothing else, and it points at a base file that no longer exists anywhere on this machine or any other.

![Desktop View](/assets/img/which-snapshot-is-your-vm-running-02.svg)

Here is the detail worth carrying away, because it is the trap inside the trap: **`qemu-img info` does not complain**. It printed that backing file path without a word of protest. A declared backing file is just a string in a header — a promise that some other file exists, which nothing verifies until the moment QEMU actually needs to read a block that lives down there. You can copy such a file around for a year, checksum it, feel good about it, and only find out it is unreadable on the day you try to boot it.

A copy of a differential layer is not a backup. It is one piece of a jigsaw, and someone else threw away the box.

## libvirt will not let you clean up either

Once the diagnosis was clear, the housekeeping step looked trivial: remove the stale domain definition and start fresh. libvirt disagreed.

```shell
$ sudo virsh undefine win11
error: Failed to undefine domain win11
error: Requested operation is not valid: cannot delete inactive domain with 2 snapshots
```

Two snapshots whose files had been gone for months, still perfectly real as far as libvirt was concerned. That is the point, restated one more time by the tool itself: **the snapshot bookkeeping is completely independent of whether the files it describes still exist.** libvirt was not checking. It never had been. It was defending the integrity of records that had been fiction since somebody's cleanup afternoon.

The flag you need is one you will not guess until you walk into it:

```shell
$ sudo virsh undefine win11 --snapshots-metadata
```

Worth knowing before you need it, along with `--nvram` and `--tpm` if the guest has those, which on a Secure Boot Windows 11 it does.

## Reading a snapshot chain from scratch

That is the whole investigation. It took a couple of hours, most of it spent believing the file names. Here is the order I would run it in next time, and the reason each step exists — because no single one of these tells you the truth on its own.

![Desktop View](/assets/img/which-snapshot-is-your-vm-running-03.svg)

1. **`virsh domblklist <domain>`** — what disk is the domain configured to boot from *today*. Not what it should be, what it is. If the filename mentions a snapshot, stop and be suspicious.
2. **`qemu-img info -U <file>`** — does this file declare a `backing file`, or is it standalone? Standalone means somebody flattened it. Present means the chain continues, so repeat this step on the parent, and the parent's parent, all the way down. (`-U` lets you inspect a disk that is in use without QEMU objecting.)
3. **`virsh snapshot-list <domain> --name`**, then **`virsh snapshot-info <domain> <name>`** for each one — this is where you find `Parent`, `Children`, and above all `Current`. Current tells you which state the domain is *supposed* to be on, which is the single most useful field when something has gone sideways.
4. **`virsh snapshot-dumpxml <domain> <name>`** — the full backing chain declared at snapshot time, plus a complete copy of the domain XML as it was at that historical moment. When the current definition has been edited beyond recognition, these dumps are the closest thing you have to a version history.
5. **`ls -la` on the actual image directory** — and then sit down and cross-reference all of it.

That last step is the one that does the real work. The first four commands read what libvirt and the qcow2 headers *claim*. Only `ls` reads what is there.

**The metadata tells a story; the filesystem tells the truth. The incident lives in the gap between them.**

## What I took away from it

I did rebuild the VM afterwards, but that is a different post. The part I want to remember is smaller and more general.

External snapshot chains are a fine mechanism and a terrible filing system. They spread one logical machine state across several files that only mean something in relation to each other, with the relationships stored in two places that can drift apart silently — a string in a qcow2 header, and an XML file in libvirt's state directory. Nothing warns you when they stop agreeing. Nothing stops you deleting the layer everything else depends on. And the names, which are the only part a human reads six months later, carry no reliable information about which state is good.

So the guest got rebuilt with boring, self-contained checkpoints instead: internal qcow2 snapshots for quick reverts during configuration, and a full independent copy on a second physical disk, verified with `sha256sum`, for the actual restore path. Both of those are one file that means one thing. Less clever, considerably harder to lose.

And whatever mechanism you pick — write down what the good state is called. Future you will be reading those names in a hurry, with something broken, and no memory of what "pre" was supposed to mean.
