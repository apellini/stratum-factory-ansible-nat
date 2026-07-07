# Changelog

## [0.4.3] - 2026-07-07

### Fixed

- iptables-services on RHEL/Rocky 9 ships a default `/etc/sysconfig/iptables` that
  sets `FORWARD DROP` policy and appends a `REJECT --reject-with icmp-host-prohibited`
  rule. After the `iptables` service starts, the role's own FORWARD ACCEPT rules were
  appended **after** the REJECT rule and therefore never reached, blocking all forwarded
  packets (k3s → bastion → internet). Added two tasks that run after service start and
  before the NAT rules: one removes the default REJECT rule (`state: absent`), the other
  sets the FORWARD chain default policy to ACCEPT. Both are idempotent and RHEL-only
  (D-INFRA-28).

## [0.3.0] - 2026-07-03

### Changed

- `nat_external_interface` and `nat_internal_interface` defaults changed from `eth0`/`eth1`
  to `""` (auto-detect). Ubuntu 24.04 on GCP uses predictable names (`ens4`/`ens5`), so
  hardcoded `eth*` names caused a "Cannot find device eth1" failure on the live bastion and
  silently-broken MASQUERADE/FORWARD rules on `eth0` (D-INFRA-21).

### Added

- Interface auto-detection block in `tasks/main.yml`: external iface resolved from
  `ansible_facts.default_ipv4.interface`; internal iface resolved by matching `nat_internal_ip`
  against gathered interface facts. Explicit overrides via `nat_external_interface` /
  `nat_internal_interface` still take precedence (exercised by molecule tests).
- `assert` task fails early with a descriptive message if resolution produces an empty name.
- Template `stratum-internal-route.service.j2` now uses the resolved `_nat_internal_iface`
  fact variable so the systemd unit references the actual kernel interface name.

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
