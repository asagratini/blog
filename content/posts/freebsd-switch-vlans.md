+++
title = 'A FreeBSD switch with VLANs: from netgraph to vlanfilter bridges'
date = 2026-11-15T12:00:00+01:00
draft = false
tags = ['freebsd', 'networking', 'vlan', 'netgraph', 'bridge']
summary = 'Turning a FreeBSD box into a VLAN-aware switch with a native VLAN on every trunk: netgraph on FreeBSD 14, a single bridge on FreeBSD 15, and the one gotcha that cost me an evening.'
ShowToc = true
+++

One of the machines on my network is a small FreeBSD box with four Intel
ports that works as a switch: it connects a few hosts, carries several
VLANs, and has its own management address. That's an ordinary job for a
managed switch, and, it turns out, a surprisingly interesting one for
FreeBSD, because the obvious approach doesn't do what a real switch does.

This post covers why, how I solved it with **netgraph** on FreeBSD 14, and
how FreeBSD 15's **VLAN-filtering bridges** made almost all of that
unnecessary.

## What a switch port actually does

On a managed switch, a port facing another switch or a server is usually a
**trunk**: it carries several VLANs **tagged** (each frame carries an 802.1Q
header saying which VLAN it belongs to), plus one **native** VLAN
**untagged** (frames without a tag belong to it). Cisco calls this
`switchport trunk native vlan`, and every managed switch supports it.

So the requirement is simple to state: on every port, untagged frames
belong to VLAN 10, and tagged frames for VLANs 20 and 30 belong to their
VLANs, all bridged between ports, and kept separate from each other.

## The classic approach, and its limit

The textbook way to bridge VLANs on FreeBSD 14 is **one bridge per VLAN**:
create a `vlan(4)` sub-interface for each VLAN on each port, and add all the
sub-interfaces of the same VLAN to the same bridge:

```sh
ifconfig em0.20 create vlan 20 vlandev em0
ifconfig em1.20 create vlan 20 vlandev em1
ifconfig bridge20 create addm em0.20 addm em1.20 up
```

That works nicely for **tagged** VLANs. The trouble starts with the native
one: untagged frames arrive on the physical interface itself, `em0`, so it
has to be a bridge member, while `em0` is also the parent of `em0.20` and
`em0.30`. A port can't cleanly be both a bridge member and a VLAN parent at
the same time, so a proper trunk with a native VLAN, the most standard
configuration there is, isn't really achievable this way.

## FreeBSD 14: netgraph to the rescue

**Netgraph** is FreeBSD's toolkit of small networking "nodes" you can wire
together in the kernel. Three of them are enough to build a real switch:

- **`ng_ether`** exposes each physical interface's raw traffic through a
  `lower` hook.
- **`ng_vlan`** sorts frames by tag: each configured VLAN goes out its own
  hook, and, crucially, everything **without** a matching tag goes out a
  hook called **`nomatch`**.
- **`ng_bridge`** is a learning bridge, one per VLAN.

The key idea: connect each port's `ng_vlan` node so that tagged VLANs go to
their bridges, and **`nomatch`**, the untagged traffic, goes to the native
VLAN's bridge. That's exactly a trunk with a native VLAN.

Here's the complete rc.d script, which builds this for every member port,
and adds the host's own interfaces:

