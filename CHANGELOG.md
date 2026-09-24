# Changes to WIN22STIG

## Based on STIG v2.9.0 - ALD Windows Alignment Updates

### Breaking changes

Measured against the previous public release: 49 variable names no longer resolve, 41 renamed and 8
removed. An override left under an old name is not read and is silently ignored. Rule toggles are
unchanged and keep the `wn22_<control id>` form.

- BREAKING: **41 variables are renamed.** 31 `wn22stig_<name>` become `win22stig_<name>`, and
  `win2022stig_<name>` becomes `win22stig_<name>` for `audit_complex`, `audit_disruptive`,
  `complexity_high` and `disruption_high`. The three category switches change suffix as well as
  prefix: `win2022stig_cat<N>_patch` becomes `win22stig_cat<N>_controls`. Three are typo
  corrections: `sebackuprivilege` becomes `sebackupprivilege`, `selockmemorprivilege` becomes
  `selockmemoryprivilege`, and `machineaccountpsswd_max_age` becomes
  `machineaccountpassword_max_age`.
- BREAKING: **8 variables are removed.** Four were dead duplicates that the tasks never read:
  `wn22stig_app_maxsize`, `wn22stig_sec_maxsize` and `wn22stig_sys_maxsize`, superseded by the
  `*_event_log_max_size` names, and `wn22stig_krbtgt_pass_age`, superseded by
  `wn22stig_krbtgt_account_pass_age`. The rest are `win22stig_cloud_based_system` with the cloud
  detection it gated, `win2022stig_system_is_container` which was inert,
  `win2022stig_min_ansible_version` which is now `min_ansible_version` in `vars/main.yml`, and the
  rule toggle `wn22_00_000290`, whose control this benchmark revision retired.

### Feature removals gated on the feature existing

