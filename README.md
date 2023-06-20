# Windows 2022 DISA STIG

## Configure a Windows 2022 system to be [DISA STIG](https://public.cyber.mil/stigs/downloads/) compliant

### Based on [Windows DISA STIG Version 1, Rel 3 released on May 17, 2023](https://dl.dod.cyber.mil/wp-content/uploads/stigs/zip/U_MS_Windows_Server_2022_V1R3_STIG.zip)

![Release](https://img.shields.io/github/v/release/ansible-lockdown/Windows-2022-STIG?style=plastic)

---

![Org Stars](https://img.shields.io/github/stars/ansible-lockdown?label=Org%20Stars&style=social)
![Stars](https://img.shields.io/github/stars/ansible-lockdown/Windows-2022-STIG?label=Repo%20Stars&style=social)
![Forks](https://img.shields.io/github/forks/ansible-lockdown/Windows-2022-STIG?style=social)
![followers](https://img.shields.io/github/followers/ansible-lockdown?style=social)
[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/AnsibleLockdown.svg?style=social&label=Follow%20%40AnsibleLockdown)](https://twitter.com/AnsibleLockdown)

![Discord Badge](https://img.shields.io/discord/925818806838919229?logo=discord)

![Devel Build Status](https://img.shields.io/github/actions/workflow/status/ansible-lockdown/Windows-2022-STIG/windows_benchmark_testing_to_devel.yml?label=Devel%20Build%20Status)
![Devel Commits](https://img.shields.io/github/commit-activity/m/ansible-lockdown/Windows-2022-STIG/devel?color=dark%20green&label=Devel%20Branch%20commits)

![Release Branch](https://img.shields.io/badge/Release%20Branch-Main-brightgreen)
![Main Build Status](https://img.shields.io/github/actions/workflow/status/ansible-lockdown/Windows-2022-STIG/windows_benchmark_testing_to_main.yml?label=Build%20Status)
![Main Release Date](https://img.shields.io/github/release-date/ansible-lockdown/Windows-2022-STIG?label=Release%20Date)
![Release Tag](https://img.shields.io/github/v/tag/ansible-lockdown/Windows-2022-STIG?label=Release%20Tag&&color=success)

![Issues Open](https://img.shields.io/github/issues-raw/ansible-lockdown/Windows-2022-STIG?label=Open%20Issues)
![Issues Closed](https://img.shields.io/github/issues-closed-raw/ansible-lockdown/Windows-2022-STIG?label=Closed%20Issues&&color=success)
![Pull Requests](https://img.shields.io/github/issues-pr/ansible-lockdown/Windows-2022-STIG?label=Pull%20Requests)

![License](https://img.shields.io/github/license/ansible-lockdown/Windows-2022-STIG?label=License)

---

## Looking for support?

[Lockdown Enterprise](https://www.lockdownenterprise.com#GH_AL_WINDOWS_2022_stig)

[Ansible support](https://www.mindpointgroup.com/cybersecurity-products/ansible-counselor#GH_AL_WINDOWS_2022_stig)

### Community

Join us on our [Discord Server](https://discord.io/ansible-lockdown) to ask questions, discuss features, or just chat with other Ansible-Lockdown users.

---

## Caution(s)

This role **will make changes to the system** which may have unintended consequences. This is not an auditing tool but rather a remediation tool to be used after an audit has been conducted.

Check Mode is not supported! The role will complete in check mode without errors, but it is not supported and should be used with caution.

This role was developed against a clean install of the Windows 2022 operating system. If you are implementing to an existing system please review this role for any site specific changes that are needed.

To use release version please point to main branch and relevant release for the STIG benchmark you wish to work with.

---

## Matching a security Level for STIG

It is possible to to only run controls that are based on a particular for security level for STIG.
This is managed using tags:

- CAT1
- CAT2
- CAT3

The control found in defaults main also need to reflect true so as this will allow the controls to run when the playbook is launched.

## Coming from a previous release

STIG releases always contain changes, it is highly recommended to review the new references and available variables. This has changed significantly since the initial release of ansible-lockdown.
This is now compatible with python3 if it is found to be the default interpreter. This does come with pre-requisites which it configures the system accordingly.

Further details can be seen in the [Changelog](./ChangeLog.md)

## Auditing (new)

Currently this release does not have a auditing tool.

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
- Please read through the tasks in this role to gain an understanding of what each control is doing. Some of the tasks are disruptive and can have unintended consequences in a live production system. Also familiarize yourself with the variables in the defaults/main.yml file.

**Technical Dependencies:**

- Windows 2022 - Other versions are not supported
- Running Ansible/Tower setup (this role is tested against Ansible version 2.9.1 and newer)
- Python3 Ansible run environment
- passlib (or python2-passlib, if using python2)
- python-lxml
- python-xmltodict
- python-jmespath
- pywinrm

Package 'python-xmltodict' is required if you enable the OpenSCAP tool installation and run a report. Packages python(2)-passlib and python-jmespath are required for tasks with custom filters or modules. These are all required on the controller host that executes Ansible.

## Role Variables

This role is designed that the end user should not have to edit the tasks themselves. All customizing should be done via the defaults/main.yml file or with extra vars within the project, job, workflow, etc. Non-disruptive CAT I, CAT II, and CAT III findings will be corrected by default. Disruptive finding remediation can be enabled by setting `win22stig_disruption_high` to `yes`.

## Tags

Below is an example of the tag section from a control within this role. Using this example if you set your run to skip all controls with the tag CCI-000366, this task will be skipped. The opposite can also happen where you run only controls tagged with CCI-000366.

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

We encourage you (the community) to contribute to this role. Please read the rules below.

- Your work is done in your own individual branch. Make sure to Signed-off and GPG sign all commits you intend to merge.
- All community Pull Requests are pulled into the devel branch.
- Pull Requests into devel will confirm your commits have a GPG signature, Signed-off, and a functional test before being approved.
- Once your changes are merged and a more detailed review is complete, an authorized member will merge your changes into the main branch for a new release.

## Pipeline Testing

uses:

- ansible-core 2.12
- ansible collections - pulls in the latest version based on requirements file
- runs the audit using the devel branch
- This is an automated test that occurs on pull requests into devel
