# Changelog

## Unreleased
- Fix: a comma-separated `linux_hardening_ips_sshd_port` no longer disables the self-ban guard silently; the role fails if `ss` errors outside a container.
- Check mode: read-only queries run under `--check`; on a host without fail2ban only the install is previewed, and the re-audit is skipped.
- Task names in sub-task files are prefixed (`facts |`, `audit |`, `configure |`).
- CI runs the pre-commit hooks and a gitleaks history scan; the default scenario runs a check-mode pass.
- yamllint enforces a 120-character line limit and `true`/`false` booleans (as in kernel-hardening); long lines wrapped, `argument_specs` in block style. CONTRIBUTING matches the podman/pre-commit setup.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
