# Modernizing the galaxyproject.slurm role

## Goal

Make this role work with Slurm 23.11 and newer on current distributions
(EL 8/9/10, Ubuntu 22.04/24.04), prioritizing the latest 26.05 version. Do this with the **smallest diff possible**.
This is a compatibility update, not a rewrite.

## Rules for every change

1. **Minimal diff.** Change only the lines a task requires. Do not reformat,
   reorder, re-indent, rename, or convert untouched lines to FQCN, even if
   ansible-lint complains about them.
2. **Preserve the interface.** Keep every existing variable name and its
   behavior. If a default value must change, record it in CHANGELOG.
3. **Data over logic.** Prefer editing defaults or vars files to adding new
   tasks with `when:` conditions. Add a new task only when a task says to.
4. **No new abstractions.** No new files, roles, collections, or
   dependencies unless the task explicitly lists them.
5. **Stop on surprises.** If the repo doesn't match what TASKS.md assumes,
   stop and report. Do not improvise a larger change to work around it.
6. **Verify Slurm facts.** Before encoding any claim about Slurm (a removed
   parameter, a default, a version), confirm it in the official docs at
   slurm.schedmd.com or the NEWS/RELEASE_NOTES file for that release tag.
   Put the version in a short comment where it helps a future reader.
7. **One task, one commit.** Use the commit message format
   `slurm: <task id> <short summary>`.

## Diff budget

Aim for under ~20 changed lines per task, excluding CHANGELOG and README.
If a task needs more, stop and explain why before continuing.

## Verification after each task

- Run `ansible-playbook --syntax-check` on a test playbook that applies the role.
- Run `ansible-lint` on **changed files only**. Fix only issues you introduced.
- If a Molecule scenario exists, run `molecule test`.

## Reporting

After each task, report:
- the files touched
- the number of lines added and removed (`git diff --stat`)
- how you verified the change
- anything you deferred
