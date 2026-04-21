# Changelog

## 2026 ALD Demo

April 2026 Updates
  - Removed conditional check on Gather Distribution Info task to ensure facts are always freshly gathered.
  - Updated OS check assertion to use `is search()` test instead of `regex_search` filter to return a proper boolean and avoid Ansible 2.19 NoneType deprecation warning.
  - Added `platform` to gather_subset for reliable fact collection in AAP.
  - Replaced string-based OS check with version-based detection using `os_family`, `os_installation_type`, and `distribution_version` (build 10.0.20348 = Server 2022).

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