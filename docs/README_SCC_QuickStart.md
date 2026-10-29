# Auditing with SCAP Compliance Checker

This role remediates. To measure whether a host actually passes the benchmark, the role can scan
it with SCAP Compliance Checker (SCC), the scanner DISA authors and validates its SCAP content
against.

The audit is part of the role rather than a separate playbook, matching the rest of the Lockdown
fleet, and every control variable defaults false so a plain hardening run is unchanged.

## Requirements

On the control node: `ansible-core` 2.16.1 or later, `pywinrm` in the same Python that runs
Ansible, and the collections in `collections/requirements.yml`.

On the target: Windows Server 2022 reachable with an account in the local Administrators group,
roughly 1 GB free on the system drive, and outbound HTTPS to `dl.dod.cyber.mil`, or
`scc_bundle_url` pointed at an internal mirror.

## Running it

| Variable | Default | Purpose |
|---|---|---|
| `setup_audit` | `false` | stage SCC on the target |
| `run_audit` | `false` | scan before and after remediation |
| `audit_only` | `false` | scan and stop, applying nothing |
| `fetch_audit_output` | `false` | collect the result sets |
| `audit_output_collection_method` | `fetch` | `fetch` to the control node, `copy` to relocate on the node |
| `audit_output_destination` | see defaults | where collected results are written |
| `audit_log_dir` | `C:\ProgramData\audit_logs` | where result sets are written on the node |

Measured hardening run:

```bash
ansible-playbook -i <inventory> site.yml \
  -e setup_audit=true -e run_audit=true -e fetch_audit_output=true
```

Scan only, nothing applied:

```bash
ansible-playbook -i <inventory> site.yml -e audit_only=true -e setup_audit=true
```

Both scans run in a measured hardening pass because the delta is the evidence. A post-remediation
score on its own cannot tell you whether the role fixed findings or whether the image was already
compliant, so scan a host that has not been hardened before, or roll back to a clean snapshot
first.

### Scanner settings

In `defaults/main/audit.yml`: `scc_version`, `scc_bundle_url`, `scc_bundle_sha256`,
`scc_benchmark`, `scc_profile`.

Moving to a new SCC release is a one line change to `scc_version`. The download URL, both nested
zip names and the binary path all derive from it.

`scc_bundle_sha256` is pinned by default to the SHA-256 DISA publishes for the 5.15 Windows
bundle. `win_get_url` validates the TLS certificate, so the download is already authenticated to
the host serving it; the digest is content integrity on top of that, and it is what stops a mirror
serving a different archive. The check is enforced even when the file is already on disk.

DISA publishes a digest for every artifact in a release:

```
https://dl.dod.cyber.mil/wp-content/uploads/stigs/txt/SCC_5.15_Checksums.txt
```

The pin is version specific. Bumping `scc_version` without updating it fails every run until it is
re-pinned, which is deliberate: take the new value from the checksums file for that release. To
skip the check entirely, against a mirror that repackages the archive, set it empty:

```bash
ansible-playbook -i <inventory> site.yml -e setup_audit=true -e scc_bundle_sha256=
```

### Where results go

Each scan writes two files to `audit_log_dir` on the node: a markdown summary and a zip holding
SCC's full output, including the XCCDF results XML and the CKL and CKLB checklists. Filenames
carry the host and a timestamp frozen once per run, so the pre and post sets never collide.

Open findings are keyed by STIG control ID (`WN22-CC-000080`), the same identifier the role uses
for its task names and tags, so a finding maps straight to the task that addresses it.

### The pre and post scans use separate working directories

`scc_root` is scoped by scan label, giving `scc_pipeline_pre` and `scc_pipeline_post`. Cleanup
runs in an `always` block, so a shared path would mean one scan's teardown deleting the other's
tree mid-run. Both scans hit the same host in the same play, so this is routine rather than an
edge case.

### Passing Windows paths on the command line

`-e key=value` interprets backslash escapes, so a Windows path is silently corrupted before the
role ever sees it. `\a` becomes a BEL character, `\t` a tab, and so on:

