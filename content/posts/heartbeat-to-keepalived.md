+++
title = 'From Heartbeat to keepalived: a failover story in five surprises'
date = 2026-12-20T12:00:00+01:00
draft = false
tags = ['linux', 'keepalived', 'heartbeat', 'drbd', 'high-availability']
summary = 'Replacing an unmaintained cluster manager on a two-node Debian pair: a notify script killed by systemd, a promotion race, an automation tool that cut off its own branch, and an application that was never meant for HA.'
ShowToc = true
+++

Two Debian servers on my network share a set of virtual IPs and a replicated
volume: whichever node is active owns the IPs, mounts the volume and runs the
services. For years, that was handled by **Heartbeat**, the classic Linux-HA
cluster manager, configured with an `haresources` file that said, in effect,
"on takeover: promote DRBD, mount `/srv`, start these services."

Heartbeat still works, but it hasn't been developed in a long time. Its
modern replacement for simple active/passive pairs is **keepalived**, which
speaks VRRP, moves virtual IPs, and runs a script on every state change.
The migration looked like an afternoon's work. It turned into a tour of
everything that can go wrong during a failover.

## The keepalived side

The configuration is short:

```
vrrp_script chk_named {
    script "/etc/keepalived/check_named.sh"
    interval 5
    timeout 3
    fall 2
    rise 2
}

vrrp_instance cluster {
    state BACKUP
    interface backplane
    virtual_router_id 20
    priority 150            # 100 on the other node
    advert_int 1

    unicast_src_ip 10.0.0.4
    unicast_peer {
        10.0.0.5
    }

    virtual_ipaddress {
        10.0.0.11/26 dev backplane
    }

    track_script {
        chk_named
    }

    notify /etc/keepalived/notify.sh
}
```

A few choices worth explaining:

- **Both nodes start as `BACKUP`;** the higher priority wins the election,
  and takes the IPs back when it returns, like Heartbeat's `auto_failback`.
- **Unicast** VRRP between the two nodes, instead of multicast.
- **A `virtual_router_id` distinct from anything else on the network.** My
  routers run CARP, which uses the same protocol family; colliding IDs make
  for very confusing packet captures.
- **A health check:** the IPs only stay on a node whose DNS server answers.

All the real work happens in `notify.sh`, which keepalived calls with the new
state: on `MASTER`, promote DRBD, mount `/srv`, start the services; on
`BACKUP` or `FAULT`, the reverse.

## Surprise 1: systemd killed the notify script

The first failover tests were inconsistent: sometimes services stopped
cleanly, sometimes halfway. The cause was systemd. When keepalived itself
stopped or restarted, systemd killed **every process in its control group**,
including a `notify.sh` that was still unmounting `/srv`.

A drop-in for keepalived's unit fixed it:

```ini
[Service]
KillMode=process
```

Now systemd only signals keepalived's main process, and lets the notify
script finish its work.

## Surprise 2: the promotion race

During a planned failover, the new master would sometimes fail to promote
DRBD. The old master was still stopping services and unmounting `/srv`, so
its DRBD resource was still `Primary`, and DRBD, correctly, refused a second
one.

The notify script now retries instead of failing immediately:

```sh
tries=0
until /usr/sbin/drbdadm primary r0 > /dev/null 2>&1; do
	tries=$((tries + 1))
	if [ "$tries" -ge 60 ]; then
		log "drbdadm primary r0 still failing after 60 seconds, giving up"
		exit 1
	fi
	sleep 1
done
```

The same pattern, with a timeout and then progressively stronger signals,
handles the unmount on the other side.

## Surprise 3: Ansible sawed off the branch it was sitting on

The repository that configures these servers was cloned onto the replicated
volume, under `/srv`. During one test, a playbook run disabled Heartbeat on
both nodes, which triggered a failover, which unmounted `/srv`, which removed
Ansible's own working directory mid-run. Both nodes ended up passive, and I
recovered them by hand.

The fix was simple once understood: **automation that manages the cluster
must not live on a cluster resource.** The deployment clones now live on each
node's local disk.

## Surprise 4: an application that takes its time

UniFi's controller ran from the replicated volume too, and it shuts down
slowly: Java and a database, inside containers, taking fifteen to twenty
seconds. Failovers stalled on `umount`, because UniFi was still holding files
on `/srv`.

Retries and longer timeouts helped, but the real lesson was the next one.

## Surprise 5: not everything belongs on shared storage

UniFi's controller assumes it owns one machine with fixed local paths. Every
HA arrangement around it, replicated volume included, is a workaround it
doesn't know about. So it moved **off** the replicated volume: each node has
its own local copy, only the active node runs it, and a nightly job stops it
briefly and copies the data to the standby with `rsync`. A failover can lose
up to a day of controller changes; the network devices keep working with
their last configuration regardless, which makes that an easy trade.

The move had a sting in its tail. The controller runs in rootless Podman,
and Podman records **absolute paths** for its volumes when it creates them.
Back when the data lived on the replicated volume, the paths were reached
through symlinks, which Podman resolved and stored. After the move, the
containers ran from local disk, but all seven volumes were still registered
under the old path, still on the replicated volume. A fresh install, and a
restore from the controller's own backup, finally put everything local.

## Takeaways

- **keepalived plus a notify script** covers simple active/passive pairs
  nicely, and is actively maintained.
- **`KillMode=process`** for keepalived, or systemd may kill your failover
  halfway through.
- **Retry the promotion,** since the old master may still be letting go.
- **Keep your automation off the resources it manages.**
- **Shared storage is for applications that tolerate it.** For the ones that
  don't, one active copy plus a cold sync is often the honest design.
