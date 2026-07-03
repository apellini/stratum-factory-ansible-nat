# stratum_factory.nat

Ansible role that configures iptables NAT / IP-masquerade on the STRATUM dev bastion (Ubuntu 24.04).
It replaces a bash startup script and provides the same behaviour idiomatically, with interface names
either auto-detected from Ansible facts or supplied explicitly.

## What it does

1. Enables IPv4 forwarding persistently via `sysctl` (writes a drop-in under `/etc/sysctl.d/`).
2. Installs `iptables-persistent` (non-interactive).
3. **Auto-detects the external and internal network interfaces** from Ansible gathered facts
   (or uses explicit overrides when set).
4. Adds an iptables `MASQUERADE` rule on the external interface (`nat` table, `POSTROUTING` chain).
5. Adds `FORWARD` rules so packets from the internal VPC reach the internet and return traffic is allowed.
6. Persists rules to `/etc/iptables/rules.v4` via the `persist iptables` handler.
7. **Installs an explicit main-table route** for `nat_internal_cidr` via the internal interface
   so bastion-originated traffic to internal nodes egresses the internal NIC (see *ProxyJump note*
   below). Persisted across reboots via a systemd oneshot unit.

## Requirements

### Collections

```yaml
collections:
  - ansible.posix   # provides the sysctl module
  - ansible.builtin # provides package, iptables, shell
```

Install with:

```bash
ansible-galaxy collection install ansible.posix
```

`ansible.builtin` ships with every Ansible installation.

### Target host

- Ubuntu 24.04 LTS (GCP NIC naming: `ens4` for nic0/external, `ens5` for nic1/internal)
- Ansible connection user must have `root` or full `sudo` privileges
- Two network interfaces: one external (internet-facing), one internal (VPC-facing)
- `gather_facts: true` (the default) must be in effect — the role reads `ansible_facts`

## Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `nat_external_interface` | str | `""` | Interface connected to the internet. Empty = auto-detect from `ansible_facts.default_ipv4.interface`. |
| `nat_internal_interface` | str | `""` | Interface connected to the internal VPC. Empty = auto-detect via interface holding `nat_internal_ip`. |
| `nat_internal_cidr` | str | `10.100.0.0/24` | CIDR of the internal subnet. Used in route management and verify assertions. |
| `nat_sysctl_conf` | str | `/etc/sysctl.d/99-stratum-nat.conf` | Path to the sysctl drop-in file for persistent ip_forward. |
| `nat_internal_route_manage` | bool | `true` | When true, install an explicit route for `nat_internal_cidr` via the resolved internal interface. |
| `nat_internal_ip` | str | `""` | Source IP for the route (empty = kernel picks). Also drives internal interface auto-detection. |
| `nat_internal_route_unit` | str | `/etc/systemd/system/stratum-internal-route.service` | Path for the systemd oneshot unit that persists the route. |

## Interface auto-detection

Ubuntu 24.04 on GCP uses predictable NIC names (`ens4`, `ens5`, …) rather than the legacy
`eth0`/`eth1` names. With the default empty strings, the role auto-detects both interfaces:

- **External** — the interface that carries the default IPv4 route
  (`ansible_facts.default_ipv4.interface`). On GCP Ubuntu 24.04 this is `ens4`.
- **Internal** — the interface whose configured IPv4 address equals `nat_internal_ip`.
  On the STRATUM dev bastion this is `ens5` (static IP `10.100.0.2`).

An `assert` task fails early with a descriptive message if detection produces an empty
name (e.g. `nat_internal_ip` not set and `nat_internal_interface` left empty).

Explicit values always take precedence — useful for molecule tests, non-GCP hosts, or
any environment where the naming convention differs.

## ProxyJump note (two-VPC / multi-NIC bastion)

On a GCP dual-NIC VM, the secondary interface's subnet route may not appear in the main
routing table. Without it, traffic that the **bastion itself originates** to an internal
address (e.g., `10.100.0.3`) falls through to the default route, gets MASQUERADE'd,
and never reaches the target — causing a ProxyJump inner hop timeout.

`nat_internal_route_manage: true` (the default) fixes this by running:

```
ip route replace 10.100.0.0/24 dev <internal-iface> [src 10.100.0.2]
```

The route is applied live (idempotent) and persisted via a systemd oneshot unit so it
survives reboots within a session. Because the STRATUM dev bastion is recreated and
`site.yml` re-runs at every burn-up, the live-apply path is the primary mechanism; the
systemd unit is belt-and-suspenders.

## Example playbook

Auto-detect (recommended for STRATUM dev bastion on GCP Ubuntu 24.04):

```yaml
---
- name: Configure NAT on bastion
  hosts: bastion
  become: true

  roles:
    - role: stratum_factory.nat
      vars:
        nat_internal_cidr: "10.100.0.0/24"
        nat_internal_ip: "10.100.0.2"   # drives internal interface detection
```

Explicit interface override (molecule tests, non-GCP hosts):

```yaml
---
- name: Configure NAT on bastion (explicit)
  hosts: bastion
  become: true

  roles:
    - role: stratum_factory.nat
      vars:
        nat_external_interface: ens4   # GCP NIC0 — external VPC
        nat_internal_interface: ens5   # GCP NIC1 — internal VPC (10.100.0.x)
        nat_internal_cidr: "10.100.0.0/24"
        nat_internal_ip: "10.100.0.2"
```

## License

MIT

## Author

apellini — STRATUM Factory (`stratum_factory` namespace)