```sh
#!/bin/sh
#
# Turn a FreeBSD host into a VLAN-aware switch using netgraph (ng_vlan +
# ng_bridge), with a native (untagged) VLAN on every port, and optional
# addresses on any number of VLANs. On FreeBSD 15+, the bridgevlan script
# (vlanfilter bridge) is the simpler alternative.
#

# PROVIDE: netgraph
# REQUIRE: devd netif
# BEFORE: pflog dhcpcd

# rc.subr reads name, desc, rcvar and the *_cmd variables
# shellcheck disable=SC2034

. /etc/rc.subr

name="netgraph"
desc="Bridges all the connections and sets VLANs using netgraph"
rcvar="netgraph_enable"

load_rc_config $name
: ${netgraph_enable:="NO"}
: ${netgraph_ifaces:="em0 em1"}         # member ports
: ${netgraph_nativevlan:="1"}           # untagged VLAN on member ports
: ${netgraph_taggedvlan:="2"}           # tagged VLANs, space-separated
: ${netgraph_vlanifs:=""}               # "name:vlan:addr[,addr...] ...", e.g.
                                        # "lan:1:192.168.0.2/24 mgmt:2:10.0.3.2/26,2001:db8::2/64"
: ${netgraph_gateway:=""}               # default route, if any

start_cmd="netgraph_start"
stop_cmd="netgraph_stop"
extra_commands="stats"
stats_cmd="netgraph_stats"

# Routine to verify node's existence.
netgraph_verify() {
	for vlan in ${netgraph_taggedvlan} ${netgraph_nativevlan}; do
		if ! /usr/sbin/ngctl info ${vlan}: >/dev/null 2>&1; then
			echo "${vlan}: bridge network not found"
			exit 1
		fi
	done
}

# Routine to get and display one bridge link's stats.
netgraph_linkstats() {
	stats=$(/usr/sbin/ngctl msg "$1": getstats "$2") || exit 1
	echo "${stats}" | /usr/bin/fmt 2 | /usr/bin/awk '/=/ { fl=index($0, "="); \
	    printf "%20s = %s\n", substr($0, 0, fl - 1), substr($0, fl + 1); }'
}

# Stats routine.
netgraph_stats() {
	# Make sure nodes exist.
	netgraph_verify

	for vlan in ${netgraph_taggedvlan} ${netgraph_nativevlan}; do
		echo ""
		echo "Statistics for bridging network ${vlan}:"
		echo ""
		linknum=0
		for iface in ${netgraph_ifaces}; do
			echo "Network interface ${iface}:"
			netgraph_linkstats ${vlan} ${linknum}
			linknum=$((linknum + 1))
		done
		for entry in ${netgraph_vlanifs}; do
			rest=${entry#*:}
			if [ "${rest%%:*}" = "${vlan}" ]; then
				echo "Local host interface ${entry%%:*}:"
				netgraph_linkstats ${vlan} ${linknum}
			fi
		done
	done
}

netgraph_start()
{
	# Reset all interfaces.
	netgraph_stop

	# Load netgraph KLD's as necessary.
	for kld in ng_ether ng_bridge ng_vlan ng_socket ng_eiface; do
		if ! /sbin/kldstat -v | /usr/bin/grep -qw ${kld}; then
			printf "Loading %s.ko... " "${kld}"
			/sbin/kldload ${kld} || exit 1
			echo "done"
		fi
	done

	# Verify all interfaces exist.
	for iface in ${netgraph_ifaces}; do
		if ! /usr/sbin/ngctl info ${iface}: >/dev/null 2>&1; then
			echo "Error: interface ${iface} does not exist"
			exit 1
		fi
		/sbin/ifconfig ${iface} up || exit 1
	done

	# Disables lro, tso and cxsum
	for iface in ${netgraph_ifaces}; do
		/sbin/ifconfig ${iface} -rxcsum -txcsum -tso -lro
	done

	linknum=0
	# Create bridges, vlans and attaches them to the bridges
	firstif=$(echo ${netgraph_ifaces} | /usr/bin/awk '{ print $1 }')
	for iface in ${netgraph_ifaces}; do
		/usr/sbin/ngctl mkpeer ${iface}: vlan lower downstream
		/usr/sbin/ngctl name ${iface}:lower vlan${iface}
		if [ "${iface}" = "${firstif}" ]; then
			for vlan in ${netgraph_taggedvlan}; do
				/usr/sbin/ngctl mkpeer vlan${iface}: bridge ${vlan} link${linknum}
				/usr/sbin/ngctl name vlan${iface}:${vlan} ${vlan}
				/usr/sbin/ngctl msg ${vlan}: setconfig "{debugLevel=1 loopTimeout=60 maxStaleness=60 minStableAge=1 }"
			done
			# Untagged frames go to the native VLAN's bridge
			/usr/sbin/ngctl mkpeer vlan${iface}: bridge nomatch link${linknum}
			/usr/sbin/ngctl name vlan${iface}:nomatch ${netgraph_nativevlan}
			/usr/sbin/ngctl msg ${netgraph_nativevlan}: setconfig "{debugLevel=1 loopTimeout=60 maxStaleness=60 minStableAge=1 }"
		else
			for vlan in ${netgraph_taggedvlan}; do
				/usr/sbin/ngctl connect vlan${iface}: ${vlan}: ${vlan} link${linknum}
			done
			/usr/sbin/ngctl connect vlan${iface}: ${netgraph_nativevlan}: nomatch link${linknum}
		fi

		for vlan in ${netgraph_taggedvlan}; do
			/usr/sbin/ngctl msg vlan${iface}: addfilter '{vlan='${vlan}' hook="'${vlan}'"}'
		done

		/usr/sbin/ngctl msg ${iface}: setpromisc 1
		/usr/sbin/ngctl msg ${iface}: setautosrc 0
		linknum=$((linknum + 1))
	done

	# One host interface per entry, attached to its VLAN's bridge on the
	# next free link, and named after its role
	for entry in ${netgraph_vlanifs}; do
		ifname=${entry%%:*}
		rest=${entry#*:}
		vlan=${rest%%:*}
		addrs=${rest#*:}
		/usr/sbin/ngctl mkpeer ${vlan}: eiface link${linknum} ether
		/usr/sbin/ngctl name ${vlan}:link${linknum} ng${ifname}
		ngif=$(/usr/sbin/ngctl msg ng${ifname}: getifname | \
		    /usr/bin/sed -n 's/.*Args:[[:space:]]*"\(.*\)"/\1/p')
		/sbin/ifconfig ${ngif} name ${ifname} > /dev/null 2>&1
		/sbin/ifconfig ${ifname} inet6 accept_rtadv -no_radr auto_linklocal
		for addr in $(echo "${addrs}" | /usr/bin/tr ',' ' '); do
			case "${addr}" in
			*:*)	/sbin/ifconfig ${ifname} inet6 ${addr} alias ;;
			*)	/sbin/ifconfig ${ifname} inet ${addr} alias ;;
			esac
		done
		/sbin/ifconfig ${ifname} up
	done

	if [ -n "${netgraph_gateway}" ]; then
		/sbin/route add default ${netgraph_gateway}
	fi
}

netgraph_stop()
{
	# Host interfaces first (destroying the eiface node removes the interface)
	for entry in ${netgraph_vlanifs}; do
		/usr/sbin/ngctl kill ng${entry%%:*}: >/dev/null 2>&1
	done

	for vlan in ${netgraph_taggedvlan} ${netgraph_nativevlan}; do
		/usr/sbin/ngctl kill ${vlan}: >/dev/null 2>&1
	done

	for iface in ${netgraph_ifaces}; do
		/usr/sbin/ngctl kill vlan${iface}: >/dev/null 2>&1
	done
}

run_rc_command "$1"
```

