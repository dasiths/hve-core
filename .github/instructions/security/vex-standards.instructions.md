---
description: "OpenVEX format reference, confidence routing, forbidden transitions, licensing posture, and author-of-record contract for VEX document management - Brought to you by microsoft/hve-core"
applyTo: '**/security/vex/**, **/.copilot-tracking/security/vex/**'
---

# VEX Standards

This file defines the authoritative conventions for OpenVEX document management in hve-core. All agents, prompts, and human contributors producing or reviewing VEX artifacts follow these rules.

## Scope

Applies to all files under `security/vex/` and `.copilot-tracking/security/vex/`. Governs confidence routing, status transitions, licensing compliance, and accountability for VEX statements.

## Confidence Routing

The 5-band confidence routing table determines how agents and humans handle each vulnerability assessment based on evidence strength.

| Band | Criteria | Agent Action | Human Action |
|---|---|---|---|
| High: not_affected | Vulnerable symbol provably unreachable (no import path, dead code, or guarded by mitigation) | Drafts `not_affected` with `vulnerable_code_not_in_execute_path` justification and code citations | Approves PR (skim evidence) |
| High: affected | Vulnerable symbol on a reachable execution path | Drafts `affected` and links to remediation issue | Approves PR and triages remediation |
| Medium | Symbol reachable in some configurations but ambiguous (feature flags, optional codepaths, runtime conditional) | Drafts `under_investigation` with structured questions for reviewer | Decides final status, edits PR |
| Low | Cannot determine reachability (closed-source dep, dynamic dispatch, native code) | Drafts `under_investigation` only; forbidden from drafting `not_affected` | Does manual analysis, may downgrade |
| Vendor-disputed | OSV/NVD shows dispute or CVSS < 4.0 with no known exploit | Drafts `not_affected` with `inline_mitigations_already_exist` only when accompanied by code citation | Approves PR |

## Forbidden Transitions

Certain status transitions are forbidden regardless of confidence band. These rules prevent premature closure of vulnerability assessments.

| Transition | Verdict |
|---|---|
| `unknown reachability` to `not_affected` | FORBIDDEN |
| `unknown reachability` to `affected` | FORBIDDEN |

When reachability cannot be determined, the default status is `under_investigation`. Agents must not escalate or close a vulnerability without sufficient evidence.

## Licensing Posture

VEX documents reference vulnerability data from multiple sources. Each source carries distinct licensing obligations.

| Source | License | Usage Guidance |
|---|---|---|
| OSV.dev | CC0 (public domain) | Preferred source for vulnerability data and drafted prose |
| NVD | US Government public domain | Used for CVSS scores and CWE classifications |
| GHSA | CC-BY-4.0 | Avoid quoting advisory prose; use only as a pointer to the advisory URL |

Prefer OSV.dev data when multiple sources provide equivalent information. When referencing GHSA advisories, link to the advisory URL rather than reproducing its text.

## Author-of-Record Contract

VEX documents in this repository follow a draft-and-merge accountability model.

The AI agent drafts VEX statements, including status determinations, justification codes, and supporting evidence. The human reviewer merges the pull request after validating the evidence and confirming the status determination.

The merge commit author is the accountable author of record for the VEX statement. The Sigstore identity attached by the release workflow serves as the trust anchor for published VEX documents.

## Validation

Verify compliance with these standards by checking:

* Confidence band in each VEX triage PR matches the routing table criteria
* No forbidden transitions appear in the VEX document history
* Data source licensing is respected (no quoted GHSA prose)
* Every merged VEX statement has an identifiable human author via merge commit
