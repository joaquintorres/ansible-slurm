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

### Fixed

- Debian/Ubuntu: the default `SlurmdSpoolDir` is now `/var/lib/slurm/slurmd`,
  the directory the Debian `slurmd` package creates. It was
  `/var/lib/slurm/slurmctld`, the same directory as `StateSaveLocation`.
- `slurmdbd.conf`: the default PID file key is now `PidFile`. It was
  `SlurmctldPidFile`, which slurmdbd doesn't recognize.
- The slurmdbd cluster check now looks for `__slurm_cluster_name` in
  `sacctmgr -n -P list cluster format=Cluster` and no longer reports a change
  on every run. It used to compare the fourth space-separated field with the
  literal `"cluster"`.
- logrotate: the rotated path now follows the role's log directory. On current
  Debian/Ubuntu that is `/var/log/slurm`; it used to be hardcoded to
  `/var/log/slurm-llnl`.
- `meta/main.yml`: `min_ansible_version` is now `"2.10.4"`. The role's
  `ansible.builtin.include_tasks` tasks need it: Ansible 2.10.0–2.10.3
  recognize only the bare `include_tasks` name. The old value was `"2.5"`.
- `slurmdbd.conf` changes now restart slurmdbd instead of reloading it, so an
  `AuthType` change takes effect; `slurmdbd.conf(5)` requires a restart for
  that. Running jobs aren't interrupted: slurmctld queues accounting records
  while slurmdbd is down. A new "Restart slurmdbd" handler runs before the
  slurmctld and slurmd handlers.

### Added (auth/slurm)

- New variable `slurm_key`. When `slurm_config.AuthType` is `auth/slurm`, the
  role installs `slurm.key` in `slurm_config_dir` on controller, execution, and
  slurmdbd hosts (owner Slurm user, mode 0600) and restarts the daemons when
  it changes. If `slurm_key` is unset, it generates the key once on the control
  node as `slurm.key` next to the playbook (1024 random bytes, as in the Slurm
  authentication docs) and never regenerates an existing file.
