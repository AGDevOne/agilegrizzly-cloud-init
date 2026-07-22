# AgileGrizzly cloud-init (public)

Public, non-secret cloud-init base configuration for AgileGrizzly nodes.

> [!WARNING]
> This repository is **public and world-readable**. It MUST NEVER contain
> secrets: no users, SSH keys, passwords, API tokens, IP addresses, or anything
> that identifies a specific deployment. Everything here is safe to publish.

## Provider-specific configs -- pick the one that matches the node

Cloud-init behaviour differs per cloud provider (disk layout, network
interfaces, how the OS is installed, IPv6 reachability, etc.), so each provider
has its **own** config file and doc. Select the file that matches the node's
cloud provider when wiring the cloud-init `#include`:

| Provider | Config file | Docs |
|----------|-------------|------|
| Hetzner Cloud | [`hetzner-cloud-config.yaml`](hetzner-cloud-config.yaml) | [docs/hetzner-cloud-init.md](docs/hetzner-cloud-init.md) |

Additional providers get their own `<provider>-cloud-config.yaml` plus a matching
`docs/<provider>-cloud-init.md`. Do not use one provider's config on another
provider's nodes.

## What these are

Each `<provider>-cloud-config.yaml` is a [cloud-init](https://cloudinit.readthedocs.io/)
`#cloud-config` document that performs the non-secret base setup of a node:
package updates, base + security packages, SSH hardening drop-in, host firewall
(UFW), fail2ban, and unattended security upgrades.

## How they are consumed

During provisioning the node's disk is reinstalled with LUKS full-disk
encryption. The installer seeds a [NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html)
datasource so cloud-init runs on the encrypted OS's first boot. The seeded
user-data is a MIME multipart document combining:

1. **The public config for that provider**, referenced via an `#include` pointing
   at a **pinned commit** of this repo's raw URL, e.g.

   ```
   https://raw.githubusercontent.com/<owner>/<repo>/<commit-sha>/hetzner-cloud-config.yaml
   ```

   The include is always pinned to an immutable commit SHA (never a moving
   branch) so the boot-time fetch is reproducible and tamper-evident.

2. **A private inline `#cloud-config`** generated from the encrypted secret
   store, containing the admin user and its `ssh_authorized_keys`. That part is
   never committed here.

IPv4-less (IPv6-only) nodes cannot reach `raw.githubusercontent.com` (no
reliable IPv6), so for those nodes the provisioning layer embeds the contents of
the provider's config inline instead of using `#include`.

See the per-provider doc under [`docs/`](docs/) for exactly what each config does.

## Changing a config

Any change must be committed and pushed, and the consuming `#include` re-pinned
to the new commit SHA. Because this runs at first boot on regulated hosts, treat
it as a supply-chain surface: review changes carefully and never introduce a
secret.