In `netgraph_start`, the first port creates the bridges, the others connect
to them, and each port's `ng_vlan` gets a filter per tagged VLAN. The host's own address lives
on an **`ng_eiface`** node, a virtual interface attached to the right VLAN's
bridge.

Two details matter in practice:

- **Promiscuous mode on, `autosrc` off:** the ports must accept frames for
  every MAC address, and must not rewrite source addresses on frames they
  forward.
- **Hardware offloads off:** checksum offloading, TSO and LRO don't mix with
  bridging in netgraph, so the script disables them on every member port
  (`-rxcsum -txcsum -tso -lro`).

It works, and it ran my network for a long time. But it's also several
dozen lines of `ngctl` calls to express something a switch does by default,
and the generalized version (any number of VLAN interfaces, any addresses,
proper teardown) grew to more than a hundred.

## FreeBSD 15: one bridge, VLAN-aware

FreeBSD 15 added **VLAN filtering** to the standard `if_bridge`. A single
bridge, created with `vlanfilter`, now understands 802.1Q itself, and each
member port is told which VLAN is untagged and which are tagged, exactly
like a managed switch's port configuration:

```sh
ifconfig bridge0 create vlanfilter
ifconfig bridge0 addm em0 untagged 10 tagged 20,30
ifconfig bridge0 addm em1 untagged 10 tagged 20,30
ifconfig bridge0 name switch
ifconfig switch up
```

That's the whole switch. The netgraph script, the bridge-per-VLAN pairs and
the `nomatch` trick all collapse into a handful of lines. My replacement
rc.d script takes its configuration from `rc.conf`:

```sh
bridgevlan_enable="YES"
bridgevlan_name="switch"
bridgevlan_ifaces="igc0 igc1 igc2 igc3"
bridgevlan_nativevlan="10"
bridgevlan_taggedvlan="20,30"
bridgevlan_vlanifs="mgmt:30:192.168.30.2/24"
bridgevlan_gateway="192.168.30.1"
```

And here's the script itself, which reads those settings:

