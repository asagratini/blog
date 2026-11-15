+++
title = "Why I chose FreeBSD (and where I didn't)"
date = 2027-01-10T12:00:00+01:00
draft = true
tags = ['freebsd', 'linux', 'networking', 'opinion']
summary = 'FreeBSD runs my routers and my switch; Debian runs the services next to them. Here is what FreeBSD does well, what it costs, and why the answer is "where it fits", not "instead of Linux".'
ShowToc = true
+++

Posts called "Why I chose X" are usually advocacy. This one tries not to
be. FreeBSD runs the most important machines on my home network, two
routers in a high-availability pair and a box working as a switch, and it
has earned that place. It also took me some time to get all the
configuration right, and there are a few things it simply doesn't do well.
Both sides belong in the answer.

## What works

### One system, released as a whole

FreeBSD's kernel and its base userland (the shell, the core tools, the
libraries, the network daemons) live in one source tree, are developed
together, and are released together. That sounds like an implementation
detail. In practice, it shapes everything else:

- **The documentation matches the system.** The Handbook and the man pages
  describe the tools that are actually installed, not one distribution's
  variant of them.
- **APIs and ABIs stay put within a major version.** The project doesn't
  break its programming interfaces (APIs) or its binary interfaces (ABIs)
  within a stable branch, so software built for 15.0 keeps running on every
  later 15.x release without being rebuilt.
- **The calendar is predictable.** A new minor release roughly every six
  months, each supported until three months after the next one, and the
  whole branch supported for a fixed period. From FreeBSD 15 onward that's
  **four years** from the .0 release (stable/15 is supported until the end
  of 2029); older branches got five.
- **Fixes come for the whole base at once.**
  [Security advisories and errata notices](https://www.freebsd.org/security/)
  cover the base system as a unit, and, since FreeBSD 15,
  **pkgbase** delivers the base system itself as binary packages, so
  updating it works like updating anything else.

### A clean line between the system and everything else

Everything that comes from ports and packages lives under `/usr/local`,
and nothing else does. The base system lives in `/` and the rest of
`/usr`; its configuration is in `/etc`, while every application's
configuration is in `/usr/local/etc`.

It's documented in `hier(7)`, and it's more than tidiness: when something
misbehaves, I know immediately whether it's part of the operating system
or something I installed, and upgrading one never quietly rearranges the
other.

### The network stack is in the box

Everything my routers rely on ships in the base system:

- **pf**, the firewall, with tables I update from scripts;
- **CARP**, for the shared addresses that move between the two routers;
- **HAST**, which replicates a disk between them for the database;
- **VLAN-filtering bridges** (new in FreeBSD 15) and **netgraph**, which
  together turned a small box into a proper switch.

None of these are separate projects I had to assemble, version-match, and
hope would keep cooperating. They're part of the same release, with the
same documentation.

### ZFS as a first-class citizen

ZFS isn't an add-on with licensing caveats on FreeBSD; it's integrated, and
well understood. Pools for package builds and for crash dumps are a couple
of commands each, and **boot environments** make upgrades reversible: if a
new release misbehaves, the previous system is one reboot away.

### Packages built my way

**Poudriere** builds my own package repository, with the options I want:
nginx with the GeoIP2 and traffic-status modules, for example. Both
routers build it, sign it, and install from it, so every machine runs
exactly the software I chose, configured at build time the way I want.

## What it costs

### Hardware support is narrower, and later

Drivers usually arrive on Linux first, sometimes much later on FreeBSD,
and some never arrive. I spent time on USB network adapters that never
behaved reliably before settling on proper PCIe cards. On server-class
hardware this rarely matters; in home and smaller-scale networks, it often
does. Check before you buy.

### A smaller ecosystem and a smaller community, which feed each other

These two belong together, because they're one loop. Fewer users means
fewer vendors ship FreeBSD versions of their software, fewer guides get
written, and fewer people can answer when something breaks. That, in turn,
keeps the user base smaller.

I feel that loop in three ways:

- **Software built around containers.** Several things I run are
  distributed as Linux container images, with no FreeBSD equivalent.
  That's a big part of why some of my services run on a pair of Debian
  machines next to the routers.
- **Searching for answers.** For an obscure problem, the Linux results
  come in pages; the FreeBSD ones, sometimes in a single forum thread from
  years ago.
- **Finding bugs yourself.** After upgrading to FreeBSD 15.0, HAST
  replication on my routers stopped working. It turned out to be a real
  regression, which I helped confirm on the project's bug tracker, and
  which was fixed in an errata notice. With fewer users, early adopters are
  more likely to be the ones who find these things. That's rewarding to fix,
  and costly when it happens on a Sunday.

## So, why FreeBSD?

Because I didn't choose it *instead of* Linux. I chose it **where it fits
best**: machines whose whole job is networking, where a coherent,
predictable system with pf, CARP, HAST and ZFS built in is exactly what I
want, and where the software I need is all there.

For services distributed as Linux containers, Debian, or any stable Linux
distribution, is the better tool, and that's what runs them. The two sit side by side on the same network,
each doing what it's good at.

## Takeaways

- **Coherence is FreeBSD's real feature:** one system, documented and
  released as a whole, with stable interfaces for years.
- **The `/usr/local` split** makes it obvious what's the system and what's
  yours.
- **For routers and appliances,** the built-in network stack and ZFS are
  hard to beat.
- **Budget for the costs:** narrower hardware support, a smaller ecosystem,
  and the occasional bug you'll be the one to find.
- **Choose for the job, not for the current trends.**
