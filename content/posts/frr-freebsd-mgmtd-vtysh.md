+++
title = 'From Quagga to FRR on FreeBSD: the mgmtd and vtysh -b gotchas'
date = 2026-12-06T12:00:00+01:00
draft = false
tags = ['freebsd', 'frr', 'quagga', 'rip', 'networking']
summary = 'Migrating a tiny RIP setup from Quagga to FRR looked trivial. Then FRR started, said it loaded the configuration, and did nothing.'
ShowToc = true
+++

My routers run a routing daemon for exactly one job. My ISP is **O2 Spain**,
Telefónica's second brand, which runs on **Movistar**'s network, and uses
the same setup: internet, TV and telephony each arrive on their own VLAN,
and the routes to the voice servers, on the "voip" VLAN, are announced
over **RIPv2**. The routers listen, learn those routes, and send voice traffic
the right way. That's it.

For years, I have had **Quagga** doing this job, but it also hasn't seen a release in a
long time; its successor, **FRRouting (FRR)**, is a fork that's actively
developed and packaged on FreeBSD.
Migrating a configuration this small should have taken ten minutes on paper, but it took a bit longer, because two things work differently in recent FRR versions, and neither fails loudly.

## The configuration

The routing part barely changes between the two. In FRR, everything lives in
one integrated file, `/usr/local/etc/frr/frr.conf`:

```
frr version 10
frr defaults traditional
hostname router1
log file /var/log/frr/frr.log
!
router rip
 version 2
 network voip
 passive-interface voip
exit
!
```

`network voip` enables RIP on the VoIP interface, and `passive-interface`
makes the router **listen without announcing** anything: I want Movistar's
routes, and Movistar has no business receiving mine.

Next to it, `vtysh.conf` tells FRR's shell to work with that single file:

```
service integrated-vtysh-config
```

And in `rc.conf`, Quagga's lines are replaced with FRR's:

```sh
frr_enable="YES"
frr_daemons="zebra ripd"
```

`zebra` manages the kernel's routing table, and `ripd` speaks RIP. I started
FRR, it reported loading the configuration, and the running configuration
was empty. No RIP, no routes, no errors.

## Gotcha 1: mgmtd

Recent FRR versions introduced **`mgmtd`**, a management daemon, and moved
configuration handling for some protocols into it. RIP is one of them:
**`ripd` no longer reads its configuration on its own**. Configuration
commands go to `mgmtd`, which validates them and hands them to `ripd`.

Without `mgmtd` running, nothing receives the `router rip` section. FRR's
tooling reports the configuration as sent, because it was, just to nobody
listening. The fix is adding it to the daemon list, first, since the other
daemons connect to it:

```sh
frr_daemons="mgmtd zebra ripd"
```

If you're migrating an older setup, it's worth checking the release notes of
your FRR version for which daemons are handled through `mgmtd`; the list has
grown over recent releases.

## Gotcha 2: who actually loads frr.conf?

With an integrated configuration, the individual daemons don't read
`frr.conf` themselves. It's **`vtysh -b`**, "boot," that reads the file and
pushes it into the running daemons. On FreeBSD, the port's rc.d script does
that only when asked:

```sh
frr_vtysh_boot="YES"
```

Without it, FRR starts with an empty configuration at every boot, which is
especially nasty because a manual `vtysh -b`, run while debugging, makes
everything look fixed until the next reboot.

## The final setup

```sh
frr_enable="YES"
frr_daemons="mgmtd zebra ripd"
frr_vtysh_boot="YES"
```

And three checks that show it actually works:

```sh
vtysh -c "show running-config"      # the router rip section is there
vtysh -c "show ip rip"              # routes learned from the ISP
sockstat -4l | grep ':520'          # ripd listening on the RIP port
```

The second one is the real proof: RIP-learned routes, with Movistar's
router as next hop, and the voice servers reachable through the VoIP VLAN.

## The supervisor

Quagga came with `watchquagga`, which restarted daemons that died. FRR's
equivalent, **`watchfrr`**, is started by the rc.d script along with the
daemons, so the separate `watchquagga_enable` and `watchquagga_flags` lines
simply go away.

## Takeaways

- **Quagga is unmaintained; FRR is its living successor,** and the
  configuration syntax carries over almost unchanged.
- **Recent FRR needs `mgmtd`** for protocols whose configuration it now
  manages, RIP included. Without it, configuration is silently ignored.
- **With an integrated `frr.conf`, `vtysh -b` loads it,** and on FreeBSD
  that means `frr_vtysh_boot="YES"`.
- **Check the running configuration, not the startup output.** "Done" only
  meant the commands were sent.
