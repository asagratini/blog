+++
title = 'Contributing to FreeBSD as a user: an errata notice, a regression, a Handbook patch and a couple of ports'
date = 2027-01-03T12:00:00+01:00
draft = false
tags = ['freebsd', 'open-source', 'contributing']
summary = "You don't need to be a committer to improve FreeBSD. Most of my contributions started as \"this broke on my network,\" and ended somewhere useful."
ShowToc = true
+++

I'm not a FreeBSD developer. I'm a user who runs it on routers, a switch and
two other machines at home, and who occasionally hits something that
doesn't work. This post is about what happens next, because the path from
"this broke on my setup" to "this is fixed for everyone" is shorter than
many people think.

Here are four examples, each a different kind of contribution.

## 1. A bug in hastd, and an Errata Notice

My routers replicate a disk with **HAST**, FreeBSD's block-level
replication, which keeps a PostgreSQL database available across both
routers. After an upgrade to a more recent FreeBSD -RELEASE, replication stopped working:
the secondary side hung while setting up its connection, reading the socket type.

The first step was finding out whether it was me. Searching FreeBSD's
Bugzilla turned up **bug 292322**, "hastd failing to g_attach device on
FreeBSD 15.0," describing the same behavior. Adding my own observations, with
exact versions, logs and how to reproduce it, turned a single report into a
confirmed problem.

The fix shipped as **FreeBSD-EN-26:26.hastd**, an Errata Notice, which is
how FreeBSD delivers important fixes to supported releases outside the
regular release cycle. It went out to **15.1-RELEASE-p5**,
**14.5-RELEASE-p2** and **14.4-RELEASE-p11**, so anyone running a supported
release receives it through the normal update process.

Lessons from this one:

- **Search first.** Someone may have reported it already, and confirming an
  existing bug is as valuable as filing a new one.
- **Make it reproducible.** Versions, configuration, exact messages and the
  steps that trigger it are what turn a report into a fix.
- **Test the fix,** and say so in the report, so the people reviewing it know
  it works outside the developer's machine.

## 2. A regression in dhcpcd

dhcpcd isn't part of FreeBSD's base system, but it's what my routers use to
get IPv6 from my ISP. After upgrading to FreeBSD 15.1, **dhcpcd 10.5.0**
showed a regression, which I reported upstream, on the project's GitHub, as
**issue #691**, and which was fixed in **#694**.

The lesson here is about **where** to report. When software comes from the
ports tree, the right place is usually the upstream project, with the port
maintainer in the loop, since a fix upstream reaches every operating system
that uses it, and then flows back into the port with the next update.

## 3. A Handbook patch about CARP and spanning tree

The least glamorous contribution may be the most useful. My routers use
**CARP** for failover, and after a reboot, both of them briefly believed they
were the master. The cause turned out to be **spanning tree** on the switch
ports facing the routers: while a port moves through STP's listening and
learning states, which can take around 30 seconds, the rebooted router can't
hear its peer's advertisements, and promotes itself.

The fix is configuring those switch ports as **edge ports**, or disabling
STP on them. The FreeBSD Handbook's CARP chapter didn't mention it, so I sent
a patch adding a note, **PR 300522**, so the next person reading it doesn't
spend an evening rediscovering the problem.

Documentation patches are the easiest way to start contributing: no build
system, no tests, and every reader benefits.

## 4. Maintaining ports

I maintain eight ports in the FreeBSD ports tree, and every one of them
started the same way: I needed it, it wasn't there, or it was there and
nobody was looking after it.

| Port | What it is |
|---|---|
| `devel/py-oci` | Oracle Cloud Infrastructure's Python SDK |
| `devel/oci-cli` | Oracle Cloud's command-line tool, built on the SDK |
| `devel/rubygem-oci` | Oracle Cloud's Ruby SDK |
| `devel/py-circuitbreaker` | A Python implementation of the circuit breaker pattern, a dependency of the Oracle Cloud SDK |
| `devel/py-hvac` | A Python client for HashiCorp Vault |
| `devel/py-pyhcl` | A parser for HashiCorp's HCL configuration language |
| `databases/py-pymemcache` | A Python client for memcached |
| `math/mcsim` | GNU MCSim, a simulation package for statistical and differential-equation models |

The groupings tell the story. The Oracle Cloud ports came from a project that
needed them, and **`py-circuitbreaker`** came along because the SDK depends
on it. **`py-hvac`** is the library my own deployment tooling uses to unseal
Vault and fetch secrets, so I have a direct interest in it working.
**`mcsim`** is the odd one out, a scientific package I picked up and kept.

The Oracle Cloud ports also turned out to matter beyond my own use: FreeBSD's
developers used them to publish the **official FreeBSD images** on Oracle
Cloud, which is a nice illustration of how a small port can end up as part
of a project's tooling.

Maintaining a port means keeping it building, updating it when upstream
releases, and responding to problem reports. FreeBSD helps with the second
part: **portscout** scans upstream sources, and lists each maintainer's ports
with any newer versions it finds, so a quick look shows what's waiting for an
update. It also means knowing when to stop: handing a port back, when you no
longer have the time or the need, is a normal and healthy part of the
process. An unmaintained port is visible and fixable; a neglected one quietly
rots.

## Takeaways

- **You don't need a commit bit.** Bug reports, testing, documentation and
  ports are all real contributions.
- **The best bug report is a reproducible one,** and confirming someone
  else's report counts too.
- **Report to the right place:** FreeBSD's Bugzilla for the base system,
  upstream for software that comes from ports.
- **Documentation fixes save hours,** for people you'll never meet.
- **Maintain what you use, and hand it back when you stop.**
