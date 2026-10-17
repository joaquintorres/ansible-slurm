# Tasks

Work top to bottom. Check a box only when its acceptance criteria are met
and the change is committed. Follow the rules in CLAUDE.md throughout.

---

## [ ] T0 — Audit (no code changes)

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

## [ ] T1 — Installed version fact

After the package install, set a fact `__slurm_version` (for example,
`"26.05"`).

- Get the version from `ansible.builtin.package_facts` if the role already
  gathers it; otherwise run a `command` task with `changed_when: false`.
- Make it work in `--check` mode.

**Acceptance:** One or two tasks added; nothing else changed. Later tasks
compare against this fact with `is version(...)`.

---

## [ ] T2 — Config paths

Make Debian and Ubuntu use `/etc/slurm` on the supported releases.

- Edit only the OS vars files.
- Leave any older-release entries as they are.

**Acceptance:** No task or template changes.

---

## [ ] T3 — Modern `slurm.conf` defaults

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

## [ ] T4 — `cgroup.conf` defaults for cgroup v2

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

## [ ] T6 — `slurmdbd.conf` permissions

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

### Future work
