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

## [x] T5 — Guard against removed parameters

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

## [x] T7 — Handler order

Reorder the existing handler definitions so restarts run in this order:
slurmdbd, then slurmctld, then slurmd. Move blocks only; don't edit them.

If T0 found a handler that does `scontrol reconfigure` where a restart is
required, or the reverse, report it. Fix it only if the fix takes one line.

**Acceptance:** The diff consists of moved lines only, plus at most one
changed line.

---

## [x] T8 — Platform metadata and CI matrix

Update the `platforms` list in `meta/main.yml`, and any existing CI matrix,
to the supported OS list in CLAUDE.md. Don't add new CI systems.

**Acceptance:** Only list entries change.

---

## [x] T9 — Docs

In the README, update the supported Slurm and OS versions and add a short
"Upgrading from previous versions" note. That note lists each changed
default and explains the removed-parameter guard. Keep it under 30 lines.

**Acceptance:** README and CHANGELOG only.

---

## [ ] T10 — Molecule in GitHub CI

A Molecule scenario already exists in `molecule/default/` (commit 6ce62b8).
It runs a one-node controller and exec setup on Ubuntu 24.04, and its verify
step runs `srun hostname`. It has never run, because there was no Docker in
the authoring environment. Make it run in GitHub Actions:

- Add one workflow under `.github/workflows/` that runs `molecule test` on
  pull requests and on pushes to `main`. Don't add any other CI system.
- The workflow installs `ansible-core`, `molecule`, the `community.docker`
  collection, and the Docker SDK for Python. Pin action and tool versions.
- Run one matrix job per platform:
  - Ubuntu 24.04 (Slurm 23.11 from universe). This job is required.
  - EL 10 (Rocky 10). It is allowed to fail (`continue-on-error`) until T12
    switches it to OpenHPC 4. EPEL has no Slurm for released EL 10 (10.2);
    only EPEL 10.3/10.4 have 26.05.

  Select the image per job, for example through an environment variable
  read in `molecule.yml`. EPEL setup goes in `molecule/default/prepare.yml`,
  not in the role.
- Change the scenario only as far as it needs to pass. If a role bug blocks
  it, stop and report; T11 owns role fixes.

New files are allowed in `.github/workflows/` and `molecule/` only.

**Acceptance:** The Ubuntu 24.04 job passes on GitHub, including the
`srun hostname` verify step. Link the run in the report. If you can't
push or run it from this environment, say so and leave the box unchecked.

---

## [ ] T11 — Pre-existing bugs, AuthType restart, and slurm.key

Make each item its own commit (`slurm: T11a ...`, `slurm: T11b ...`), with
its own diff-budget check. Verify every Slurm claim as in CLAUDE.md rule 6.
Add CHANGELOG entries for any changed default or new variable.

- **T11a** — `defaults/main.yml`: the Debian `SlurmdSpoolDir` is
  `/var/lib/slurm/slurmctld`, the same directory as `StateSaveLocation`. Use
  the slurmd spool path that the Debian/Ubuntu package creates. Check the
  package to confirm it.
- **T11b** — `defaults/main.yml`: the slurmdbd default key
  `SlurmctldPidFile` should be `PidFile`. Confirm it in the `slurmdbd.conf`
  docs.
- **T11c** — `tasks/slurmdbd_cluster.yml`:
  - Compare the `sacctmgr` output with `__slurm_cluster_name`, not the
    literal `"cluster"`.
  - Make the check task report no change.
- **T11d** — `templates/logrotate.j2`: use `__slurm_log_dir` instead of
  hardcoding `/var/log/slurm-llnl` for every Debian-family host.
- **T11e** — `meta/main.yml`: raise `min_ansible_version` from `"2.5"` to
  the lowest version the role actually needs. The FQCN module names need
  2.10 or later; confirm whether anything else needs more.
- **T11f** — AuthType restart. Changing `AuthType` needs slurmdbd,
  slurmctld, and slurmd restarted, not reconfigured (scontrol and
  `slurmdbd.conf` docs).
  - `slurm.conf` changes already trigger restarts.
  - Add a "Restart slurmdbd" handler, defined before the slurmctld handlers
    so the order stays slurmdbd → slurmctld → slurmd.
  - Make a `slurmdbd.conf` change restart slurmdbd when `AuthType` changes.
    If that is not practical, restart slurmdbd on every `slurmdbd.conf`
    change and explain why in the report.
  - Keep the existing handler names.
