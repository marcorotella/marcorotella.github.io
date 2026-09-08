---
title: Who's really in charge of your firewall?
date: 2026-09-08 19:00:00 +0200
categories: [Posts]
tags: [linux, networking, nftables, firewall, docker, libvirt]     # TAG names should always be lowercase
image: /assets/img/firewall-who-is-in-charge-social.png
---

![Desktop View](/assets/img/firewall-who-is-in-charge-header.svg)

I spent a full day this week inside the firewall of a single Linux box. Not building a new one, not migrating from one thing to another: just trying to understand the one that was already there.

The machine is a bit unusual for a homelab, but the problem it handed me is not: a Beckhoff industrial PC running TwinCAT Linux RT — Debian underneath, real-time kernel, automation runtime on top — plus a few Docker containers, one KVM guest, and a pile of nftables rules that had grown over time, partly written by me, partly by somebody else, partly by nobody at all. By the end of that day I had broken it, fixed it, broken it again in a new and more creative way, and finally understood something I was pretty sure I already knew.

So this is not a "top 10 nftables rules you must have" post, there are plenty of those. This is the mental model I wish I had had that morning, written down while the bruises are still fresh.

## You are already running nftables, even when you type iptables

First thing to get out of the way, because it confuses everybody (me included, for years).

On a modern distribution, `iptables` is very likely not "the old firewall" running next to "the new firewall". It is a **front end**. The `iptables-nft` binary takes your familiar `-A FORWARD -j ACCEPT` syntax and translates it into nftables rules, which is what the kernel actually stores. You can see the seam if you dump the live ruleset:

```shell
$ sudo nft list ruleset
table ip filter {
        # Warning: table ip filter is managed by iptables-nft, do not touch!
        ...
}
```

That comment is nftables telling you, politely, that somebody else is holding this pen. On my box that somebody is Docker: it speaks the iptables dialect, and every time a container starts, stops or publishes a port, its chains (`DOCKER`, `DOCKER-USER`, `DOCKER-FORWARD` and friends) appear, change or disappear in the kernel ruleset. libvirt does exactly the same for the VM bridge, with its own set of `LIBVIRT_*` chains.

So the interesting question was never "iptables or nftables". It is: **who is writing rules into my kernel, and when?** On that box the answer was "three of us, at different moments, none of them asking the others first".

## Everything hangs off a hook

Netfilter, the thing inside the kernel that actually filters, exposes a handful of hook points along the path a packet takes: `prerouting`, `input`, `forward`, `output`, `postrouting`. A *base chain* is simply a chain that says "attach me to hook X with priority Y". Lower priority numbers (negative ones included) are evaluated first.

Two consequences that took me an embarrassingly long time to internalise.

**One: the hooks happen in a fixed order, and NAT comes early.** `prerouting` runs *before* the kernel decides whether the packet is for this host or has to be forwarded somewhere else. Which means a DNAT rule can silently invalidate a filter rule that looks perfectly correct.

![Desktop View](/assets/img/firewall-who-is-in-charge-01.svg)

Real example from my audit. The host has a Linux desktop with xrdp listening on 3389, and there was an `input` rule accepting 3389, documented, intentional, with a comment and everything. There was also, somewhere else entirely, a static DNAT sending 3389 arriving on the LAN port straight to the Windows VM. Guess who wins. The destination address is rewritten in `prerouting`, so by the time the routing decision happens the packet is no longer "for me", it goes down the `forward` branch, and the `input` rule is never even consulted. That accept rule had been dead code for who knows how long, and nothing in the config would ever tell you: it is not wrong, it is just unreachable.

Whenever a rule "obviously should work" and doesn't, check whether an earlier stage already made the decision for you.

**Two: several base chains can attach to the same hook.** Yours at priority -10, Docker's at 0, libvirt's somewhere in between, all on `forward`, all evaluated one after the other. And that leads to the part that actually bit me.

## "accept" does not mean accept

