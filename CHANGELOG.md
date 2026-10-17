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