```
-e audit_output_destination=C:\ProgramData\audit_summaries
   -> C:\ProgramData<BEL>udit_summaries
```

Nothing warns you, the path in your shell history looks correct, and the collection simply writes
nowhere. Use the JSON form, or a vars file:

```bash
-e '{"audit_output_destination":"C:\\ProgramData\\audit_summaries"}'
-e @vars.yml
```

Both preserve the path exactly. This applies to any Windows path passed with `-e`, not just this
variable, and it is the reason `audit_output_destination` is better set in inventory or
`group_vars` than on the command line.

## Choosing which outputs SCC writes

SCC writes a large set of reports by default. The role narrows that to two deliverables plus one
it needs internally, set with `--setOpt` before the scan:

| Option | Default here | What it is |
|---|---|---|
| `allSettingsHTMLReport` | `1` | every rule, human readable |
| `saveDISACKL` | `1` | DISA CKL checklist, for STIG Viewer |
| `saveDISACKLB` | `1` | CKLB checklist, **required**, see below |
| `allSettingsTextReport` | `0` | |
| `nonComplianceHTMLReport` | `0` | failures only |
| `nonComplianceSummaryHTMLReport` | `0` | |
| `nonComplianceSummaryTextReport` | `0` | |
| `allSettingsSummaryHTMLReport` | `0` | |
| `allSettingsSummaryTextReport` | `0` | |
| `keepXCCDFXML` | `1` | XCCDF results XML, kept deliberately |
| `nonComplianceTextReport` | `0` | |
| `enableSummaryViewer` | `0` | index page, earns little with two reports |
| `keepARFXML` | `0` | ARF results |
| `keepCPEXML` | `0` | |
| `keepOCILXML` | `0` | |
| `keepOVALXML` | `3` | **not a boolean**, 3 means do not keep |

That is all sixteen keys the role sets, not a selection from them. `keepXCCDFXML` is set to `1`
rather than left at SCC's default: it is the canonical machine-readable form and most downstream
tooling expects it, so the role pins it rather than inheriting whatever the scanner ships with.

`scc_keep_working_dir` is a separate operator toggle, default `false`. Set it `true` to leave the
roughly 100 MB scanner tree in place after a stage for diagnosis instead of removing it.

**Do not disable `saveDISACKLB`.** The role's tally parses the CKLB, which is JSON. CKL is XML, so
turning CKLB off does not fall back gracefully, it leaves the tally with nothing to read.

### Changing them

Override individual keys with `scc_report_options_override`, not by redefining
`scc_report_options`. Ansible's default hash behavior is `replace`, so redefining the whole dict
silently drops every key you leave out and those revert to SCC's own defaults. The role combines
the two:

```yaml
scc_report_options_override:
  keepXCCDFXML: 0
  nonComplianceHTMLReport: 1
```

or on the command line:

```bash
-e '{"scc_report_options_override": {"nonComplianceHTMLReport": 1}}'
```

`cscc --listOpt` on a staged copy prints every option SCC accepts, well beyond the reporting ones
listed here.

**Size and sensitivity.** The All Settings report is the bulk of the result set and is a full
configuration dump of the scanned host. If you are publishing results anywhere, that is the file
to think about first.

## What the audit guards against

**Directory permissions.** `C:\ProgramData` grants `BUILTIN\Users` read that reaches files, so by
default any local user could read the result sets while they sit on disk. Those name local and
domain accounts and carry their SIDs. The role breaks inheritance on both the working directory
and the result set directory, granting SYSTEM and Administrators only on each. The result set
directory is hardened by the scan itself rather than by the staging step, so a run that reuses a
scanner staged earlier still gets the restricted ACL.

**Scanner cleanup.** Each scan stage works in its own directory, `scc_pipeline_pre` or
`scc_pipeline_post` under `C:\ProgramData`, removed in an `always` block so a failed scan does not
strand roughly 100 MB of scanner on the host.

**Result sets are NOT cleaned up, deliberately.** `audit_log_dir`, by default
`C:\ProgramData\audit_logs`, keeps every scan's summary and zip. They are the output, and
`audit_output_collection_method: copy` exists precisely to leave files on the node.

