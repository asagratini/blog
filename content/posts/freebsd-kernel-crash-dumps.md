+++
title = 'Kernel crash dumps on FreeBSD: swap as dumpdev, savecore into its own pool'
date = 2026-12-27T12:00:00+01:00
draft = false
tags = ['freebsd', 'kernel', 'debugging', 'zfs']
summary = 'When a FreeBSD box panics, a crash dump is the difference between a mystery and a bug report. Setting it up takes three lines, and one test you should run before you need it.'
ShowToc = true
+++

When a FreeBSD machine panics, it usually reboots and comes back as if
nothing happened. Without a crash dump, all you have is a gap in the logs and
a vague feeling. With one, you have the kernel's state at the moment it
died, and something a developer can actually work with.

I set this up properly on my routers recently, and it's one of those things
that takes ten minutes, and pays for itself the first time something goes
wrong.

## How it works

Crash dumps on FreeBSD are a two-step process, and understanding the two
steps makes everything else obvious:

1. **At panic time,** the kernel writes its memory to a **dump device**,
   usually the swap partition. The system is in no state to deal with
   filesystems at that point, so it writes raw blocks to a device it already
   knows about.
2. **At the next boot,** `savecore` finds the dump on that device, and
   copies it into a regular file, in `/var/crash` by default, before swap
   gets used again and overwrites it.

So you need a dump device big enough for the dump, and a directory with
enough room to keep it.

## Step 1: the dump device

In `/etc/rc.conf`:

```sh
dumpdev="/dev/gpt/swap0"
```

`AUTO` also works, picking the first swap device, but naming it explicitly
avoids surprises on machines with several disks. Using the **GPT label**
rather than a device name like `ada0p2` keeps it correct even if disks are
renumbered.

You can check what the kernel will use, right now, without rebooting:

```sh
dumpon -l
```

By default, FreeBSD writes **minidumps**, containing only kernel memory in
use, which keeps them far smaller than physical RAM. If space is tight,
`dumpon` can also compress them, with `-z` for gzip or `-Z` for zstd.

## Step 2: somewhere to keep them

`savecore` writes to `/var/crash`. A dump can be large, and I didn't want a
crash to fill the root filesystem at the worst possible moment, so on my
routers, `/var/crash` is its own small ZFS pool, called `crash`, mounted
there. A full pool then only means a missing dump, never a system that won't
boot properly.

`savecore` keeps track of what it has already saved through `bounds` and
`minfree` files in that directory; writing a number of kilobytes into
`/var/crash/minfree` reserves free space it will never use.

At boot, alongside `savecore`, the `crashinfo` script also writes a text
summary of each dump, `core.txt.N`, with the panic message and a backtrace,
which is often all you need to understand what happened, or to attach to a
bug report.

## Step 3: test it before you need it

The only way to know a crash dump setup works is to crash on purpose. On a
machine where that's acceptable, and preferably from the console:

```sh
sysctl debug.kdb.panic=1
```

The system panics immediately, writes the dump, and reboots. Afterward:

```sh
ls -l /var/crash/
less /var/crash/core.txt.0
```

You should see `vmcore.0`, `info.0`, and the `core.txt.0` summary, starting
with the panic message you caused. On a router pair, do it on the standby
node, and let the other one carry the network meanwhile.

## Reading a dump

The summary covers most cases. For a deeper look, `kgdb`, from the
`devel/gdb` package, opens the dump against the matching kernel:

```sh
kgdb /boot/kernel/kernel /var/crash/vmcore.0
(kgdb) bt
```

With FreeBSD's debug symbols installed, the backtrace shows function names
and source lines, which turns "the router rebooted" into a report someone can
act on.

## Takeaways

- **Two steps:** the kernel writes to `dumpdev` at panic time, and
  `savecore` copies it to `/var/crash` at boot.
- **Name the dump device by label,** and check it with `dumpon -l`.
- **Give `/var/crash` its own space,** so a dump can never fill the root
  filesystem.
- **Test with `sysctl debug.kdb.panic=1`,** on a node you can afford to
  reboot, before you need a dump for real.
