# Changelog

## [0.1.2] - 2026-07-03

### Fixed

- Add `ansible.builtin.apt` cache refresh (Debian family, `cache_valid_time: 3600`)
  before the `iptables-persistent` install task. Removes implicit dependency on the
  `ocserv` role running first to warm the apt cache; the `nat` role now manages its
  own index state and is safe to run in any order.

## [0.1.0] - 2026-07-02

### Added

- Initial release: ip_forward sysctl, iptables MASQUERADE NAT, iptables-persistent