Here is the bit that is genuinely counterintuitive if you come from the mental model of "first match wins, done".

An `accept` verdict terminates **the current chain**, not the packet's journey. It means "this chain has no objection". The packet then carries on to the next base chain registered on the same hook. Only `drop` or `reject` ends the story immediately.

![Desktop View](/assets/img/firewall-who-is-in-charge-02.svg)

Put differently: with several owners on the same hook, and at least one of them running a default-drop policy, your packet has to survive **every** chain along the way, not just find one that likes it.

Which explains something I found in the original configuration and initially took for sloppiness. Traffic forwarded to the Windows VM was accepted in three different places: a dedicated table at priority -10, again in a chain at priority -5, and once more, in a generic form without the specific destination address, in the ordinary `forward` chain at priority 0. Three layers of belt and braces for the same packets.

I spent a good while trying to establish which of the three was actually load-bearing, and I could not prove it. Not from reading the ruleset, anyway: the interaction between chains from different families (`inet` versus `ip`), owned by different daemons, attached to the same hook at different priorities, with Docker and libvirt free to insert their own default-drop chains in the middle at any time, is genuinely hard to reason about without an end-to-end test for every path. The cost of keeping the redundant rules is zero. The cost of removing one and being wrong is a VM that silently becomes unreachable at some later reboot, probably while I am not looking.

So they stayed, with a comment explaining exactly that. In a layered firewall, prudence sometimes beats purity, and I have made my peace with it.

## The ruleset lives in the kernel; your files are only an opinion

This is the real lesson of the day, and the one that generalises well beyond this machine.

`nft list ruleset` shows you what is in the kernel *right now*. Your `/etc/` files show you what somebody intended at some point. On a box with Docker or libvirt those two things drift apart constantly and by design, because those daemons write their rules directly into the kernel and never to a file.

![Desktop View](/assets/img/firewall-who-is-in-charge-03.svg)

Now the classic trap, which has happened on this host in the past, before my time on it. You want your rules to survive a reboot, so you do the obvious thing:

```shell
$ sudo nft list ruleset > /etc/nftables.conf
```

Looks reasonable. It is a disaster waiting for the next service restart. That dump contains **everything that was in the kernel at that instant**, including Docker's dynamically managed tables, frozen as a static snapshot. And a config file loaded with `nft -f` typically starts with `flush ruleset`, which wipes *every* table in the kernel regardless of who created it. Reload that file while Docker is running and the container NAT and port-forwarding rules evaporate, until Docker is restarted (which, without `live-restore`, restarts every container with it).

The fun part is that I proceeded to make a variant of exactly the same mistake, that same afternoon, while carefully documenting how to avoid it. My new `/etc/nftables.conf` started with a `flush ruleset`, and I had not yet discovered that on this box the firewall service does not load that file directly, it loads Beckhoff's `/etc/nftables-bhf.conf`, which *includes* it and then includes the modular config directory again on its own. Net result: every rule applied twice, and the flush in the middle propagating right through Docker's and libvirt's tables. Nine containers restarted. Knowing the rule is not the same as noticing you are breaking it.

There is a debugging trap hidden in the aftermath, too. After an accidental flush, `docker ps` happily shows every container `Up` — container network namespaces are not destroyed by a host-level flush, the processes are perfectly alive. That tells you nothing about whether their NAT rules exist. Check `nft list tables` for the presence of Docker's tables, and test connectivity that actually matters on that network, not a reflex ping to a public DNS resolver on a box that has no internet access by design.

## The elegant pattern that was already there

The best part of the day was discovering that the machine already contained the solution, shipped by Beckhoff as part of the TwinCAT Linux RT image, and undocumented anywhere I had looked. A vendor systemd drop-in quietly points the firewall service at `/etc/nftables-bhf.conf` instead of the `/etc/nftables.conf` everybody assumes is in charge.

Instead of a global `flush ruleset`, that orchestrator file does a surgical, per-table reset:

