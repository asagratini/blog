+++
title = 'When CARP, HAST and Spanning Tree meet at boot'
date = 2026-11-08T12:00:00+01:00
draft = false
tags = ['freebsd', 'carp', 'hast', 'networking', 'high-availability']
summary = 'How a 30-second spanning-tree delay made both of my FreeBSD routers MASTER, and the hastd bug I found along the way.'
ShowToc = true
+++

My home network runs on two FreeBSD routers, `altair` and `rigel`, set up as
a highly available pair. It's more than a home network strictly needs, but it
doubles as a lab, and it has taught me more about failure modes than any
textbook. This post is about one of them: a race at boot time between CARP,
HAST and the switches' spanning tree, and a FreeBSD bug I ran into while
chasing it.

## The setup

The two routers share their duties with two FreeBSD building blocks:

- **CARP** (Common Address Redundancy Protocol) gives each network a
  virtual IP address. One router is `MASTER` and holds it; the other is
  `BACKUP`, ready to take over if the master stops sending advertisements.
- **HAST** (Highly Available Storage) replicates a block device between the
  two routers over a dedicated link. One side is `primary`, where the
  filesystem is mounted and services such as PostgreSQL run; the other is
  `secondary`, receiving every write.

The rule that holds it together: **the CARP master must also be the HAST
primary**, and there must only ever be one of each. A script called
`carpcontrol`, triggered by `devd` when CARP changes state, switches HAST's
role and starts or stops the services that live on it, much like the failover
example in the FreeBSD Handbook.

Each router connects to a UniFi USW-Aggregation switch with an LACP `lagg` of two
10G links, and the switches run Rapid Spanning Tree (RSTP), as most managed
switches do by default.

## The symptom

Every so often, after rebooting one of the routers, I'd find **both of them
`MASTER`**, both holding the virtual IPs, and, worse, **both HAST nodes
`primary`**: a split brain, with two copies of the data diverging.

Rebooting a router is precisely the moment a highly available pair is meant
to handle gracefully, so this was the opposite of what I'd built it for.

## What was actually happening

The culprit was **spanning tree**. When a switch port comes up, a classic
STP or RSTP port that isn't configured as an edge port goes through the
*listening* and *learning* states before it forwards traffic. With default
timers, that's **up to about 30 seconds**: twice the 15-second forward
delay.

During those 30 seconds:

1. The rebooting router's links are up, but the switch isn't forwarding its
   frames yet.
2. CARP starts, hears **no advertisements** from its peer, since they're
   being dropped by the switch, and concludes it's alone on the network.
3. It promotes itself to `MASTER`, and the `devd` script dutifully switches
   HAST to `primary`.
4. When the port finally starts forwarding, there are two masters and two
   primaries.

Nothing was wrong with CARP or HAST themselves: both did exactly what they
were designed to do with the information they had. They just had the wrong
information, for half a minute.

## The fix: edge ports

The ports facing the routers connect to **hosts**, not to other switches, so
there's no loop for spanning tree to protect against there. Configuring them
as **edge ports** (Cisco calls it *PortFast*) lets them forward immediately
when the link comes up, while spanning tree keeps protecting the links
between switches.

With that in place, CARP hears its peer from the first second after boot, the
rebooting router comes up as `BACKUP`, and HAST stays `secondary`, which is
how it should be.

Disabling spanning tree on those ports would work too, but edge ports keep
some protection in case someone ever plugs a switch where a router used to
be.

## Belt and braces: waiting for CARP to settle

Edge ports removed the cause, but the experience left me wary of a deeper
assumption: that the **first** CARP state a router sees after boot is the
right one. Links negotiating LACP, a slow switch, a peer still booting: there
are other ways for a router to spend a few seconds alone on the network.

So I added a small rc script, `carpsync`, that runs at boot after `devd`,
`hastd` and the filesystems, and makes the boot-time decision deliberately
instead of reacting to the first event:

1. **Start as HAST secondary.** If the peer is already primary, it can
   connect and keep replicating to us straight away, and `carpcontrol`
   refuses to become primary while that connection exists.
2. **Wait for the CARP state to settle:** the same value for 15 seconds in a
   row, giving up after 60 seconds and using whatever it last saw.
3. **Act once, on the settled state,** through `carpcontrol`, with a lock so
   it can't race the `devd`-triggered runs. On shutdown, it switches cleanly
   to `BACKUP`, so the peer takes over before the node goes away.