- FIXED: **eight feature removal controls aborted the play on a host that does not ship the feature.**
  `win_feature` raises "The role, role service, or feature name is not valid" for a name the OS does
  not know, which ends the run on a control whose requirement is already trivially met. The affected
  controls are `WN22-00-000320` (Fax), `-000330` (Web-Ftp-Server), `-000340` (PNRP), `-000350`
  (Simple-TCPIP), `-000360` (Telnet-Client), `-000370` (TFTP-Client), `-000380` (FS-SMB1) and
  `-000410` (PowerShell-V2), none of which carried any guard. `prelim.yml` now enumerates the valid
  feature names once with `Get-WindowsFeature` and each control is gated on membership of that list.
  Reported by the community as
  [ansible-lockdown/Windows-2022-STIG#20](https://github.com/ansible-lockdown/Windows-2022-STIG/issues/20).

### Connection severing controls guarded - carried across from the Windows Fleet

Benchmark: **Windows Server 2022 STIG v2.9.0**.

A Windows Fleet role was run against a live domain joined host in September 2026 and lost the
host twice to controls that remove a local account's network access. Both have direct equivalents
here, and neither was guarded. Neither is reachable by `ansible-lint`, `yamllint` or
`--syntax-check`; both only bite at runtime, against a domain joined member server.

**These guards are ported and statically verified. They have not been run against a Windows Server 2022
host.** Each was host proven on the fleet test host.

- FIXED: **WN22-MS-000020 would sever the control connection.** It writes
  `LocalAccountTokenFilterPolicy=0`, filtering the privileged token of local accounts on network
  logon, so a run connected over WinRM or psrp as a local administrator loses the host as it
  applies. It presents as `the specified credentials were rejected by the server` rather than a
  dropped connection, because the port keeps answering and the service keeps running.

- FIXED: **WN22-MS-000080 would sever both transports at once.** Its member server branch denies
  network logon to `Guests`, `Enterprise Admins`, `Domain Admins`, `Local account` and
  `Local account and member of Administrators group`. That removes the network logon right from
  every local account on the host, including the one the run is connected as. Unlike WN22-MS-000020
  it takes SSH with it, because Windows OpenSSH password authentication uses the same logon path.
  On the fleet test host, after the equivalent control applied, every remote transport failed - WinRM
  password as both a local and a domain admin principal, SSH password, SSH public key, and WinRM
  certificate authentication - while 5985, 5986 and 22 all still accepted TCP. The console was the
  only way back.

  Note the deny list also names the two privileged domain groups, so **a Domain Admin does not
  survive it either**. The principal that does is a domain account that is neither a Domain Admin
  nor an Enterprise Admin, and that is a member of the local Administrators group on the target.

  Both controls now carry `not win_skip_for_test` (this role's default remains `false`) and are
  listed with the other connection severing controls in `defaults/main/main.yml`. Both are **also**
  skipped automatically when the run connects as a local account, whatever `win_skip_for_test` is
  set to, from two new facts in `prelim.yml`. The operator is warned once per control and each
  warning is counted through `warning_facts.yml`.

  The facts are deliberately separate. `prelim_control_account_is_local` is about the principal and
  applies on any transport, and guards WN22-MS-000080. `prelim_local_account_network_logon` adds the
  transport test and guards WN22-MS-000020. Collapsing them would either leave WN22-MS-000080 able to
  sever an SSH run, or stop WN22-MS-000020 applying over SSH where it is safe.

  The guard sits on the **member server branch only**. The non-domain branch of WN22-MS-000080, which
  sets the right to `Guests` alone and cannot sever anything, is untouched and still applies.

- NOTE: the local account detection builds its backslash from a YAML single quoted variable rather
  than a Jinja literal. In a Jinja expression neither a single nor a doubled backslash literal
  matches one backslash in this position - the first is a syntax error, the second tests for two -
  so a `DOMAIN\user` principal went undetected. The YAML form takes the character literally and
  never reaches Jinja's string parser.

- CHANGED: the compliance facts file is now JSON. `templates/compliance_facts.ps1.j2` is replaced by
  `templates/compliance_facts.json.j2`, and the role writes
  `C:\ProgramData\ansible\facts.d\compliance_facts.json` instead of `...\compliance_facts.ps1`.
  This aligns the ALD Windows roles on a single facts format. The
  file is now parsed rather than executed to produce the fact.

  **The fact name and its keys are unchanged.** `ansible.windows.setup` builds the key from the
  file's `BaseName`, so it is still `ansible_compliance_facts`, and the five existing keys plus the
  conditional `Cat_N_tag_run` entries keep their names and meanings. Anything already reading this
  fact continues to work.

  Every value goes through `| to_json`, so the CAT toggles are now real JSON booleans rather than
  PowerShell `$true` / `$false`. The conditional tag entries use a leading comma, because a trailing
  comma after `cat_3_hardening_enabled` would emit invalid JSON on the default untagged run.

- ADDED: a task removing a superseded `compliance_facts.ps1` before the new file is written. Both
  extensions are collected and both produce the same fact key, so on a host hardened by an earlier
  release the stale file would otherwise be a second source for `ansible_compliance_facts` with no
  guaranteed precedence.

- ADDED: a `managed_by` key. The PowerShell file opened with a "managed by ansible" comment banner
  and JSON cannot hold comments, so that provenance is recorded as a field instead of being lost. Its
  value is derived from `file_managed_by_ansible`, so the wording stays in one place and the variable
  remains in use.

- `win_template` now sets `newline_sequence: "\r\n"`, so the file is written with CRLF line endings.
  **Not executed.** No Windows inventory was available, so the file is never actually collected by
  `ansible.windows.setup` here. The reasoning about fact naming comes from reading the module source.
  The rendered output was validated offline against all eight CAT tag combinations.

- `README.md`: the `Community Contribution` section still described the old open-contribution model -
  "We encourage you (the community) to contribute to this role" and "All community Pull Requests are
  pulled into the devel branch". That contradicted both `CONTRIBUTING.md`, which states pull requests
  come from approved contributors, and this README's own `Contributing` section a few screens above
  it. Replaced with the wording the Windows Fleet already carried: pull requests from
  approved contributors, issues welcome from everyone, and a pointer to `CONTRIBUTING.md` for
  onboarding and the commit signing requirements. The Windows Fleet now carries an identical
  section.

- `README.md`: removed four controller-side dependencies the role does not use. It declared
  `passlib`, `python-lxml`, `python-xmltodict` and `python-jmespath`, and a paragraph describing an
  OpenSCAP tool installation. This role calls only `ansible.windows`, `community.windows` and
  `ansible.builtin` modules: there is no `password_hash` filter, no `xml` module, no `json_query`
  filter and no OpenSCAP task anywhere in it. `pywinrm` is retained and now carries the note, since
  it is the controller-side connection library. The only XML in the role is PowerShell
  `Get-AppLockerPolicy -Effective -XML` running on the target, which implies nothing about
  controller packages.

- FIXED: `.yamllint` now ignores `.ansible/`, matching the Windows Fleet. An `ansible-lint` run
  installs `collections/requirements.yml` into `.ansible/collections/`, and `yamllint` then walked
  that tree and linted several hundred vendored `ansible.windows` and `community.windows` files
  against this role's style rules. Only dependency code is excluded: role YAML is still linted.

### Account policy scope on domain joined hosts

- ADDED: **the 13 secedit backed `[System Access]` controls are skipped on a domain joined host.**
  On such a host the Default Domain Policy owns that section and overwrites it at every policy
  refresh, so a local write reverts within roughly two hours while the run still reports success.
  An audit taken straight afterwards therefore reports a compliance that does not last. A new
  `discovered_account_policy_is_domain_scoped` fact gates the account lockout trio and the password
  policy controls, and the operator is warned once with a pointer to set them in the Default Domain
  Policy instead.

  This is the failure class behind the `The key 'LockoutDuration' in section 'System Access' is not
  a valid key` reports from domain joined hosts. Not reachable by `ansible-lint`, `yamllint` or
  `--syntax-check`: the controls are individually valid and only fail against a joined host.

- ADDED: `prelim.yml` gathers the `windows_domain` fact subset explicitly when
  `ansible_facts['windows_domain_member']` is not already defined, rather than assuming a prior
  gather supplied it. `discovered_domain_joined` is seeded `false` in `vars/main.yml`, so a host
  whose membership cannot be determined still applies the controls rather than silently skipping
  them.

- REMOVED: `tasks/cat2_cloud_lockout_order.yml`, the conditional import that loaded it, and the
  `win22stig_cloud_based_system` and `win22stig_cloud_vendors` variables. The three account lockout
  controls were implemented twice, here and in that file, selected by a cloud detection fact.
  Both copies had converged on the same keys in the same order, so the duplicate carried
  nothing the standard path did not.
  Cloud detection was also unreliable by construction: Azure and on-prem Hyper-V both report
  `Microsoft Corporation` with model `Virtual Machine`, so the two could not be told apart by the
  facts the detection used. Any inventory setting either variable can drop it; neither is read any
  more.


- `README.md`: added a `Domain Members` section documenting that account policy is domain scoped,
  which controls are skipped on a domain joined host, and the instruction to set them in the
  Default Domain Policy instead.

### Repository hygiene

- FIXED: **Tofu Destroy did not run when `ENABLE_DEBUG` was unset.** The teardown step was gated on
  `env.ENABLE_DEBUG == 'false'`, which is false for an unset or empty variable, so the Azure test
  instance was left running. Now gated on `!= 'true'`, which tears down by default and keeps the
  instance only when debugging is explicitly requested.
- FIXED: **the IAC_BRANCH test was a shell syntax error when the variable was unset.**
  `if [ ${{ vars.IAC_BRANCH }} != '' ]` expands to `if [ != '' ]` with no value. Replaced with
  `if [ -n "${{ vars.IAC_BRANCH }}" ]`.
- FIXED: **the debug step echoed an undefined variable.** `$benchmark_type` is never set; the
  environment carries `TF_VAR_benchmark_type`. Corrected, so `DEBUG - Show IaC files` reports the
  benchmark type instead of an empty string.
- ADDED: explicit least-privilege `permissions:` blocks on both pipeline jobs, rather than inheriting
  the default token scope, and `workflow_dispatch` so either pipeline can be run manually.
- ADDED: `issue_message` to the pinned `actions/first-interaction@v3.1.0` step. The v3.1.0 runtime
  calls `getInput` for it with `required: true` even though its own `action.yml` does not mark it
  required, so the welcome job failed without it.
- `README.md`: rewrote the `Pipeline Testing` section. The previous text described an audit-on-devel
  pipeline this role does not run, and claimed collections are resolved from the requirements file,
  which no pipeline step does. It now records the ansible-core floor, the Azure target and its
  teardown, the branches each pipeline gates on, and the fork restriction on the job holding the
  cloud credentials. The two pipeline-status badges are dropped, matching the rest of the Windows
  Fleet. `Local Testing` is unchanged and still accurate.
- REMOVED: **`update_galaxy.yml`.** It ran once, on 2025-09-03, and failed. The repository holds no
  `GALAXY_API_KEY`, and no Windows Server 2022 STIG role is published on Ansible Galaxy, so it was
  not the mechanism keeping anything current.

- FIXED: **seven NIST tags were misspelled, so selecting by them matched nothing.** A stray `R` in
  `NIST800-53R_4_AC-3` (twice), a trailing `s` on `NIST800-53_4_AC-3s` and
  `NIST800-53A_4_IA-5_1_ds`, a truncated `NIST800-53_4_CM-6_`, a hyphen for an underscore in
  `NIST800-53_4-AC-7_b`, a missing `-53` in `NIST800_AC-8_c`, and an uppercase suffix in
  `NIST800-53_SI-11_B`. Each was a single occurrence against a well populated correct form, and
  each is now that form.

- FIXED: **17 tags used `NIST800-53-` where the rest use `NIST800-53_`.** The two forms are distinct
  tag strings, so `--tags` selection silently missed the minority spelling. Normalized to the
  underscore separator. Ten duplicate tag entries within a single `tags:` list were removed at the
  same time, nine of them pre-existing and one produced by the corrections above resolving to a
  tag the task already carried. Control identifier tags are untouched: `SV-`, `V-`, `CCI-` and
  `WN22-` counts are identical before and after, and coverage stays at 279 of 279.

  This role and one Windows sibling share a tag vocabulary and carried the same defects. The
  remaining three siblings use a different one (`NIST800-53R4_` and `NIST800-53A_`, the latter with
  dot separators, and one of them uniformly `NIST800-53R4_` with no dots) and carry none of these.

### Legal banner title corrected

- FIXED: **`WN22-SO-000140` set the banner title to `DOD Notice and Consent Banner`, which V2R9 does
  not list.** The benchmark gives `DoD Notice and Consent Banner` and `US Department of Defense
  Warning Statement` as the recognized titles. Anything else is an organization-defined equivalent,
  which the control text says requires a manual review rather than passing automatically. The
  uppercase form is what later Windows benchmarks use, but not this one. The default is now the V2R9
  spelling, which also matches the comment directly above it.

### Remediation defects fixed - Windows 10 parity sweep

Found while auditing the Windows Fleet; this was present here too.

- FIXED: the `WN22-00-000090` TPM check carried three defects. Its warning fired on the *compliant*
  TPM state, because the conditions tested for the healthy values being present rather than absent.
  Every assertion read `stderr_lines`, where `wmic` writes nothing but its "No Instance(s)"
  diagnostic, so they could essentially never match and the inversion was masked. And the
  SpecVersion clause read `'SpecVersion=2' or 'SpecVersion=1.2' in ...`, where the bare non-empty
  string on the left of the `or` is always truthy, making that disjunct unconditionally true. The
  probe now uses `Get-CimInstance` rather than `wmic`, which is deprecated and is a
  Feature-on-Demand that can be absent, leaving the register with no `stdout`/`stderr` keys at all.
  Adopted from the Windows Fleet, where it had already been corrected.


- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. The Windows Fleet now ships a byte-identical file
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, normalized the social
  badge to the `X URL` form on `x.com`, and pointed the Discord link at
  `https://www.lockdownenterprise.com/discord`
- `README.md`: aligned the shared heading text and the benchmark banner format with the rest of the
  Windows fleet. Role-specific sections are unchanged
- `defaults/main.yml` moved to `defaults/main/main.yml`, matching the directory layout the Linux
  roles have adopted. Ansible loads role defaults from the directory, so no task or template change
  was needed and no variable name or value changed. Verified by loading the defaults tree through
  Ansible's role-defaults loader
- CHANGED: `benchmark_version` is now expressed numerically, `v2r9` becomes `v2.9.0`. This is the
  value `templates/compliance_facts.json.j2` renders into the local fact `Benchmark_release`, so that
  fact changes from `STIG-v2r9` to `STIG-v2.9.0` on the next run
- the benchmark content is unchanged and remains V2R9

## 2026 August - Contributing guide and README refresh

- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`

## Based on STIG v2.9.0 - Release 2.9.0 - August 2026


Updated to DISA STIG Windows Server 2022 Version 2, Release 9 (benchmark date 01 July 2026),
sourced from `U_MS_Windows_Server_2022_STIG_V2R9_Manual-xccdf.xml`. Rule coverage is 279 of 279.

- BREAKING: role behavior variables and security tunables standardized on the `win22stig_`
  prefix. The `wn22stig_` names are no longer read. Rule toggles keep the `wn22_<control id>` form.
- BREAKING: `win22stig_cat1_patch`, `win22stig_cat2_patch` and `win22stig_cat3_patch` renamed to
  `win22stig_cat1_controls`, `win22stig_cat2_controls` and `win22stig_cat3_controls` to match the
  names the tasks read.
- BREAKING: removed the GPO authoring path. `tasks/gpo_creation/`, `tasks/domain_creation/`, the
  two GPO pipeline workflows and the `win22stig_create_gpos` / `win22stig_create_domain` /
  `win22stig_ansible_remediation` switches are gone; the role is now remediation only. The only
  control unique to the GPO path was WN22-DC-000391, which no longer exists in the benchmark.
- BREAKING: removed the dead duplicate variables `win22stig_app_maxsize`, `win22stig_sec_maxsize`,
  `win22stig_sys_maxsize` and `win22stig_krbtgt_pass_age`, and the inert
  `win22stig_system_is_container`.
- BREAKING: `win22stig_min_ansible_version` is renamed to `min_ansible_version` and moved from
  `defaults/main.yml` to `vars/main.yml`, matching the rest of the Ansible Lockdown fleet. Rename
  it if you override it; the old name is no longer read.
- ADDED: `skip_os_check`, defaulting to false. Setting it true bypasses the OS version and family
  assert for hosts the check misidentifies.
- ADDED: `create_benchmark_facts` and `ansible_facts_path`. The role now writes
  `compliance_facts.ps1` under `C:\ProgramData\ansible\facts.d`, recording the benchmark release,
  the run date and which CAT levels were enabled. Unlike Linux, Windows has no default local-facts
  directory, so the file is collected only when `ansible.windows.setup` is given a matching
  `fact_path`; it then surfaces as `ansible_compliance_facts`.
- ADDED: `company_title` and `file_managed_by_ansible` in `vars/main.yml`, the provenance strings
  the rest of the fleet uses to head managed files.
- CHANGED: `change_requires_reboot` moved from `defaults/main.yml` to `vars/main.yml`. Controls
  raise it at run time with `set_fact`; it was never a user tunable, and pinning it from inventory
  could mask an outstanding reboot.
- ADDED: WN22-AU-000583, WN22-AU-000585, WN22-AU-000586, WN22-AU-000587 and WN22-AU-000588,
  covering handle manipulation, registry and sensitive privilege use auditing.
- REMOVED: WN22-00-000290, retired in this benchmark revision.
- FIXED: the role could not run. `win22stig_cat1_controls`, `win22stig_cat2_controls`,
  `win22stig_cat3_controls`, `change_requires_reboot` and `skip_reboot` were read by the tasks but
  defined nowhere, so the play aborted on an undefined variable before applying any control.
- FIXED: the WN22-00-000420 and WN22-00-000430 FTP audits aborted the play on any host without
  FTP installed. `regex_search` returns None on a miss and the result was then tested with `in`.
  Addresses ansible-lockdown/Windows-2022-STIG#20.
- FIXED: WN22-AC-000030 had its `when:` nested inside the `win_security_policy` module arguments,
  so the guard never applied and the module received an unsupported parameter. The warning tasks
  for the control were also ungated and untagged.
- FIXED: WN22-AC-000040 carried no `when:` or `tags:`, so it ran regardless of its toggle and was
  unreachable by tag. One of its task names used the malformed ID `WN22-22-000040`.
- FIXED: WN22-AU-000220 and WN22-AU-000230 shared a single task that applied both success and
  failure auditing, so either toggle enabled both. They are now separate controls, and
  WN22-AU-000230 has its own V-254316 identifier, which had been dropped.
- FIXED: reboots are now requested through the `Change_requires_reboot` handler and performed by
  `tasks/post.yml`, so `skip_reboot` is honoured. The previous handler rebooted immediately and
  bypassed it.
- BREAKING: two tunables were misspelled against the Windows rights they set and are renamed.
  `win22stig_sebackuprivilege` becomes `win22stig_sebackupprivilege` (`SeBackupPrivilege`) and
  `win22stig_selockmemorprivilege` becomes `win22stig_selockmemoryprivilege`
  (`SeLockMemoryPrivilege`). Rename them if you override either.
- FIXED: two warning messages named the wrong unit. `win22stig_lockoutbadcount` is a count of failed
  logon attempts and `win22stig_resetlockoutcount` is in minutes; both were described as days.
- CHANGED: the six controls that manage user rights now use `ansible.windows.win_user_right`
  instead of `community.windows.win_security_policy` with `section: Privilege Rights`. Ansible
  warns that the latter is error-prone for rights and privileges, and the split across two modules
  is what allowed WN22-MS-000130 to silently undo WN22-DC-000420 without any same-module conflict
  check noticing. Affected: WN22-UR-000010, WN22-UR-000060, WN22-UR-000080, WN22-UR-000160,
  WN22-DC-000390 and WN22-MS-000130.
- BREAKING: `win22stig_selockmemoryprivilege` is now a **list**, defaulting to `[]`, because
  `win_user_right` takes a list of accounts. Override it with a list, not a string.
- FIXED: every domain controller control was gated on
  `ansible_facts.windows_domain_role == "Primary domain controller"`. `ansible.windows` maps
  DomainRole 4 to "Backup domain controller" and 5 to "Primary domain controller", and only the
  PDC-emulator FSMO holder reports 5. On every other domain controller all 59 DC controls skipped
  and the member-server controls ran instead - including WN22-MS-000070, which applies
  `SeNetworkLogonRight` with `action: set` and no `Enterprise Domain Controllers`, breaking AD
  replication. All 59 gates now test `'controller' in ansible_facts.windows_domain_role`.
- FIXED: WN22-MS-000130 had no domain gate, so on a domain controller it cleared
  `SeEnableDelegationPrivilege` after WN22-DC-000420 had set it, leaving DC-000420 non-compliant.
- FIXED: three warn-count conditions in WN22-00-000190, WN22-00-000210 and WN22-00-000310 were
  written as three ANDed list elements where the `or` bound inside only the second, so the third
  dereferenced a skipped register. They raised on a domain controller and never fired on a member
  server.
- FIXED: `win22stig_legalnoticecaption` defaulted to the entire banner body. WN22-SO-000140 checks
  the banner title, and the XCCDF states automated tools search only for the listed titles, so the
  control could never pass. It now defaults to "DOD Notice and Consent Banner".
- FIXED: WN22-00-000090 was gated on `ansible_facts.windows_domain_member` and named
  "domain-joined systems". The V2R9 title is "Windows Server 2022 systems" with no applicability
  carve-out, so the control was silently skipped on every stand-alone server.
- FIXED: 14 `results[0]` reads had no guard and raise when the producing loop returns no results.
- FIXED: a task name carried a bare `V` where the control ID should be; it is WN22-AU-000160.

- FIXED: WN22-AU-000090 enabled failure auditing for Other Account Management Events when the
  benchmark requires success auditing, so the control was never met.
  Addresses ansible-lockdown/Windows-2022-STIG#14.
- FIXED: WN22-AU-000330 tested for `Success` before enabling IPsec Driver failure auditing, so
  the change was skipped whenever success auditing happened to be on already.
  Addresses ansible-lockdown/Windows-2022-STIG#16.
- CHANGED: runtime-discovered values now use the fleet `discovered_` prefix instead of
  `wn22_`, matching the Linux roles and the Windows Fleet. 93 registers and
  set_fact values were renamed, for example `wn22_00_000020_audit_dc` becomes
  `discovered_00_000020_audit_dc`. Facts gathered once in `prelim.yml` keep the `prelim_`
  prefix, which is the same split 2019 uses. Rule toggles are untouched and keep the
  `wn22_<control id>` form.
  This removes a namespace collision: `wn22_00_000020` (a user toggle) and
  `wn22_00_000020_audit_dc` (a runtime register) previously differed only by suffix, so any
  sweep over toggles had to separate them by regex. These names are internal to the role,
  so overriding a toggle is unaffected.
- FIXED: every operational switch used in a `when:` **or in a template** is now filtered through `| bool`. Passed as an
  extra var (`-e skip_reboot=false`) a switch arrives as the **string** `"false"`, which is truthy,
  so a negated conditional evaluated to the opposite of what was asked. `-e skip_os_check=false`,
  meaning "do run the OS check", silently skipped it; `-e skip_reboot=false`, meaning "do reboot",
  silently suppressed the reboot. Non-negated switches failed outright on ansible-core 2.19+ with
  "Conditionals must have a boolean result". Affects `skip_os_check`, `skip_reboot`,
  `create_benchmark_facts`, `win_skip_for_test`, `change_requires_reboot`, the three
  `win22stig_cat*_controls` switches, `win22stig_cloud_based_system`, `win22stig_disruption_high`,
  `win22stig_complexity_high`, `win22stig_lengthy_search` and `win22stig_install_openssh`.
  The same defect existed in the templates: `compliance_facts.ps1.j2` would have recorded
  `cat_N_hardening_enabled = $true` for a CAT level disabled via `-e`, so the compliance record
  would have misreported what the role applied.
- FIXED: WN22-UC-000010 wrote `SaveZoneInformation` to `HKLM`, but the benchmark reads
  `HKEY_CURRENT_USER` and the fix text points at a User Configuration policy, so the remediation
  could not affect the state being checked. It now clears the value in every loaded user hive and
  in the Default profile, which covers all logged-on users and all future profiles, and warns for
  any profile whose hive is not loaded rather than skipping it silently.
- BREAKING: `win22stig_machineaccountpsswd_max_age` is renamed to
  `win22stig_machineaccountpassword_max_age`. Rename it if you override it.
- FIXED: WN22-MS-000140 wrote `RequirePlatformSecurityFeatures`, which belongs to WN22-CC-000110.
  MS-000140 set it to 1 in CAT I and CC-000110 then set it from `win22stig_dma_protection`
  (default 3) in CAT II, so on every run of a domain-joined member server both tasks reported
  changed and the tunable was silently overridden. The control's check text requires only
  `LsaCfgFlags`, so the colliding value has been removed from its loop.
- FIXED: two controls carried no domain-role gate despite the benchmark scoping them.
  WN22-DC-000160 applies to domain controllers only but ran everywhere, and WN22-CC-000140 is
  marked not applicable to domain controllers but applied to them.
- FIXED: WN22-SO-000370 wrote `ProtectionMode` as a REG_SZ; the benchmark requires REG_DWORD.
  Addresses ansible-lockdown/Windows-2022-STIG#12.
- FIXED: WN22-MS-000050 wrote `CachedLogonsCount` as a REG_DWORD; the benchmark requires REG_SZ.
  The value is now tunable through `win22stig_cached_logons_count`.
- FIXED: WN22-CC-000260 hardcoded its registry data and ignored `win22stig_dodownloadmode`.
- FIXED: rule toggles `wn22_ac_000010` and `wn22_ac_000030` were set to the name of another
  toggle rather than a boolean, which failed the play's boolean evaluation.
  Addresses ansible-lockdown/Windows-2022-STIG#19.
- FIXED: cloud detection in `tasks/prelim.yml` classified almost every host as cloud based. The
  condition was `not virtualization_type == 'VMware' or (system_vendor == 'Microsoft Corporation'
  and virtualization_type in ['Hyper-V', 'hvm', 'kvm'])`. Any value in that list already satisfies
  `!= 'VMware'`, so the right-hand side is a strict subset of the left and the whole expression
  reduces to `virtualization_type != 'VMware'`. Bare metal (`NA`), VirtualBox, Xen, on-prem
  Hyper-V and local KVM were all marked cloud based; only a literal `VMware` was not.
   - the fact selects which order the three AC lockout controls are applied in - WN22-AC-000010,
     WN22-AC-000020 and WN22-AC-000030 - so a wrong classification means the wrong ordering, and
     those controls fail if applied out of order. The two paths are mutually exclusive on the same
     boolean, so there was no double-apply or coverage gap.
   - replaced with a two-item AND: `system_vendor` must be in the new `win22stig_cloud_vendors`
     list and `virtualization_type` must be one of `Hyper-V`, `kvm`, `xen`. `system_vendor` is the
     discriminator, not the type: `ansible.windows` derives `virtualization_type` from the
     Manufacturer, so Amazon EC2, Google and DigitalOcean all report `kvm` - and so does a local
     QEMU box. Only the vendor separates them.
   - dropped `hvm`. `ansible.windows` never emits it on Windows; it is a Linux-only value, and the
     comment above the task cited a Linux fact-collection source as its reference. Added `xen`,
     which older AWS instances report through model `HVM domU`, together with the matching `Xen`
     vendor so that entry can actually fire.
   - `win22stig_cloud_vendors` is a variable so an unlisted cloud can be added without editing the
     task. Azure and on-prem Hyper-V both report `Microsoft Corporation` with model
     `Virtual Machine`, so they cannot be told apart by these facts; set
     `win22stig_cloud_based_system` explicitly where that matters.
- FIXED: 43 stale `SV-` rule identifiers, including `SV-2254352r958804_rule`, which matched no
  rule in any revision. Corrected 5 `V-` identifiers, 2 of which referenced Windows Server 2019.
  Added 20 missing CCI references and 1 missing SRG reference.
- FIXED: WN22-00-000250 was implemented as a CAT II control but is CAT I in the benchmark.
- FIXED: WN22-DC-000405 wrote to `HKLM:\SYSTEM\SYSTEM\CurrentControlSet\Services\Kdc`. The
  doubled `SYSTEM` created a stray key and reported success while
  `StrongCertificateBindingEnforcement` was never set on the real one.
- FIXED: WN22-CC-000250 set `AllowTelemetry` to 0, which V2R9 lists as a finding. It now uses
  `win22stig_allow_telemetry`, defaulting to 1.
- FIXED: WN22-CC-000280 required only 196608 KB. V2R9 raised the security event log to 5120000 KB;
  the threshold, the warning conditions and the default now all reflect that.
- FIXED: WN22-00-000250 is CAT I in V2R9. It was moved to `tasks/Cat1/` but kept a `MEDIUM` name
  prefix and a `CAT2` tag, so `--tags CAT2` pulled a CAT I control into a CAT II run.
- FIXED: WN22-00-000300 registered its audit output under `wn22_00_000330_*`, the identifier of a
  different control. The member and standalone branch then gated on the domain controller
  register while rendering the member one, so the warning never appeared on the hosts it targets
  and could raise an undefined-attribute error on a domain controller. Its warn count tested the
  same operand twice.
- FIXED: WN22-00-000020 ran `Get-LocalUser` ungated on domain controllers, where the cmdlet is
  unavailable, and neither audit shell tolerated failure. Its warn count required domain
  controller and member output at once and ended in a tautology, and a
  `win22stig_pass_age_administrator <= 60` guard suppressed the warning precisely when an
  operator had configured a non-compliant age.
- FIXED: WN22-SO-000030 and WN22-AU-000250 each carried the preceding control's SV identifier, so
  `SV-254447r991589_rule` and `SV-254318r991583_rule` appeared nowhere in the role.
  WN22-SO-000250 carried `254469r958524_rule`, missing its `SV-` prefix and a revision behind.
- FIXED: WN22-SO-000050 was tagged `CCI-000169s`, so `--tags CCI-000169` skipped it. Removed four
  stale V2R4 CCI references that V2R9 no longer lists.
- FIXED: 41 registered audit shells had no `check_mode: false`, so under `--check` they were
  skipped and the first downstream `.stdout` read aborted the play.
- FIXED: the cloud lockout ordering file omitted the NIST tags its non-cloud counterpart carries,
  so a NIST-tag-driven run silently skipped those three controls on cloud hosts.
- CHANGED: tasks restructured into `tasks/Cat1`, `tasks/Cat2` and `tasks/Cat3` with one file per
  control ID band.
- CHANGED: the `complexity` lint rule is skipped. The one-file-per-band layout puts 149 and 136
  tasks in the two largest files, over the rule's hard-coded limit of 100, which is not
  configurable.
- CHANGED: removed a prelim task that claimed to detect a TPM but read the OS product type, and
  whose register nothing consumed.
- CHANGED: `.ansible-lint` carried a `parseable` key that current ansible-lint rejects, so the
  role could not be linted at all. Removing it cleared a backlog of 549 findings, and yamllint
  now reports none.
- CHANGED: `.gitignore` no longer ignores `.github/` wholesale, which had been silently
  swallowing new workflow files.
- CHANGED: collection requirements are pinned by version rather than tracking git HEAD, and the
  unused `community.general` requirement was dropped.
- CHANGED: migrated 81 bare `ansible_*` fact references to the `ansible_facts` form.
- CHANGED: rebranded to MindPoint Group - A Quantum Sky Company.
- SECURITY: the pipeline workflows ran untrusted pull request code on a self-hosted runner with
  Azure credentials and `WIN_PASSWORD` in job scope. The credentialed job is now restricted to
  pull requests raised from a branch of this repository, and the actions it uses are pinned.
  Addresses ansible-lockdown/Windows-2022-STIG#18.

## Release 2.4.0

April 2025 Release Updates
  - Updated RuleID, as version numbers were incremented to the next whole number.
  - Rule numbers updated throughout due to changes in content management system.


## Release 2.3.0

January 2025 Release Updates
  - Updated Check Text for WN22-SO-000360.
  - WN22-DC-000405 - Added requirement for MSFT DC Certificate Authentication Review vulnerability.
  - WN22-DC-000406 - Added requirement for Named-Based strong mappings for certificates.
  - Updated RuleID, as version numbers were incremented to the next whole number.
  - Rule numbers updated throughout due to changes in content management system.

## Release 2.2.0

November 2024 Release Updates
  - Updated RuleID, as version numbers were incremented to the next whole number.
  - Rule numbers updated throughout due to changes in content management system.
  - Updated License to reflect Tyto acquisition. 

## Release 2.1.0

July 2024 Release Updates.