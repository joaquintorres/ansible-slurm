Slurm
=====

Install and configure a Slurm cluster on Enterprise Linux or Ubuntu servers

Supported Versions
------------------

Slurm 23.11 or newer, with 26.05 as the primary target, installed from the distribution's packages on EL 8/9/10
(from EPEL) and Ubuntu 24.04. Of these, only Ubuntu 24.04 currently ships Slurm 23.11 or newer: EPEL has Slurm 20.11 for
EL 8 and 22.05 for EL 9, and none for EL 10 before 10.3. On EL, provide newer packages through your own repository.

Upgrading from Previous Versions
--------------------------------

These defaults changed (see [CHANGELOG](CHANGELOG.md)):

- The fallback controller line is `SlurmctldHost=localhost` instead of `ControlMachine=localhost`. It is
  written only when you set neither key.
- `CryptoType: crypto/munge` is replaced by `CredType`. It is `cred/munge`, or `cred/slurm` when
  `slurm_config.AuthType` is `auth/slurm`. If you set `CryptoType` yourself, switch to `CredType`.
- `slurmdbd.conf`'s default `AuthType` follows `slurm.conf`'s `AuthType`.

The role now stops before writing any config file if `slurm_config`, `slurm_cgroup_config`, or `slurmdbd_config`
sets a parameter that Slurm removed by 23.11, such as `SelectType: select/cons_res`, `FastSchedule`,
`CgroupAutomount`, or `TaskAffinity`. The error names the replacement key, if there is one. Remove or
replace those keys before upgrading. The full list is `__slurm_removed_params` in [vars/main.yml](vars/main.yml).

Role Variables
--------------

All variables are optional. If nothing is set, the role will install the Slurm client programs, munge, and
create a `slurm.conf` with a single `localhost` node and `debug` partition.
See the [defaults](defaults/main.yml) and [example playbooks](#example-playbooks) for examples.

For the various roles a slurm node can play, you can either set group names, or add values to a list, `slurm_roles`.

- group slurmservers or `slurm_roles: ['controller']`
- group slurmexechosts or `slurm_roles: ['exec']`
- group slurmdbdservers or `slurm_roles: ['dbd']`

General config options for slurm.conf go in `slurm_config`, a hash. Keys are Slurm config option names.

Partitions and nodes go in `slurm_partitions` and `slurm_nodes`, lists of hashes. The only required key in the hash is
`name`, which becomes the `PartitionName` or `NodeName` for that line. All other keys/values are placed on to the line
of that partition or node.

Options for the additional configuration files [acct_gather.conf](https://slurm.schedmd.com/acct_gather.conf.html),
[cgroup.conf](https://slurm.schedmd.com/cgroup.conf.html) and [gres.conf](https://slurm.schedmd.com/gres.conf.html)
may be specified in the `slurm_acct_gather_config`, `slurm_cgroup_config` (both of them hashes) and
`slurm_gres_config` (list of hashes) respectively.

Set `slurm_upgrade` to true to upgrade the installed Slurm packages.

With `AuthType: auth/slurm` in `slurm_config` (Slurm 23.11 or newer), the role installs `slurm.key` in
`slurm_config_dir` on controller, execution and slurmdbd hosts (owned by the Slurm user, mode 0600). Set `slurm_key`
to the path of a key on the control node to use your own key. Otherwise the role generates one once, as
`slurm.key` next to your playbook, and reuses it on later runs. Keep that file safe (for example with
`ansible-vault`). If you run the playbook from another directory, the role generates and installs a new key.

You can use `slurm_user` (a hash) and `slurm_create_user` (a bool) to pre-create a Slurm user so that uids match.

Note that this role requires root access, so enable ``become`` either globally in your playbook / on the commandline or
just for the role like [shown below](#example-playbooks).

Dependencies
------------

None.

Example Playbooks
-----------------

Minimal setup, all services on one node:

```yaml
- name: Slurm all in One
  hosts: all
  vars:
    slurm_roles: ['controller', 'exec', 'dbd']
  roles:
    - role: galaxyproject.slurm
      become: True
```

More extensive example:

```yaml
- name: Slurm execution hosts
  hosts: all
  roles:
    - role: galaxyproject.slurm
      become: True
  vars:
    slurm_cgroup_config:
      CgroupMountpoint: "/sys/fs/cgroup"
      ConstrainCores: yes
      ConstrainRAMSpace: yes
      ConstrainSwapSpace: no
      ConstrainDevices: no
      AllowedRamSpace: 100
      AllowedSwapSpace: 0
      MaxRAMPercent: 100
      MaxSwapPercent: 100
      MinRAMSpace: 30
    slurm_config:
      AccountingStorageType: "accounting_storage/none"
      ClusterName: cluster
      GresTypes: gpu
      JobAcctGatherType: "jobacct_gather/none"
      MpiDefault: none
      ProctrackType: "proctrack/cgroup"
      ReturnToService: 1
      SchedulerType: "sched/backfill"
      SelectType: "select/cons_tres"
      SelectTypeParameters: "CR_Core"
      SlurmctldHost: "slurmctl"
      SlurmctldLogFile: "/var/log/slurm/slurmctld.log"
      SlurmctldPidFile: "/var/run/slurmctld.pid"
      SlurmdLogFile: "/var/log/slurm/slurmd.log"
      SlurmdPidFile: "/var/run/slurmd.pid"
      SlurmdSpoolDir: "/var/spool/slurmd"
      StateSaveLocation: "/var/spool/slurmctld"
      SwitchType: "switch/none"
      TaskPlugin: "task/affinity,task/cgroup"
      TaskPluginParam: Sched
    slurm_create_user: yes
    slurm_gres_config:
      - File: /dev/nvidia[0-3]
        Name: gpu
        NodeName: gpu[01-10]
        Type: tesla
    slurm_munge_key: "../../../munge.key"
    slurm_nodes:
      - name: "gpu[01-10]"
        CoresPerSocket: 18
        Gres: "gpu:tesla:4"
        Sockets: 2
        ThreadsPerCore: 2
    slurm_partitions:
      - name: gpu
        Default: YES
        MaxTime: UNLIMITED
        Nodes: "gpu[01-10]"
    slurm_roles: ['exec']
    slurm_user:
      comment: "Slurm Workload Manager"
      gid: 888
      group: slurm
      home: "/var/lib/slurm"
      name: slurm
      shell: "/usr/sbin/nologin"
      uid: 888
```

License
-------

MIT

Author Information
------------------

- [Nate Coraor](https://github.com/natefoo)
- [Helena Rasche](https://github.com/erasche)

[View contributors on GitHub](https://github.com/galaxyproject/ansible-slurm/graphs/contributors)