```sh
#!/bin/sh
# PROVIDE: carpsync
# REQUIRE: devd hastd FILESYSTEMS
# BEFORE: LOGIN
# KEYWORD: shutdown
. /etc/rc.subr
name="carpsync"
rcvar="carpsync_enable"
start_cmd="carpsync_start"
stop_cmd="carpsync_stop"

log="local3.debug"
hastresources="shared"
lock="/var/run/carpcontrol.lock"

carp_state()
{
	/sbin/ifconfig iot | /usr/bin/awk '/carp:/ { print $2; exit }'
}

carpsync_start()
{
	/usr/bin/logger -p $log -t $name "Boot sync: waiting for CARP state to settle"
	# 1. Start as HAST secondary, so a live primary on the peer can connect
	#    to us. carpcontrol MASTER waits while that connection exists.
	for disk in ${hastresources}; do
		/sbin/hastctl role secondary ${disk} || \
		    /usr/bin/logger -p $log -t $name "Unable to set ${disk} to secondary at boot"
	done

	# 2. Wait for the CARP state to settle: the same value for 15 seconds
	#    in a row, giving up after 60 seconds.
	stable=0; last=""
	for i in $(/usr/bin/jot 60); do
		state=$(carp_state)
		if [ "$state" = "$last" ]; then
			stable=$((stable + 1))
		else
			stable=0
		fi
		last=$state
		[ $stable -ge 15 ] && break
		sleep 1
	done

	[ $stable -lt 15 ] && /usr/bin/logger -p $log -t $name "CARP state did not settle within 60s, using $state"

	# 3. Act on the settled state: anything other than MASTER means BACKUP.
	[ "$state" = "MASTER" ] || state="BACKUP"
	/usr/bin/logger -p $log -t $name "Boot sync: CARP state is $state (stable for ${stable}s)"
	/usr/bin/lockf -k $lock /usr/local/sbin/carpcontrol "$state"
}

carpsync_stop()
{
	/usr/bin/logger -p $log -t $name "Shutdown: switching to BACKUP"
	/usr/bin/lockf -k $lock /usr/local/sbin/carpcontrol BACKUP
}

load_rc_config $name
run_rc_command "$1"
```

A few seconds of extra boot time is a small price for never again having to
untangle a HAST split brain by hand.

## The bug I found along the way

While testing all of this, with plenty of reboots, I noticed something else:
sometimes the pair came up **degraded**, with the primary working normally
but **no replication** to the secondary at all. Everything looked fine,
unless you checked `hastctl status` and saw the secondary wasn't receiving
anything.

It turned out to be a regression in `hastd`: when the two nodes started, the
secondary's worker process could **hang forever** waiting to read the socket
type from its peer, leaving the pair permanently degraded. That's exactly the
kind of silent failure that's dangerous in production: everything works,
until the day you need the secondary.

It's tracked in
[bug 292322](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=292322),
"hastd failing to g_attach device on FreeBSD 15.0", and the fix shipped as
[FreeBSD-EN-26:26.hastd](https://www.freebsd.org/security/advisories/FreeBSD-EN-26:26.hastd.asc),
an Errata Notice that also fixes `hastctl` commands failing on empty
responses, and uninitialized padding that could leak small amounts of data
into HAST metadata and network messages. It's corrected in **15.1-RELEASE-p5**,
**14.5-RELEASE-p2** and **14.4-RELEASE-p11**, so if you run HAST, a quick
`freebsd-version -ku` is worth it.

## Making sure the next person doesn't hit it

The spanning-tree interaction wasn't mentioned anywhere in the Handbook's
CARP or HAST sections, and it's easy to miss, since everything works once
the network has settled. So I submitted a small patch adding a warning to the
CARP section, with a cross-reference from the HAST section:
[PR 300522](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=300522).

## Takeaways

- **High availability is mostly about boot and failure timing.** The steady
  state is easy; the interesting bugs live in the first few seconds after a
  reboot.
- **Check every layer in the path.** CARP and HAST were configured correctly;
  the problem was two layers down, in the switch.
- **Ports facing hosts should be edge ports**, especially for anything that
  elects a master by listening for its peer: CARP, VRRP, keepalived, and
  friends.
- **Decide once, on settled state.** At boot, acting on the first event is
  fragile; waiting for the state to stay the same for a while is cheap
  insurance.
- **Report what you find.** A bug noticed on a home network turned into an
  Errata Notice that protects everyone running HAST.