```sh
#!/bin/sh
#
# Turn a FreeBSD 15+ host into a VLAN-aware switch using a single
# vlanfilter bridge, with optional addresses on any number of VLANs.
#

# PROVIDE: bridgevlan
# REQUIRE: devd netif
# BEFORE: pflog dhcpcd

# rc.subr reads name, desc, rcvar and the *_cmd variables
# shellcheck disable=SC2034

. /etc/rc.subr

name="bridgevlan"
desc="Bridges all the connections and sets VLANs"
rcvar="bridgevlan_enable"

load_rc_config $name
: ${bridgevlan_enable:="NO"}
: ${bridgevlan_name:="switch"}          # bridge name; "bridge0" skips the rename
: ${bridgevlan_ifaces:="em0 em1"}       # member ports
: ${bridgevlan_nativevlan:="1"}         # untagged VLAN on member ports
: ${bridgevlan_taggedvlan:="2"}         # tagged VLANs, comma-separated
: ${bridgevlan_vlanifs:=""}             # "name:vlan:address ...", e.g.
                                        # "lan:200:192.168.0.2/24 mgmt:203:10.0.3.2/26"
: ${bridgevlan_gateway:=""}             # default route, if any

start_cmd="bridgevlan_start"
stop_cmd="bridgevlan_stop"
extra_commands="stats"
stats_cmd="bridgevlan_stats"

# True if an interface with exactly this name exists
bridgevlan_ifexists()
{
	/sbin/ifconfig -l | /usr/bin/tr ' ' '\n' | /usr/bin/grep -qx "$1"
}

bridgevlan_start()
{
	# Reset all interfaces
	bridgevlan_stop

	/sbin/ifconfig bridge0 create vlanfilter
	for i in ${bridgevlan_ifaces}; do
		/sbin/ifconfig bridge0 addm $i untagged ${bridgevlan_nativevlan} tagged ${bridgevlan_taggedvlan}
	done
	if [ "${bridgevlan_name}" != "bridge0" ]; then
		/sbin/ifconfig bridge0 name ${bridgevlan_name}
	fi
	/sbin/ifconfig ${bridgevlan_name} up

	# One VLAN interface per entry, named after its role
	for entry in ${bridgevlan_vlanifs}; do
		ifname=${entry%%:*}
		rest=${entry#*:}
		vlan=${rest%%:*}
		addr=${rest#*:}
		/sbin/ifconfig ${bridgevlan_name}.${vlan} create inet ${addr}
		if [ "${ifname}" != "${bridgevlan_name}.${vlan}" ]; then
			/sbin/ifconfig ${bridgevlan_name}.${vlan} name ${ifname}
		fi
		/sbin/ifconfig ${ifname} inet6 accept_rtadv -no_radr auto_linklocal
	done

	if [ -n "${bridgevlan_gateway}" ]; then
		/sbin/route add default ${bridgevlan_gateway}
	fi
}

bridgevlan_stop()
{
	# VLAN interfaces first, then their parent bridge
	for entry in ${bridgevlan_vlanifs}; do
		ifname=${entry%%:*}
		if bridgevlan_ifexists "${ifname}"; then
			/sbin/ifconfig ${ifname} destroy
		fi
	done
	if bridgevlan_ifexists "${bridgevlan_name}"; then
		/sbin/ifconfig ${bridgevlan_name} destroy
	fi
}

bridgevlan_stats()
{
	/usr/bin/netstat -b -nI ${bridgevlan_name}
	for entry in ${bridgevlan_vlanifs}; do
		/usr/bin/netstat -b -nI ${entry%%:*}
	done
}

run_rc_command "$1"
```

Both scripts are plain rc.d scripts: drop them in `/usr/local/etc/rc.d/`,
make them executable, and enable **one** of them in `rc.conf`, since both
build the same switch from the same ports.

Both scripts are released under the **BSD 2-Clause license**, in keeping
with FreeBSD itself: use them, adapt them and share them as you like.

## The gotcha: don't put the address on the bridge

Here's the part that cost me an evening. With a classic bridge, you give
the bridge interface itself an address, and the host is reachable from every
member port. With a **VLAN-filtering** bridge, that stops working: an
address on `switch` itself was simply **unreachable** from hosts on the
untagged ports.

The fix, once I understood it, makes perfect sense: on a VLAN-aware bridge,
the host's address belongs to a **VLAN**, so it goes on a VLAN interface
**on top of the bridge**, like a switch's management VLAN interface:

```sh
ifconfig switch.10 create inet 192.168.10.2/24
```

That's what the `bridgevlan_vlanifs` setting above does for each entry: it
creates `switch.<vlan>`, assigns the addresses, and renames it after its
role, `mgmt` in this example. Once the address lived on `switch.10`, hosts
on the native VLAN reached it immediately.

## Takeaways

- **A native VLAN on a trunk is standard 802.1Q,** not an exotic setup, and
  it's the case the bridge-per-VLAN approach handles poorly.
- **On FreeBSD 14, netgraph does it properly,** with `ng_vlan`'s `nomatch`
  hook sending untagged traffic to the native VLAN's bridge.
- **On FreeBSD 15, use a `vlanfilter` bridge:** one bridge, per-port
  `untagged` and `tagged` settings, and far less to maintain.
- **Put host addresses on VLAN interfaces** (`switch.10`), never on a
  VLAN-filtering bridge itself.
