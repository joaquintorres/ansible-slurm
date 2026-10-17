# Tasks

Work top to bottom. Check a box only when its acceptance criteria are met
and the change is committed. Follow the rules in CLAUDE.md throughout.

---

## [x] T0 — Audit (no code changes)

Read the whole role: defaults, vars, tasks, templates, handlers, meta, and
any tests or CI. Record your findings in the **Findings** section at the
bottom of this file.

Answer these questions:

- How are `slurm.conf`, `cgroup.conf`, and `slurmdbd.conf` rendered? Free-form
  dicts, hardcoded keys, or both?
- Do any defaults or templates use `ControlMachine`, `ControlAddr`,
  `BackupController`, `select/cons_res`, `FastSchedule`, or `CryptoType`?
- Which `cgroup.conf` keys are set by default?
- Which config paths are used per OS? Is `slurm-llnl` still in use anywhere?
- In what order are the handlers defined? (Ansible runs handlers in
  definition order, not notification order.)
- What tests and CI exist, and which platforms do they cover?
- Which Slurm version does each target OS ship? Verify each one.
- What are the sources for each corresponding package?

**Acceptance:** The Findings section is filled in. If any later task doesn't
apply, mark it `N/A` with a one-line reason.

---

## [x] T1 — Installed version fact

After the package install, set a fact `__slurm_version` (for example,
`"26.05"`).

- Get the version from `ansible.builtin.package_facts` if the role already
  gathers it; otherwise run a `command` task with `changed_when: false`.
- Make it work in `--check` mode.

**Acceptance:** One or two tasks added; nothing else changed. Later tasks
compare against this fact with `is version(...)`.

---

## [ ] T2 — Config paths — N/A: supported releases already use /etc/slurm; no vars files

Make Debian and Ubuntu use `/etc/slurm` on the supported releases.

- Edit only the OS vars files.
- Leave any older-release entries as they are.

**Acceptance:** No task or template changes.

---

## [x] T3 — Modern `slurm.conf` defaults

Replace removed or renamed keys in role **defaults and templates only**:

- `ControlMachine` / `ControlAddr` / `BackupController` → `SlurmctldHost`
- `select/cons_res` → `select/cons_tres` (cons_res was removed in 23.11; verify)
- Drop `FastSchedule`.
- `CryptoType` → `CredType`, if it is present.

Don't change the templating mechanism. If the template derives the controller
name from inventory, keep that logic and change only the key it emits.

**Acceptance:** The rendered `slurm.conf` contains no removed keys when
defaults are used. CHANGELOG notes each changed default.

---

## [ ] T4 — `cgroup.conf` defaults for cgroup v2 — N/A: role sets no cgroup.conf keys

Remove default keys that current Slurm rejects or ignores under `cgroup/v2`.
Candidates include `CgroupAutomount`, `TaskAffinity`, and
`CgroupMountpoint`; verify each against the `cgroup.conf` docs.

Leave user-supplied keys untouched; T5 guards those.

**Acceptance:** The defaults diff only removes keys; nothing new is added.

---

## [ ] T5 — Guard against removed parameters

Add one dict to the existing `vars/main.yml`, named
`__slurm_removed_params`. It maps each removed key to its replacement, or to
`null` if it has none. Include only keys you verified in T3, T4, and the
Slurm NEWS file.

Add **one** `ansible.builtin.assert` task before the templating tasks. It
fails with a readable message if `slurm_config`, `slurm_cgroup_config`, or
`slurmdbd_config` contains a removed key.

**Acceptance:** One vars entry and one task are added. A deliberately bad
config fails with a message that names the replacement key.

---

## [ ] T6 — `slurmdbd.conf` permissions — N/A: already 0600, owner slurm user

Ensure `slurmdbd.conf` is mode `0600` and owned by the Slurm user. Change
only the existing template task's `mode` and `owner`.

**Acceptance:** The change is one or two lines.

---

## [ ] T7 — Handler order

Reorder the existing handler definitions so restarts run in this order:
slurmdbd, then slurmctld, then slurmd. Move blocks only; don't edit them.

If T0 found a handler that does `scontrol reconfigure` where a restart is
required, or the reverse, report it. Fix it only if the fix takes one line.

**Acceptance:** The diff consists of moved lines only, plus at most one
changed line.

