# Windows 2022 DISA STIG

## Configure a Windows 2022 system to be [DISA STIG](https://public.cyber.mil/stigs/downloads/) compliant

### Based on [Windows DISA STIG Version 2, Rel 9 released on July 1st, 2026](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_Server_2022_V2R9_STIG.zip)

---

![Org Stars](https://img.shields.io/github/stars/ansible-lockdown?label=Org%20Stars&style=social)
![Stars](https://img.shields.io/github/stars/ansible-lockdown/Windows-2022-STIG?label=Repo%20Stars&style=social)
![Forks](https://img.shields.io/github/forks/ansible-lockdown/Windows-2022-STIG?style=social)
![Followers](https://img.shields.io/github/followers/ansible-lockdown?style=social)
[![X URL](https://img.shields.io/twitter/url/https/x.com/AnsibleLockdown.svg?style=social&label=Follow%20%40AnsibleLockdown)](https://x.com/AnsibleLockdown)

![Discord Badge](https://img.shields.io/discord/925818806838919229?logo=discord)

![Release Branch](https://img.shields.io/badge/Release%20Branch-Main-brightgreen)
![Release Tag](https://img.shields.io/github/v/tag/ansible-lockdown/Windows-2022-STIG?label=Release%20Tag&color=success)
![Main Release Date](https://img.shields.io/github/release-date/ansible-lockdown/Windows-2022-STIG?label=Release%20Date)

![Devel Commits](https://img.shields.io/github/commit-activity/m/ansible-lockdown/Windows-2022-STIG/devel?color=dark%20green&label=Devel%20Branch%20Commits)

![Issues Open](https://img.shields.io/github/issues-raw/ansible-lockdown/Windows-2022-STIG?label=Open%20Issues)
![Issues Closed](https://img.shields.io/github/issues-closed-raw/ansible-lockdown/Windows-2022-STIG?label=Closed%20Issues&color=success)
![Pull Requests](https://img.shields.io/github/issues-pr/ansible-lockdown/Windows-2022-STIG?label=Pull%20Requests)

![License](https://img.shields.io/github/license/ansible-lockdown/Windows-2022-STIG?label=License)

---

## Looking For Support?

[Lockdown Enterprise](https://www.lockdownenterprise.com#GH_AL_WINDOWS_2022_stig)

[Ansible support](https://www.mindpointgroup.com/cybersecurity-products/ansible-counselor#GH_AL_WINDOWS_2022_stig)

### Community

Join us on our [Discord Server](https://www.lockdownenterprise.com/discord) to ask questions, discuss features, or just chat with other Ansible-Lockdown users.

### Contributing

Bug reports and feature requests are welcome from everyone, please raise an issue.

Pull requests are accepted from approved contributors only. To be onboarded, join the [Discord Server](https://www.lockdownenterprise.com/discord) and request contributor access. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process.

---

## Caution(s)

This role **will make changes to the system** which may have unintended consequences. This is not an auditing tool but a remediation tool to be used after an audit.

Check Mode is not supported! The role will complete in check mode without errors, but it is not supported and should be used with caution.

This role was developed against a clean install of the Windows 2022 operating system. If you are implementing an existing system please review this role for any site-specific changes that are needed.

To use the release version please point to the main branch and relevant release for the STIG benchmark you wish to work with.

---

## Domain Members

Account policy is domain scoped. On a domain joined host the Default Domain Policy owns
`[System Access]`, so the 13 secedit backed controls in this role cannot hold there. `prelim.yml`
detects domain membership, skips those controls and warns once, rather than writing settings the
domain will not keep. Set them in the Default Domain Policy instead.

Everything outside `[System Access]` applies normally on a domain member.

This behavior was verified on a domain joined workstation during Windows Fleet testing, where a
complete hardening run left all `[System Access]` values byte-identical to the pre-run baseline. It
has not been separately measured on Windows Server 2022.

---

## Matching A Security Level For STIG

It is possible to only run controls that are based on a particular security level for STIG.
This is managed using tags:

- CAT1
- CAT2
- CAT3

The control found in the defaults main also needs to reflect true so as this will allow the controls to run when the playbook is launched.

## Coming From A Previous Release

STIG releases always contain changes, so it is highly recommended to review the new references and available variables. This has changed significantly since the initial release of ansible-lockdown.
This is now compatible with python3 if it is found to be the default interpreter. This does come with prerequisites that configure the system accordingly.

Further details can be seen in the [Changelog](./CHANGELOG.md)

## Auditing (new)

Currently, this release does not have an auditing tool.

## Compliance facts

With `create_benchmark_facts` enabled (the default), the role writes a record of what it applied to:

```
C:\ProgramData\ansible\facts.d\compliance_facts.json
```

It captures the benchmark release, the run date and which CAT levels were enabled.

Windows has no default local-facts directory, so unlike the Linux roles this file is **not**
collected automatically. Ask for it explicitly:

```yaml
- name: Read the compliance facts
  ansible.windows.setup:
    fact_path: 'C:\ProgramData\ansible\facts.d'
```

It then appears as `ansible_compliance_facts` - not under `ansible_local`, which is the Linux
convention. Set `ansible_facts_path` to relocate the directory, or `create_benchmark_facts: false`
to skip writing it.

## Documentation

- [Read The Docs](https://ansible-lockdown.readthedocs.io/en/latest/)
- [Getting Started](https://www.lockdownenterprise.com/docs/getting-started-with-lockdown#GH_AL_WINDOWS_2022_stig)
- [Customizing Roles](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise#GH_AL_WINDOWS_2022_stig)
- [Per-Host Configuration](https://www.lockdownenterprise.com/docs/per-host-lockdown-enterprise-configuration#GH_AL_WINDOWS_2022_stig)
- [Getting the Most Out of the Role](https://www.lockdownenterprise.com/docs/get-the-most-out-of-lockdown-enterprise#GH_AL_WINDOWS_2022_stig)

## Requirements

**General:**

- Basic knowledge of Ansible, below are some links to the Ansible documentation to help get started if you are unfamiliar with Ansible

  - [Main Ansible documentation page](https://docs.ansible.com)
  - [Ansible Getting Started](https://docs.ansible.com/ansible/latest/user_guide/intro_getting_started.html)
  - [Tower User Guide](https://docs.ansible.com/ansible-tower/latest/html/userguide/index.html)
  - [Ansible Community Info](https://docs.ansible.com/ansible/latest/community/index.html)
- Functioning Ansible and/or Tower Installed, configured, and running. This includes all of the base Ansible/Tower configurations, needed packages installed, and infrastructure setup.
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consequences in a live production system. Also, familiarize yourself with the variables in the defaults/main/main.yml file.

**Technical Dependencies:**

- Windows 2022 - Other versions are not supported
- Running Ansible/Tower setup. This role requires ansible-core 2.16.1 or newer; the role asserts this at run time.
- Python3 Ansible run environment
- pywinrm

`pywinrm` is required on the controller host that executes Ansible; it is the connection library Ansible uses to reach a Windows target.

## Role Variables

### Breaking changes in this release

**1. Variable prefix.** Role behavior variables and security tunables now share the `win22stig_`
prefix. If you override any of the security tunables in inventory, group_vars or extra vars,
rename them - the old names are no longer read and your setting will be silently ignored.

- `wn22stig_<name>` becomes `win22stig_<name>` (for example `wn22stig_lockoutbadcount`
  becomes `win22stig_lockoutbadcount`)
- `win2022stig_<name>` becomes `win22stig_<name>` for `audit_complex`, `audit_disruptive`,
  `complexity_high` and `disruption_high`
- three names also correct a typo: `sebackuprivilege` becomes `sebackupprivilege`,
  `selockmemorprivilege` becomes `selockmemoryprivilege`, and `machineaccountpsswd_max_age`
  becomes `machineaccountpassword_max_age`

Rule toggles are unchanged. They keep the `wn22_<control id>` form, for example
`wn22_au_000010`.

**2. CAT control switches renamed.** `win2022stig_cat1_patch`, `win2022stig_cat2_patch` and
`win2022stig_cat3_patch` become `win22stig_cat1_controls`, `win22stig_cat2_controls` and
`win22stig_cat3_controls`, matching the names the tasks actually read. The prefix and the suffix
both change, so the general prefix rule above does not cover these.

**3. The GPO authoring path has been removed.** This role is now remediation only. The
`tasks/gpo_creation/` and `tasks/domain_creation/` trees, their pipeline workflows, and the
`win22stig_create_gpos` / `win22stig_create_domain` / `win22stig_ansible_remediation` switches
are gone. Every control the GPO path implemented is implemented by the remediation path.

**4. Removed duplicate variables.** `win22stig_app_maxsize`, `win22stig_sec_maxsize`,
`win22stig_sys_maxsize` and `win22stig_krbtgt_pass_age` were dead duplicates that no task read.
Use `win22stig_application_event_log_max_size`, `win22stig_security_event_log_max_size`,
`win22stig_system_event_log_max_size` and `win22stig_krbtgt_account_pass_age` instead.

This role is designed so that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main/main.yml file or with extra vars within the project, job, workflow, etc. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default. Disruptive finding remediation can be enabled by setting `win22stig_disruption_high` to `true`.

## Tags

Below is an example of the tag section from control within this role. Using this example if you set your run to skip all controls with the tag CCI-000366, this task will be skipped. The opposite can also happen where you run only controls tagged with CCI-000366.

```sh
tags:
      - WN22-00-000010
      - CAT2
      - CCI-000366
      - SRG-OS-000480-GPOS-00227
      - SV-254238r848530_rule
      - V-254238
```

## Community Contribution

Pull requests are accepted from approved contributors only, and issues are welcome from everyone.
See [CONTRIBUTING.md](CONTRIBUTING.md) for the onboarding process, the rules, and the commit signing
requirements (GPG signature and Signed-off-by on every commit).

## Pipeline Testing

uses:

- ansible-core 2.16.1 or newer, from the pinned virtualenv on the runner
- runs the role against a Windows target provisioned on Azure with OpenTofu, which is torn down when
  the run ends
- self-hosted runners
- pull requests into `devel` or a `benchmark*` branch run the devel pipeline; pull requests into
  `main` or `latest` run the main pipeline
- the job runs only for pull requests raised from a branch in this repository, because it carries the
  cloud credentials

## Local Testing

- Ansible
  - ansible-core 2.16.1 or newer, with Python 3
- `pywinrm` on the controller, which is the connection library Ansible uses to reach a Windows target

## Credits and Thanks

Massive thanks to the fantastic community and all its members.

This includes a huge thanks and credit to the original authors and maintainers.