- **T11g** — `slurm.key` for `auth/slurm`. When `AuthType` is `auth/slurm`,
  install `{{ slurm_config_dir }}/slurm.key`, owned by SlurmUser and mode
  0600, on every slurmctld, slurmd, and slurmdbd host. Check the Slurm
  authentication docs first.
  - Follow the `slurm_munge_key` pattern: a new variable `slurm_key` gives a
    path on the control node to copy.
  - If `slurm_key` is unset:
    - Generate one key once, as random bytes the way the docs describe.
    - Copy that same key to every host.
    - Never regenerate it on later runs.
    - Use `no_log` for the key contents.
    - Document where the generated key is kept on the control node.
  - A key change triggers restarts.
  - Document `slurm_key` in the README and CHANGELOG.
  - Don't add `slurm.jwks`, key rotation, sackd, or munge removal.

Put new tasks in the existing task files; T11 adds no new files.

T11f and T11g are likely to exceed the ~20-line budget. Stop and explain
before going over, as CLAUDE.md requires.

**Acceptance:** Each item is committed separately. The syntax check passes,
and lint is clean on changed lines. If T10's CI exists by then, it passes.

---

## [ ] T12 — OpenHPC packages on EL 8/9/10

EPEL ships Slurm 20.11 (EL 8) and 22.05 (EL 9), both below 23.11. OpenHPC
ships newer versions (checked in the repo indexes on 2026-10-17):

- OpenHPC 2.11, EL 8: Slurm 25.05.9.
- OpenHPC 3.6, EL 9: Slurm 25.11.8.
- OpenHPC 4.1, EL 10: Slurm 25.05.7. EPEL has no Slurm for EL 10.2, so on
  released EL 10 this is the only packaged source.

Package names are the same in all three: `slurm-ohpc`, `slurm-slurmctld-ohpc`,
`slurm-slurmd-ohpc`, `slurm-slurmdbd-ohpc` (plus `munge`).

- Add one public variable, `slurm_package_source`, with values `distro`
  (the default, unchanged behaviour) and `openhpc`. Validate it.
- Put the OpenHPC package map in `vars/main.yml`, next to the existing maps.
  Make `__slurm_packages` choose it when `slurm_package_source` is
  `openhpc`. This is data, not new tasks.
- The role doesn't set up the OpenHPC repository; the user does. Document it
  in the README: the `ohpc-release` RPM, EPEL, and PowerTools/CRB. Use the
  OpenHPC install guide for the exact URLs and confirm them.
- Check that the OpenHPC packages' paths match the RedHat defaults: config
  dir, StateSaveLocation, SlurmdSpoolDir, log and pid files, and the systemd
  unit names. If they differ, stop and report before adding more overrides.
- `__slurm_version` (T1) should read 25.05 or 25.11 on these hosts.
- Switch T10's `el10` CI job to OpenHPC 4: set up the repo in `prepare.yml`,
  set `slurm_package_source: openhpc`, and remove its `continue-on-error`.
  Optionally add an EL 9 + OpenHPC 3 job.

Covers EL 8, 9, and 10. This lifts the "new installation sources" exclusion
for OpenHPC only.
Don't add other sources, building from source, or repo management in the
role.

**Acceptance:** With the default, behaviour is unchanged. With `openhpc`,
the package tasks install the `-ohpc` packages. README and CHANGELOG
document the variable and the repo setup.

---

## Out of scope

Do not do any of the following, even if they look like improvements:

- Converting untouched code to FQCN
- Adding `argument_specs`
- Adding slurmrestd or JWT support
- Adding `auth/slurm` beyond T11g (no `slurm.jwks`, sackd, key rotation, or munge removal)
- Adding configless mode
- Adding new installation sources (except OpenHPC, in T12)
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

T7 review (checked against the 26.05.5 man pages and systemd units): every
"Reload" handler sends SIGHUP (`ExecReload=/bin/kill -HUP $MAINPID`). The
`gres.conf`, `cgroup.conf`, and `acct_gather.conf` docs say changes take
effect on SIGHUP or `scontrol reconfigure`, so Reload is correct for them.
Restart is a safe superset for `slurm.conf`. No one-line fix is needed.
Option-level exceptions remain, and none of them is a one-line fix:
- `gres.conf` `Cores` and `Type` need a slurmctld (and slurmd) restart.
- A slurmdbd `AuthType` change needs a restart, and there is no "Restart
  slurmdbd" handler.