---

## [ ] T8 — Platform metadata and CI matrix

Update the `platforms` list in `meta/main.yml`, and any existing CI matrix,
to the supported OS list in CLAUDE.md. Don't add new CI systems.

**Acceptance:** Only list entries change.

---

## [ ] T9 — Docs

In the README, update the supported Slurm and OS versions and add a short
"Upgrading from previous versions" note. That note lists each changed
default and explains the removed-parameter guard. Keep it under 30 lines.

**Acceptance:** README and CHANGELOG only.

---

## [ ] T10 — Optional: smoke test

Do this only if there are no tests at all. Add a single Molecule scenario
that runs a one-node, all-in-one cluster in a systemd-capable container,
with a verify step that runs `srun hostname`.

This is the one task allowed to add new files. They go under
`molecule/` only.

---

## Out of scope

Do not do any of the following, even if they look like improvements:

- Converting untouched code to FQCN
- Adding `argument_specs`
- Adding slurmrestd or JWT support
- Adding `auth/slurm`
- Adding configless mode
- Adding new installation sources
- Adding an upgrade playbook
- Templating new config files
- Refactoring

List anything you notice in this area under Findings → *Future work*.

---

## Findings

<!-- T0 writes here -->

Audited at commit 5c7a909 (2026-10-17).

### Repo vs. TASKS.md assumptions (mismatches)

- **No `vars/` directory exists.** All OS logic lives in `defaults/main.yml`
  as Jinja expressions (`__slurm_llnl`, `__slurm_debian`, `__slurm_redhat`).
  T2 ("edit only the OS vars files") and T5 ("the existing `vars/main.yml`")
  both assume files that don't exist.
- **No CHANGELOG file exists.** CLAUDE.md rule 2, T3 and T9 assume one.
  Creating it, or `vars/main.yml`, conflicts with rule 4 (no new files).
  This needs a decision before T3 or T5.
- **The goal (Slurm 23.11 or later) can't be met with distro packages on four
  of the target OSes.** See the version table. The role installs only from
  distro repos, and "new installation sources" is out of scope.

### Rendering

- `slurm.conf` is rendered from `templates/slurm.conf.j2`. It is a free-form
  dict, `__slurm_config_default | combine(slurm_config)`, plus one hardcoded
  key: `ControlMachine=localhost` is emitted when neither `ControlMachine`
  nor `SlurmctldHost` is set (template line 7). Nodes and partitions come
  from `slurm_nodes` and `slurm_partitions`, with hardcoded `localhost` and
  `debug` fallbacks.
- `cgroup.conf`, `acct_gather.conf`, and `slurmdbd.conf` are rendered from
  `generic.conf.j2` as free-form dicts. `cgroup.conf` and `acct_gather.conf`
  are written only if `slurm_cgroup_config` or `slurm_acct_gather_config` is
  defined.
- `gres.conf` is rendered from `gres.conf.j2` as a list of dicts.

### Removed or renamed keys in defaults and templates

- `ControlMachine`: template line 7 (fallback default). Still accepted by the
  `__slurm_config_merged` check on line 5.
- `CryptoType: crypto/munge`: `defaults/main.yml:47`.
- `ControlAddr`, `BackupController`, `select/cons_res`, `FastSchedule`: not
  used anywhere. The role sets no `SelectType` at all.

### cgroup.conf

- No keys are set by default; the file is entirely user-supplied. **T4 is N/A.**

### Paths per OS

The paths are computed in `defaults/main.yml`, not in vars files.

- `__slurm_llnl` is true for Ubuntu before 22 and Debian before 11. It drives
  `/etc/slurm-llnl`, `/var/log/slurm-llnl`, and `/var/lib/slurm-llnl`.
- On every target OS (Ubuntu 22.04/24.04, EL), `__slurm_llnl` is false, so the
  role already uses `/etc/slurm`. **T2 is N/A.**
- `slurm-llnl` is still hardcoded in `templates/logrotate.j2:5` for every
  Debian-family host, regardless of release. Today this is latent, because
  `slurm_rotate_logs` defaults to RedHat only.

### Handlers

`handlers/main.yml` defines them in this order:

1. Restart munge
2. Reload slurmdbd
3. Reload slurmctld
4. Restart slurmctld
5. Reload slurmd
6. Restart slurmd

