+++
title = 'Running a home network with Ansible: inventory, secrets, push-to-deploy and a linter at zero'
date = 2026-12-13T12:00:00+01:00
draft = false
tags = ['ansible', 'automation', 'freebsd', 'linux', 'homelab']
summary = 'Two FreeBSD routers, a FreeBSD switch and a Linux HA pair, all configured from one Git repository: how it is organized, how secrets stay out of it, and how a push becomes a deployment.'
ShowToc = true
+++

Everything on my home network that isn't a phone or a laptop is configured
from a single Git repository: two FreeBSD routers in a CARP pair, a FreeBSD
box working as a switch, and two Debian servers in a keepalived pair. No host
is configured by hand anymore, at least not for long: anything changed
manually gets overwritten by the next run, which is exactly the point.

The only manual steps are the bootstrap: installing the operating system,
partitioning the disks, the initial network configuration, creating the
user Ansible connects as, and, on FreeBSD, installing Python with
`pkg install python3`, since Ansible needs it on every managed host, and
FreeBSD's base system doesn't include it. Everything after the first SSH
login is the playbooks' job.

This post is a tour of how that repository works, and of the habits that made
it pleasant to live with.

## Why bother, for a home network?

Two reasons, and they're the same ones that justify tools like Ansible
anywhere:

- **Everything is versioned.** Every change to every host is a commit, with
  a date and a message saying why. When something breaks, `git log` shows
  what changed, `git diff` shows exactly how, and going back is a revert,
  not an attempt to remember what a file looked like last month.
- **Everything is in one place.** The complete configuration of the network
  lives in a single repository, readable by whoever has access to it. Nothing
  depends on what's in one person's head, or scattered across hosts, so a
  dead disk is a reinstall and a playbook run, not an archaeology project.

Automation is the third benefit, and the one people usually mention first,
but it's really a consequence of the other two: once the configuration is
written down and versioned, applying it is the easy part.


## The layout

```
ansible-playbook/
├── ansible.cfg
├── inventory.ini
├── group_vars/
├── host_vars/
├── routers_playbook.yaml
├── switches_playbook.yaml
├── linux_ha_playbook.yaml
├── templates/
│   ├── routers/     # files only the routers get
│   ├── switches/
│   ├── linuxha/
│   └── shared/      # files every FreeBSD host gets
└── checks.sh
```

One playbook per kind of host, and a `templates` tree that mirrors each
host's filesystem: `templates/routers/etc/pf.conf` ends up as `/etc/pf.conf`
on the routers. Finding where a file comes from never requires a search.

The inventory is deliberately boring:

```ini
[freebsd]
router1 ansible_host=router1.example.com
router2 ansible_host=router2.example.com
[switches]
switch1 ansible_host=switch1.example.com
[linuxha]
server1 ansible_host=server1.example.com
server2 ansible_host=server2.example.com
```

and `ansible.cfg` points at it, so every command run from the repository
finds the right hosts without `-i`:

```ini
[defaults]
inventory = inventory.ini
```

## Host variables, and finding your peer

Anything that differs per host lives in `host_vars`: addresses on each
network, CARP priorities, disk labels. Templates then never contain
hostnames, which used to be my most common source of copy-and-paste bugs.

High-availability pairs have a recurring need: each node must know **the
other one**. Instead of hard-coding it, every template derives it from the
inventory:

```jinja
{% set peer = groups['freebsd'] | difference([inventory_hostname]) | first %}
```

On `router1`, that's `router2`, and vice versa. Sync scripts, monitoring and
failover tooling all use it, and adding a host to the inventory is the only
change a rebuild needs.

## Secrets never touch the repository

The repository is private, but secrets still don't belong in Git. Mine live
in **HashiCorp Vault**. The playbooks only ever see them as environment
variables:

```yaml
  vars:
    cloudflare_api_token: "{{ lookup('env', 'CLOUDFLARE_API_TOKEN') }}"
    carp_pass: "{{ lookup('env', 'CARP_PWD') }}"
```

The deployment runner, described below, unseals Vault, loads every secret
into its own environment, runs the playbooks, then seals Vault again.

Scripts on the hosts that need secrets don't get them inlined either: a
single template renders `/usr/local/etc/secrets.env`, readable by root only,
and each script sources it. One file per host, instead of tokens scattered
across a dozen scripts.

Because a run without the runner's environment would render every secret as
an empty string, each play starts with a guard:

```yaml
  pre_tasks:
    - name: Make sure the secrets are available
      ansible.builtin.assert:
        that: lookup('vars', item) | length > 0
        fail_msg: "Secret {{ item }} is empty: run through the deployment runner"
        quiet: true
      loop:
        - carp_pass
        - cloudflare_api_token
```

A run from a plain shell now stops before touching anything, instead of
deploying empty password files.

## Push to deploy

Deployments start with `git push`. GitHub sends a webhook, which reaches a
small receiver on the Linux pair through the reverse proxy, at an address that
follows the active node. The receiver then:

1. pulls the repository into a local clone;
2. fetches Vault's unseal keys and login credentials from AWS Parameter
   Store, unseals Vault and loads the secrets;
3. runs the three playbooks with `ansible-runner`, one after another;
4. in a `finally` block, deletes the temporary key files and seals Vault,
   whether the run succeeded, failed, or crashed.

Runs are serialized with a lock, so two quick pushes deploy one after the
other rather than in parallel. A scheduled nightly run catches anything that
drifted, and seals Vault again if something ever left it open.

## A linter at zero

For a long time, the only test was "it worked last time." Then I added a
`checks.sh`, run by a Git pre-commit hook:

- `ansible-playbook --syntax-check` on every playbook;
- `ansible-lint` with its **default** profile;
- `shellcheck` on every shell script in the repository;
- a Python syntax check on the helper scripts.

The first run reported **405** `ansible-lint` findings and **27** `shellcheck`
warnings. Most were mechanical (unquoted file modes, `yes` instead of `true`),
some were real: handlers restarting services through `command` instead of
the `service` module, a YAML block indented one level off, and two leftover
tasks that would have failed on an undefined variable. A few small scripts
did the bulk renames safely, like capitalizing handler names *and* every
`notify` pointing at them in one pass, then verifying each reference still
resolved.

It now reports zero, with no rules disabled, and the hook keeps it that way:
a commit that breaks a playbook simply doesn't happen.

## Takeaways

- **Versioned and in one place** is the real win: history, diffs and
  reverts for every change, and a network you can rebuild from the
  repository alone.
- **Mirror the filesystem in `templates/`.** The path tells you where a file
  goes.
- **Derive relationships from the inventory,** like an HA node's peer,
  instead of writing hostnames into templates.
- **Keep secrets out of Git and out of templates,** and make playbooks
  refuse to run without them.
- **Make cleanup unconditional.** Anything that unseals or writes secrets
  belongs in a `finally`.
- **Lint everything, then keep it at zero** with a pre-commit hook. The
  first run hurts; every run after that is free.
