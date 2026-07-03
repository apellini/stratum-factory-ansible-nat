# Changelog

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
