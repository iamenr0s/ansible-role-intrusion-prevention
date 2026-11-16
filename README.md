[![Molecule](https://github.com/iamenr0s/ansible-role-intrusion-prevention/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-intrusion-prevention/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_intrusion_prevention) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-intrusion-prevention/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-intrusion-prevention)

# Ansible Role: Intrusion Prevention

> Standalone copy of `intrusion_prevention` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables keep the collection-wide `linux_hardening_*` prefix.

Audit-first fail2ban for sshd on the Red Hat and Debian families (other families are **skipped** with a message and `linux_hardening_ips_findings: []`). It manages one jail drop-in (`/etc/fail2ban/jail.d/99-linux-hardening.local`) and the service. It does not manage the firewall (that is `firewall_hardening`), it only needs one that works.

## What it checks
| Check | Remediation |
|---|---|
| `fail2ban` installed | installed (EL: only with `linux_hardening_ips_enable_epel: true`, because it comes from EPEL) |
| Jail drop-in (0644): `ignoreip`, `bantime 1h`, `findtime 10m`, `maxretry 5`, `[sshd] enabled`, `port` | written from a template, backed up; running daemon reloaded |
| `fail2ban.service` enabled and active, sshd jail known to the daemon | enabled and started |
| The jail's **ban action has a working backend** (firewalld running for `firewallcmd-*`, `nft` for `nftables`, `iptables`) | **reported only**: the firewall belongs to `firewall_hardening` |

## Not locking out the administrator
- Loopback and `linux_hardening_ips_ignoreip` are written to the config.
- Every address with an **established ssh session** when the role runs is also exempted at runtime (`fail2ban-client set sshd addignoreip`), so the role cannot ban the machine configuring it. This is read with `ss`, not from `SSH_CONNECTION`, which sudo resets. It is deliberately **not** written to the config (addresses drift) and is lost when fail2ban reloads or restarts; running the role again restores it. Put the addresses you always administer from (bastions, VPN ranges) in `linux_hardening_ips_ignoreip`.

## Containers
The role writes the files and enables the unit but never starts fail2ban in a container: a ban is a rule in the **host** firewall. It reports `fail2ban running` as non-fixable there.

## Modes
`audit` (default) is read-only and sets `linux_hardening_ips_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`; after it only fixable findings fail the run.

## Example
```yaml
- hosts: all
  roles:
    - role: iamenr0s.linux_hardening.intrusion_prevention
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_ips_ignoreip: [10.0.0.0/8]
        linux_hardening_ips_enable_epel: true      # EL only
        linux_hardening_ips_sshd_port: "2222"
```

## Verification
Container scenarios `default` (package, drop-in, enable; no start), `audit`, `refuse`. **VM scenario `vagrant`** (libvirt, 2 GB; `sg libvirt -c "molecule test -s vagrant"`; box `MOLECULE_VAGRANT_BOX`, default `almalinux/9`): the jail runs, every current ssh peer (Ansible's included) is exempt, a manual ban lands in the jail **and in the firewall** (firewalld on EL, nftables on Debian), the sshd filter matches a failed-login line, a second run changes nothing, and everything is back after a reboot. It also checks that a ban action with no running backend is reported. Tested: AlmaLinux 9 and Debian 13 (VMs), AlmaLinux 9, Debian 13, Ubuntu 24.04, Fedora 44 (containers). CI runs the container scenarios only.

## Rollback
`systemctl disable --now fail2ban` and remove the drop-in (or restore its backup); `fail2ban-client unban --all` lifts current bans.

## Known gaps
- A ban is proven by `banip` and by the filter matching a sample line, **not** by really failing ssh logins until the jail trips.
- Only the sshd jail. No recidive jail, no `ignorecommand`, no mail actions; sshguard is not supported.
- Not tested on an Ubuntu or Fedora VM, EL8/EL10, or a non-default sshd port. CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-intrusion-prevention

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-intrusion-prevention
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_intrusion_prevention`
