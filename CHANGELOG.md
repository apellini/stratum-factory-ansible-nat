# Changelog

## [0.2.0] - 2026-07-03

### Added

- Internal-subnet route management (`nat_internal_route_manage: true`): installs an
  explicit main-table route for `nat_internal_cidr` via `nat_internal_interface` so
  bastion-originated traffic to internal nodes (ProxyJump inner hop to k3s at 10.100.0.3)
  egresses the internal NIC (`eth1 src 10.100.0.2`) instead of falling through to the
  `eth0` default route and being MASQUERADE'd. Fixes the 300 s `wait_for_k3s_ssh`
  timeout on the two-VPC / multi-NIC bastion (D-INFRA-20).
- Systemd oneshot unit (`stratum-internal-route.service`) persists the route across
  reboots; skipped when `ansible_facts.service_mgr != 'systemd'` (molecule containers).
- New defaults: `nat_internal_route_manage`, `nat_internal_ip`, `nat_internal_route_unit`.
- New template: `templates/stratum-internal-route.service.j2`.

## [0.1.3] - 2026-07-03

### Fixed

- Replace `when: ansible_os_family == "Debian"` with `when: ansible_facts['os_family'] == "Debian"`
  in the apt cache-refresh task. Top-level `ansible_*` fact injection (`INJECT_FACTS_AS_VARS`)
  is deprecated in ansible-core and will be removed in 2.24. Using `ansible_facts['os_family']`
  is the correct forward-compatible form.

## [0.1.2] - 2026-07-03

### Fixed

- Add `ansible.builtin.apt` cache refresh (Debian family, `cache_valid_time: 3600`)
  before the `iptables-persistent` install task. Removes implicit dependency on the
  `ocserv` role running first to warm the apt cache; the `nat` role now manages its
  own index state and is safe to run in any order.

## [0.1.0] - 2026-07-02

### Added

- Initial release: ip_forward sysctl, iptables MASQUERADE NAT, iptables-persistent