- `scontrol reconfigure` can't change ports or `AuthType` (23.11+).

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
| EL 10 | none in EPEL 10.0–10.2; 26.05.4 only in EPEL 10.3 / 10.4 (corrected 2026-10-17) | EPEL. Rocky 10.2, the newest released EL 10, gets EPEL 10.2, which has no Slurm (first CI run: "No package slurm available"). |
| Ubuntu 22.04 | 21.08.5 | universe, `slurm-wlm` (packages.ubuntu.com). Dropped from the target list. |
| Ubuntu 24.04 | 23.11.4 | universe |
| Debian 12 | 22.05.8 | main (packages.debian.org) |
| Debian 13 | 24.11.5 | main |

Only Ubuntu 24.04 and Debian 13 ship 23.11 or later today. EL 10 does only once it reaches 10.3, or with OpenHPC 4 (`/OpenHPC/4/EL_10`, version not yet checked). Ubuntu 26.04
(25.11.2) is out but isn't in CLAUDE.md's target list. SchedMD publishes
source only; it has no distro repos.

### Slurm 23.11 or later for EL 8/9 (researched 2026-10-17)

No source ships 26.05 for EL 8 or 9; EL 10 gets it from EPEL. Candidates:

- **OpenHPC 2.10 / 2.11:** EL 8. The 2.11 repo index has Slurm 25.05.9 (2.10
  moved from 23.11.x to 25.05.x).
  https://openhpc.community/?p=355
- **OpenHPC 3.5:** EL 9, Slurm 25.11.x (moved up from 24.11.x).
  https://openhpc.community/news/2026/openhpc-v3-5-release-major-slurm-upgrade-and-library-modernization/
  The `update.3.6` repo index now has 25.11.8.
  Repositories: https://repos.openhpc.community/OpenHPC/ (lines 2, 3, 4).
- **Rocky SIG/HPC:** `dnf install rocky-release-hpc` on Rocky 8/9. Its wiki
  lists `slurm22` and `slurm23` packages. Current versions are unconfirmed
  and the wiki is dated around 2023, so it is not recommended.
- **Building from source:** `rpmbuild -ta` on the SchedMD tarball, using its
  `slurm.spec`. This is the upstream method and works for any version.
- **Third-party repos**, unofficial and not recommended for the role:
  `jose-d/slurm_for_Rocky9` (EL9, 26.05.4) and
  `omnivector-solutions/slurm-repo` (Rocky 9, 25.05.1).

The OpenHPC packages have an `-ohpc` suffix (`slurm-ohpc`,
`slurm-slurmd-ohpc`, ...), so the role's built-in `__slurm_redhat_packages`
names won't match. Supporting OpenHPC needs a public package-name override.
That would be a new installation source, which is out of scope until a task
allows it.

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
- T7: done with no handler changes. They were already in order, and the
  review found no one-line fix.
- T8: applies, to `meta/main.yml` only. There is no CI matrix.
- T9: applies.
- T10: rewritten (2026-10-17) as Molecule in GitHub CI.
- T11: added (2026-10-17) for pre-existing bugs, AuthType restart, and
  slurm.key.

### Future work

- logrotate.j2 hardcodes `/var/log/slurm-llnl` for all Debian-family hosts. Now covered by T11d.
- An install source for Slurm 23.11 or later on EL 8/9 and Debian 12 (out of scope: new installation sources). See the EL 8/9 research above; OpenHPC is the best candidate.
- Ubuntu 26.04 and Debian 13 support, as candidates for the target list.
- auth/slurm: T3 derives CredType and the slurmdbd AuthType from `slurm_config.AuthType`, but the role neither distributes `slurm.key` (owned by SlurmUser, mode 0600) nor skips munge setup under auth/slurm. `slurm.key` is now covered by T11g; skipping munge is still out of scope.
- Removed-parameter guard (T5) covers only keys removed by 23.11. Keys removed later (`AccountingStorageUser` 25.11; `GetEnvTimeout`, `CoreSpecPlugin`, `PowerPlugin`, `PowerParameters` defunct in 26.05) need version gating, but `__slurm_version` is only set after config templating.
