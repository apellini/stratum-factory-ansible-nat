# stratum_factory.nat

Ansible role that configures iptables NAT / IP-masquerade on the STRATUM dev bastion (Ubuntu 24.04).
It replaces a bash startup script and provides the same behaviour idiomatically, with interface names as inputs.

## What it does

1. Enables IPv4 forwarding persistently via `sysctl` (writes a drop-in under `/etc/sysctl.d/`).
2. Installs `iptables-persistent` (non-interactive).
3. Adds an iptables `MASQUERADE` rule on the external interface (`nat` table, `POSTROUTING` chain).
4. Adds `FORWARD` rules so packets from the internal VPC reach the internet and return traffic is allowed.
5. Persists rules to `/etc/iptables/rules.v4` via the `persist iptables` handler.

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

- Ubuntu 24.04 LTS
- Ansible connection user must have `root` or full `sudo` privileges
- Two network interfaces: one external (internet-facing), one internal (VPC-facing)

## Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `nat_external_interface` | str | `eth0` | Interface connected to the internet. Used as MASQUERADE target and FORWARD out_interface. |
| `nat_internal_interface` | str | `eth1` | Interface connected to the internal VPC. Forwarded packets arrive here. |
| `nat_internal_cidr` | str | `10.100.0.0/24` | CIDR of the internal subnet. Informational; used in verify assertions. |
| `nat_sysctl_conf` | str | `/etc/sysctl.d/99-stratum-nat.conf` | Path to the sysctl drop-in file for persistent ip_forward. |

## Example playbook

```yaml
---
- name: Configure NAT on bastion
  hosts: bastion
  become: true

  roles:
    - role: stratum_factory.nat
      vars:
        nat_external_interface: ens4   # GCP NIC0 — external VPC
        nat_internal_interface: ens5   # GCP NIC1 — internal VPC (10.100.0.x)
        nat_internal_cidr: "10.100.0.0/24"
```

Using defaults (eth0/eth1) is sufficient for the STRATUM dev bastion:

```yaml
---
- name: Configure NAT on bastion (defaults)
  hosts: bastion
  become: true

  roles:
    - role: stratum_factory.nat
```

## License

MIT

## Author

apellini — STRATUM Factory (`stratum_factory` namespace)