That is already slurmdbd → slurmctld → slurmd. There is no "Restart
slurmdbd" handler.

Notify targets:

- `slurm.conf`: Restart slurmd and slurmctld.
- `cgroup.conf` / `gres.conf` / `acct_gather.conf`: Reload.
- `slurmdbd.conf` and cluster creation: Reload slurmdbd.

Possible mismatch, not yet verified against docs: `gres.conf` changes may
need a slurmd restart rather than a reconfigure. T7 can check this.
**T7's reorder is N/A**, but its review part still applies.

### slurmdbd.conf

`tasks/slurmdbd.yml:21-27` already sets `owner: __slurm_user_name`,
`group: root`, and `mode: 0600`. **T6 is N/A.**

### Tests and CI

- There is no Molecule scenario and no tests directory.
- CI has two workflows:
  - `lint.yml`: ansible-lint through reviewdog on `pull_request_target`.
  - `release.yml`: Galaxy import on tag.
- Neither has a platform matrix. The only platform list is in `meta/main.yml`
  (EL/Fedora/Ubuntu/Debian, all set to "all").

### Distro Slurm versions and package sources (verified 2026-10-17)

| OS | Version | Source |
|---|---|---|
| EL 8 | 20.11.9 | EPEL 8 (packages.fedoraproject.org) |
| EL 9 | 22.05.11 | EPEL 9 |
| EL 10 | 26.05.4 (26.05.5 in testing) | EPEL 10.3 / 10.4 |
| Ubuntu 22.04 | 21.08.5 | universe, `slurm-wlm` (packages.ubuntu.com) |
| Ubuntu 24.04 | 23.11.4 | universe |
| Debian 12 | 22.05.8 | main (packages.debian.org) |
| Debian 13 | 24.11.5 | main |

Only EL 10, Ubuntu 24.04, and Debian 13 ship 23.11 or later. Ubuntu 26.04
(25.11.2) is out but isn't in CLAUDE.md's target list. SchedMD publishes
source only; it has no distro repos.

The RedHat package list includes `munge` but not `munge-libs`. The Debian
list doesn't install `munge` explicitly; it relies on dependencies.

### Pre-existing bugs (not in any task; reported, not fixed)

- `defaults/main.yml:69-70`: Debian `SlurmdSpoolDir` is `.../slurmctld`, the
  same directory as `StateSaveLocation`.
- `defaults/main.yml:94`: the slurmdbd default key is `SlurmctldPidFile`, but
  slurmdbd.conf uses `PidFile`.
- `tasks/slurmdbd_cluster.yml:12`: it compares the output to the literal
  `"cluster"` instead of `__slurm_cluster_name`. Its first task also reports
  `changed` on every run.
- `meta/main.yml`: `min_ansible_version: "2.5"`, while the tasks use FQCN
  `ansible.builtin.*` (needs 2.10 or later).

### Status of later tasks

- T1: applies. The role doesn't gather `package_facts`, so use a `command`
  task.
- T2: N/A. Supported releases already use `/etc/slurm`, and there are no vars
  files.
- T3: applies. Template `ControlMachine` → `SlurmctldHost`; default
  `CryptoType` → `CredType`. Blocked on the CHANGELOG decision.
- T4: N/A. There are no default `cgroup.conf` keys.
- T5: blocked. `vars/main.yml` doesn't exist.
- T6: N/A. It is already mode 0600 and owned by the Slurm user.
- T7: N/A for the reorder. The review part (handler type per file) is still
  open.
- T8: applies, to `meta/main.yml` only. There is no CI matrix.
- T9: applies.
- T10: applies, since there are no tests at all.

### Future work

- logrotate.j2 hardcodes `/var/log/slurm-llnl` for all Debian-family hosts.
- An install source for Slurm 23.11 or later on EL 8/9, Ubuntu 22.04, and Debian 12 (out of scope: new installation sources).
- Ubuntu 26.04 and Debian 13 support, as candidates for the target list.
- auth/slurm: T3 derives CredType and the slurmdbd AuthType from `slurm_config.AuthType`, but the role neither distributes `slurm.key` (owned by SlurmUser, mode 0600) nor skips munge setup under auth/slurm. Out of scope ("Adding auth/slurm").