That means they accumulate indefinitely, and they name local and domain accounts and carry their
SIDs. On a host you scan regularly, prune `audit_log_dir` on whatever schedule your retention
policy calls for. The role will not do it for you.

**Scoping.** See the command line section below.

## Domain controllers and member servers score differently

Scores from two hosts are comparable only when both hold the same role. Two independent causes.

**The benchmark splits by server role.** Of the 279 rules in V2R9, 44 are domain controller only
(`WN22-DC-*`) and 14 are member server only (`WN22-MS-*`). A domain controller is therefore
measured against 265 applicable rules and a member server against 235. The two denominators are
different, so a raw pass count carries no meaning across the pair.

**The role skips account policy controls on domain members.** On a domain-joined host the Default
Domain Policy owns `[System Access]`, so the secedit-backed account policy controls in this role
cannot hold. `prelim.yml` detects domain membership, skips those controls and warns once, rather
than writing settings the domain will immediately overwrite. See the "Domain Members" section of
the role README.

On a standalone host those same controls **do** apply, and the role enforces them, so a standalone
run remediates strictly more than a domain run.

The practical consequences:

- The count of applicable rules differs, so **compare percentages within one server role only**.
- A standalone baseline typically shows more failures, because the account policy controls are in
  scope and unset on a fresh install.
- A domain member may show account policy controls as failures that the role deliberately did not
  attempt. That is the documented skip, not a role defect. Check the role's warning output before
  filing one.
- `prelim.yml` also skips `WN22-MS-000020` and `WN22-MS-000080` when either would sever the
  control connection, and warns in both cases. Those show as open findings by design.

Test whichever configuration you actually deploy. If you run both, scan both: the domain controller
path and the member server path exercise materially different parts of this role.

## SCC command line behavior worth knowing

Two behaviors cost time if you meet them cold, and both fail quietly.

**`-u/--userDir` does not create its directory.** SCC exits with
`user results directory does not exist exiting`.

**Most flags are not combinable.** `cscc --help` marks a few `[Combinable]`; the rest must be
invoked alone. Passing `-u` alongside `--enableBenchmark` makes SCC read the flag itself as the
benchmark name:

```
Benchmark '-u' with version '-u' not found.
Updated enabled status to [ Enabled  ] on 0 of 40 content streams.
```

Exit status is clean, and the only signal is that "0 of 40". A scan then runs against nothing and
reports success. The playbook asserts on the enabled-benchmark listing immediately after scoping
so this cannot pass unnoticed.

All bundled benchmarks are enabled on a fresh install, so scoping with `--disableAll` followed by
`--enableBenchmark` is required, not an optimization.

## Profiles

The SCC 5.15 content for this benchmark declares eleven profiles: the nine
`MAC-1`/`MAC-2`/`MAC-3` by `Classified`/`Public`/`Sensitive` combinations, plus `CAT_I_Only` and
`Disable_Slow_Rules`. The role defaults to `MAC-2_Sensitive`.

Counted from the datastream in the SCC 5.15 bundle: all nine MAC profiles select the same 278
rules and differ only in severity weighting, so the choice among them does not affect coverage.

`CAT_I_Only` selects **no rules at all**. It must never be used as a fast CAT I gate: it returns a
clean and entirely meaningless result.

## Manual checks

The benchmark includes rules with no automated check. SCC reports these as `Not Reviewed` and logs
a warning.

SCC ships a manual-questions template for them, but it is a blank questionnaire: it expects a
reviewer to mark one answer per question and save the completed file into
`Resources/Content/Manual_Questions/Completed_Files/`. Answering those questions is a site
decision, so the playbook does not supply them. Once your organization has completed the file, add
it to the staging step to retire those results.

## Using it in CI

The validation pipelines run `site.yml` with the audit flags set, so the pipelines themselves carry
no scanner knowledge. Results are collected to `audit_output_destination` on the runner.

**Decide what should leave your network before publishing results.** SCC output names local and
domain accounts and includes their SIDs, and the All-Settings report is a full configuration dump
of the scanned host. The open-findings list is also an enumeration of which controls are not
enforced.