```
table inet filter { }
delete table inet filter
table inet custom_routing { }
delete table inet custom_routing
```

The trick is delightful once you see it: creating a table that already exists is a no-op, and deleting one that has just been guaranteed to exist always works. So the pair means "reset **this** table, whether or not it was there", with no effect whatsoever on tables belonging to anyone else. Docker and libvirt never notice a firewall reload. That is precisely the behaviour everybody wants and almost nobody writes.

It comes with an obligation, though, and I learned this one the hard way as well: **every new table of your own has to be added to that list**. `nft -f` does not replace a chain's contents, it appends to them. A table that is not in the surgical-delete block simply gets the same rules added again on each service restart. That is what happened to a table I had added earlier in the day: after one restart, identical rules with two different handles, quietly duplicated. Harmless in behaviour, poisonous for your ability to trust what you are reading. My idempotency check is now boring and mandatory: restart the service twice, count the rules, the number must not move.

## Nobody decided who boots first

The last piece is the least technical and probably the most important.

On that host, the firewall service, Docker and libvirt had no ordering relationship at all. No `After=`, no `Wants=`, nothing. Which means the boot order was whatever systemd felt like that morning.

Why that matters: I found the MQTT port 1883 claimed by **two** independent DNAT rules. One written by hand, forwarding the port to the Windows VM. One created automatically by Docker, publishing the port of a broker container. Both attached to `prerouting`, both at the same NAT priority. There is no tie-break rule here that you can reason about from the config: the winner is whoever registered first in the kernel, on that particular boot. It was Docker when I checked, with absolutely no guarantee it would still be Docker after the next power cycle.

A firewall whose behaviour depends on service start order is not a firewall, it is a coin flip with good documentation. The fix was boring: pick an owner for the port explicitly (the container got it, the VM would need a different external port), and give the daemons an explicit ordering so the static rules always load before Docker and libvirt add theirs.

## Blast radius: the VM that stopped by itself

One last anecdote, because it illustrates how far a "local" change can travel.

libvirt supports lifecycle hook scripts, run automatically when a guest starts, stops, or is prepared. On this host the hook adds and removes the VM's port-forwarding rules, so they exist exactly as long as the VM does. Nice design, no stale rules.

During my accidental flush, libvirt was restarted. The hook duly ran its cleanup step for the "stopped" event, tried to delete iptables rules that the flush had already vaporised, got the classic `iptables: Bad rule (does a matching rule exist in that chain?)`, returned non-zero — and the VM stopped as a side effect. Clean shutdown, no data loss, but definitely not something I asked for.

Nothing was wrong with the hook. It simply assumed the world it had left behind was still there. Automation that nobody is watching will faithfully turn your small mistake into a bigger one.

## What I would tell myself that morning

- Find out **how** the firewall service actually loads its config before you edit anything. `systemctl cat nftables.service` takes five seconds and can save you a very long afternoon. The file you assume is the entry point may just be an include — on this Beckhoff TwinCAT Linux RT image it is `/etc/nftables-bhf.conf`, not the `/etc/nftables.conf` everyone reaches for first.
- Never dump the live ruleset into a static config file. Static files describe only what you own; let the other daemons manage their own tables.
- Prefer surgical `table X { }` / `delete table X` over a global `flush ruleset`, and keep that list in sync with every table you add.
- Restart your firewall service twice in a row and count the rules. If the count grows, you are appending, not reloading.
- Make service ordering explicit. Firewall first, then the daemons that inject their own rules.
- Remember that `accept` means "no objection here", not "you're through", and that NAT decides where a packet is going before anything decides whether it is allowed.
- After any flush, verify the tables, not the process list. `docker ps` will lie to you with a straight face.

Reading `nft list ruleset` on a box like this feels a bit like reading a shared document with three authors and no track changes. Understanding who wrote which paragraph, and when, turned out to be far more valuable than any individual rule.
