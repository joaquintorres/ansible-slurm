# Changelog

## Unreleased

### Changed defaults

- `slurm.conf`: the fallback controller line is now `SlurmctldHost=localhost`
  instead of `ControlMachine=localhost`. It is still emitted only when neither
  `SlurmctldHost` nor `ControlMachine` is set in `slurm_config`.
- `slurm.conf`: the default `CryptoType: crypto/munge` is replaced by
  `CredType`. `CredType` defaults to `cred/munge`, or to `cred/slurm` when
  `slurm_config.AuthType` is `auth/slurm`, as Slurm 23.11 and later require.
  If you set `CryptoType` in `slurm_config`, set `CredType` instead. Slurm
  prefers `CredType` when both are present.
- `slurmdbd.conf`: the default `AuthType` now follows the `slurm.conf`
  `AuthType` (still `auth/munge` unless you change it). Values set in
  `slurmdbd_config` still take precedence.

### Added

- The role now fails early, before writing any config file, if `slurm_config`,
  `slurm_cgroup_config`, or `slurmdbd_config` sets a parameter removed by
  Slurm 23.11 (for example `SelectType: select/cons_res`, `FastSchedule`, or
  `CgroupAutomount`). The message names the replacement, if there is one.
  The list is `__slurm_removed_params` in `vars/main.yml`.

### Platforms

- `meta/main.yml` now lists EL 8/9/10 and Ubuntu 24.04 (noble) instead of
  "all" for EL, Fedora, Ubuntu, and Debian. Ubuntu 22.04 is dropped because it
  ships Slurm 21.08. Debian and Fedora are no longer declared, but the Debian
  code paths are unchanged.

### Documentation

- The README lists supported versions and adds upgrade notes. The README
  example no longer sets the removed `CgroupAutomount` and `TaskAffinity`.
