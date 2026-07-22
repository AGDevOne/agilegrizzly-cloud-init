# Hetzner cloud-init

What [`hetzner-cloud-config.yaml`](../hetzner-cloud-config.yaml) does, and how it
is delivered to Hetzner Cloud nodes.

> This document is **Hetzner-specific**. Each cloud provider has its own
> `<provider>-cloud-config.yaml` and its own doc; pick the one that matches the
> node's provider.

## Why cloud-init runs *after* FDE on Hetzner

Hetzner nodes are provisioned in stages: `01_procure` (Terraform) creates the
server, then `02_early_boot` reinstalls the OS with LUKS full-disk encryption via
rescue-mode `installimage`. That reinstall **wipes the originally procured
image**, so cloud-init cannot be delivered through Hetzner's `user_data` field
(it would be destroyed by the reinstall).

Instead, the FDE post-install seeds a
[NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
datasource inside the freshly installed (encrypted) OS, so cloud-init runs on
that OS's **first real boot**.

```
01_procure ──► 02_early_boot (rescue + installimage, WIPE) ──► first boot ──► cloud-init
Terraform         seeds /var/lib/cloud/seed/nocloud/                          runs this config
```

## What the seeded user-data contains

The seed is a MIME-multipart document combining two parts:

1. **Public base config (this file).** For IPv4 nodes it is referenced with a
   cloud-init `#include` pointing at a **pinned commit** of this repo's raw URL:

   ```
   https://raw.githubusercontent.com/<owner>/<repo>/<commit-sha>/hetzner-cloud-config.yaml
   ```

   For IPv6-only nodes the file is embedded **inline** instead, because
   `raw.githubusercontent.com` has no reliable IPv6 and the boot-time fetch would
   fail.

2. **Private inline `#cloud-config`.** The admin user and its
   `ssh_authorized_keys`, generated from the encrypted secret store
   (`ssh.authorized_keys`). This part is never committed here.

cloud-init merges both parts.

## What this config does on the node

| Area | Action |
|------|--------|
| Packages | `package_update` + `package_upgrade`; installs base tools plus `ufw`, `fail2ban`, `unattended-upgrades` |
| SSH hardening | Writes `/etc/ssh/sshd_config.d/60-agilegrizzly-hardening.conf`: `PermitRootLogin prohibit-password`, `PasswordAuthentication no`, disables keyboard-interactive / X11 / agent forwarding, `MaxAuthTries 3` |
| Firewall | UFW default-deny incoming; allows SSH (22/tcp) and WireGuard (51820/udp). Kubernetes ports are opened later, scoped to the overlay, by the hardening stage |
| Brute-force protection | Enables `fail2ban` with an sshd jail |
| Patching | Enables `unattended-upgrades` |
| Time | Sets timezone (neutral `UTC` default) |

Root login is left as `prohibit-password` (key-only) here; the `05_finalize`
stage disables it fully once the unprivileged admin account is confirmed usable.

## Security notes

- **No secrets, ever.** This file is world-readable. No users, keys, passwords,
  tokens, or deployment-identifying IPs/hostnames belong here.
- **Pin to an immutable commit SHA.** The `#include` must reference a commit SHA,
  never a moving branch, so the boot-time fetch is reproducible and
  tamper-evident. Re-pin whenever this file changes.
- **Overlap with Ansible.** The Hetzner `bootstrap.yml` / `hardening.yml`
  playbooks cover the same ground and remain authoritative; overlapping steps are
  idempotent.

## Enabling it

Cloud-init seeding is opt-in. Set `settings.fde.cloud_init_url` in the inventory
to the pinned raw URL of `hetzner-cloud-config.yaml`. When unset, Hetzner nodes
are configured by the Ansible stages alone.
