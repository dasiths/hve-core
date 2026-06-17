# feat(agents): VEX Generation Agent — AI-Assisted Vulnerability Triage for Any Codebase #1221

- **URL**: https://github.com/microsoft/hve-core/issues/1221
- **Author**: @dasiths
- **Labels**: enhancement, security, needs-triage
- **Milestone**: v3.10.0

## Description

### Overview

Add a new Copilot agent to the `security` collection that enables any hve-core consumer to scan their own codebase for dependency vulnerabilities, perform AI-assisted exploitability analysis, and produce OpenVEX-compliant documents — all using local CLI tools and plain HTTP fetches, with no MCP server dependencies.

This agent is a general-purpose security tool shipped as part of hve-core's security plugin. Once it exists, it also enables the [VEX Workflow proposal](https://github.com/microsoft/hve-core/issues/1220) for hve-core's own releases — but that's a secondary benefit, not the primary goal.

### Motivation

**For hve-core consumers (primary):**

- Teams adopting hve-core already get security planning (`security-planner`), OWASP code review (`security-reviewer`), and supply chain posture assessment (`sssc-planner`). But there's a gap between knowing your dependencies have CVEs (Dependabot, Trivy) and communicating whether those CVEs are actually exploitable in your context (VEX).
- This agent closes that gap. A developer runs `/vex-scan`, points it at their codebase, and gets a draft VEX document with evidence-backed exploitability determinations — the same kind of analysis that currently requires a dedicated security engineer.
- The output is an OpenVEX JSON file that can be shared with consumers, auditors, and compliance tools (Grype, Trivy) to suppress false-positive CVE alerts downstream.

**For hve-core itself (secondary):**

- Once this agent ships in the security collection, the [manual VEX workflow proposal](https://github.com/microsoft/hve-core/issues/1220) becomes implementable: maintainers use `/vex-scan` against hve-core's own dependencies, review the draft, and commit the final document to `security/vex/hve-core.openvex.json` as part of the release pipeline.

**Inspiration**: [dasiths/ai_generated_vex](https://github.com/dasiths/ai_generated_vex) demonstrates this pattern using MCP servers (`trivy-mcp`, `vexdoc-mcp`, `osv-mcp`). This proposal adapts the concept for hve-core's agent architecture with a no-MCP-dependency constraint, preferring local CLI tools and direct REST API calls.

### How It Fits in hve-core

#### Within the Security Collection

| Agent             | Purpose                                                      | Relationship to VEX                                          |
|-------------------|--------------------------------------------------------------|--------------------------------------------------------------|
| security-reviewer | OWASP Top 10 assessment of application code                  | Does NOT scan dependencies for CVEs or produce VEX           |
| security-planner  | STRIDE threat modeling, security plan generation              | Strategic planning — no vulnerability scanning               |
| sssc-planner      | Supply chain posture assessment (OpenSSF, SLSA)              | Assesses process maturity — doesn't produce VEX artifacts    |
| vex-generator (new) | Dependency CVE scanning → exploitability analysis → OpenVEX output | Tactical: scans actual dependencies, determines real exploitability, produces machine-readable VEX |

#### How consumers invoke it

```text
# Mode 1: Full scan of your codebase
/vex-scan

# Mode 2: Triage an existing scan report or SBOM
/vex-triage
```

The agent asks for product name, scope (directories), and optionally an existing Trivy JSON or SBOM file, then runs autonomously through scan → enrich → analyze → generate.

### Agent Architecture

#### MCP-Free Tool Strategy

The key constraint is no MCP server dependencies. Every tool from the `ai_generated_vex` reference project is replaced with a local CLI or HTTP alternative:

| Original (MCP-based)            | Replacement                               | Implementation                                                     |
|---------------------------------|-------------------------------------------|--------------------------------------------------------------------|
| trivy-mcp (MCP server)          | Trivy CLI via runCommands                 | `trivy fs --format json --scanners vuln,misconfig,secret {scope}`  |
| osv-mcp (Docker MCP server)     | OSV.dev REST API via fetch                | `POST https://api.osv.dev/v1/vulns/{CVE-ID}`                      |
| vexdoc-mcp (MCP server)         | Agent writes OpenVEX JSON directly via editFiles | Uses embedded OpenVEX spec from skill — no vexctl or Node.js needed |

#### Supplementary CVE Data Sources (all via `fetch`, no dependencies)

| Source             | Endpoint                                                    | Data Provided                          |
|--------------------|-------------------------------------------------------------|----------------------------------------|
| OSV.dev            | `https://api.osv.dev/v1/vulns/{id}`                        | Affected versions, references, severity |
| NVD API 2.0        | `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId={id}` | CVSS scores, CWE classification        |
| GitHub Advisory DB  | `https://api.github.com/advisories/{GHSA-ID}`              | GitHub-specific severity, patched versions |

### Workflow

```text
User provides: scope, product name
         │
         ▼
Phase 1: Vulnerability Scan
  └─ runCommands: trivy fs --format json --scanners vuln,misconfig,secret {scope}
  └─ Parse JSON → CVE inventory with package PURLs
         │
         ▼
Phase 2: CVE Enrichment
  └─ For each CVE-ID:
     └─ fetch: OSV.dev → vulnerability mechanism, affected ranges
     └─ fetch: NVD API → CVSS vector, CWE, description
  └─ Build enriched CVE profiles
         │
         ▼
Phase 3: Exploitability Analysis (per-CVE, via cve-analyzer subagent)
  └─ Code reachability: codebase + search to trace import → usage → entry point
  └─ Attack vector assessment: network exposure, auth requirements, input validation
  └─ Environmental context: deployment protections, runtime mitigations
  └─ Evidence collection: code snippets, configs, execution paths
  └─ Status determination: not_affected | affected | fixed | under_investigation
         │
         ▼
Phase 4: Report Generation
  └─ editFiles: OpenVEX JSON document
  └─ editFiles: Security assessment report (markdown)
  └─ editFiles: Executive summary
  └─ Output to: docs/security/reports/{report-name}/
```

### Artifact Placement

```text
.github/
├── agents/security/
│   └── vex-generator.agent.md                ← orchestrator
│   └── subagents/
│       └── cve-analyzer.agent.md             ← per-CVE deep analysis
├── prompts/security/
│   └── vex-scan.prompt.md                    ← /vex-scan command
│   └── vex-triage.prompt.md                  ← /vex-triage command
├── instructions/security/
│   └── vex-generation.instructions.md        ← status logic, evidence requirements
├── skills/security/
│   └── openvex-spec.skill.md                 ← OpenVEX schema reference
```

### Agent Design

#### `vex-generator.agent.md` — Orchestrator

```yaml
---
name: vex-generator
description: "Scans dependencies for vulnerabilities, analyzes CVE exploitability against the
codebase, and generates OpenVEX documents using local CLI tools - Brought to you
by microsoft/hve-core"
tools: ['codebase', 'search', 'editFiles', 'fetch', 'runCommands', 'think', 'agent']
agents:
  - cve-analyzer
---
```

- Runs Trivy CLI scans via `runCommands`
- Fetches CVE details from OSV.dev and NVD via `fetch`
- Delegates per-CVE deep analysis to `cve-analyzer` subagent
- Assembles final OpenVEX JSON directly (no `vexctl` or MCP needed)
- Writes all output to `docs/security/reports/{report-name}/`

#### `cve-analyzer.agent.md` — Per-CVE Subagent

```yaml
---
name: cve-analyzer
description: "Deep exploitability analysis for a single CVE against the current codebase -
Brought to you by microsoft/hve-core"
tools: ['codebase', 'search', 'fetch', 'think']
agents: []
disable-model-invocation: true
---
```

- Receives enriched CVE profile from orchestrator
- Traces code reachability using `codebase` and `search`
- Determines VEX status with evidence-backed justification
- Returns structured finding for inclusion in final VEX document

### Consumer Value

The agent produces three deliverables for any codebase it's pointed at:

| Deliverable        | File               | Audience                                           |
|--------------------|--------------------|-----------------------------------------------------|
| Executive summary  | summary.md         | Management, security leads                          |
| Technical report   | yyyy-mm-dd-report.md | Security engineers, developers                     |
| OpenVEX document   | vex.json           | Automated tooling (Grype, Trivy), auditors, downstream consumers |

The OpenVEX document is the key differentiator: it's machine-readable, spec-compliant, and can be fed back into Trivy/Grype to suppress false-positive CVE alerts:

```bash
# Consumer uses the VEX to filter scan results
trivy fs --vex vex.json --format table .
```

### Enabling the hve-core VEX Workflow

Once this agent ships, the [VEX Workflow proposal](https://github.com/microsoft/hve-core/issues/1220) becomes straightforward:

1. hve-core maintainers run `/vex-scan` against the repository before a release
2. Review the draft VEX document for accuracy
3. Commit the reviewed document to `security/vex/hve-core.openvex.json`
4. The release pipeline attests and uploads it alongside existing SBOM artifacts

The agent produces the draft; humans provide the sign-off. The VEX workflow issue handles the release integration (attestation, upload, documentation).

### Implementation Plan

#### Phase 1: Core Agent + Skill

- Create `openvex-spec.skill.md` — OpenVEX schema reference, status definitions, justification codes
- Create `vex-generation.instructions.md` — evidence requirements, status logic, report templates
- Create `cve-analyzer.agent.md` subagent
- Create `vex-generator.agent.md` orchestrator

#### Phase 2: Prompts + Collection Integration

- Create `/vex-scan` prompt (Mode 1: full pipeline)
- Create `/vex-triage` prompt (Mode 2: from existing report)
- Add all artifacts to security collection with `maturity: experimental`
- Add `openvex` to `.cspell/general-technical.txt`

#### Phase 3: Testing

- Test against a known-vulnerable project (e.g., intentionally outdated dependencies)
- Validate OpenVEX output against spec
- Test `/vex-triage` with existing Trivy JSON and SPDX SBOM inputs
- Tune exploitability analysis prompting for accuracy

#### Phase 4: Documentation

- Agent documentation in `docs/agents/security/`
- Update `plugins/security/README.md` with new agent, commands, and skill
- Add usage examples to `docs/hve-guide/roles/security-architect.md`
- Create `docs/security/vex-verification.md` for consumers

### Prerequisites for Consumers

| Tool              | Purpose                  | Installation                                |
|-------------------|--------------------------|---------------------------------------------|
| Trivy (v0.63.0+)  | Vulnerability scanning   | `brew install trivy` / `apt install trivy`  |

That's it. No Docker, Go, Node.js, or MCP servers required.

### Open Questions

1. **Output location**: `docs/security/reports/` (persistent, git-friendly) vs. `.copilot-tracking/security/vex/` (ephemeral tracking)?
2. **Confidence thresholds**: Should the agent flag CVEs where exploitability confidence is below a threshold for mandatory human review?
3. **SBOM input support**: Should Mode 2 (`/vex-triage`) accept SPDX JSON SBOMs in addition to Trivy JSON reports?
4. **OWASP integration**: Should the agent optionally invoke `security-reviewer` for application-level vulnerability discovery alongside dependency CVE scanning?

### References

- [dasiths/ai_generated_vex](https://github.com/dasiths/ai_generated_vex) — Inspiration (MCP-based approach)
- [OpenVEX Specification](https://openvex.dev/)
- [OSV.dev API](https://osv.dev/docs/)
- [NVD API 2.0](https://nvd.nist.gov/developers/vulnerabilities)
- [Trivy CLI Documentation](https://aquasecurity.github.io/trivy/)
- [VEX Workflow Proposal](https://github.com/microsoft/hve-core/issues/1220) — manual VEX maintenance for hve-core releases (enabled by this agent)
- [SBOM Verification Guide](https://github.com/microsoft/hve-core/tree/main/docs/security/sbom-verification.md) — existing SBOM consumer documentation pattern

---

## WilliamBerryiii's Comment: Cross-link + integration notes for the workflow side (#1220)

> Strong support for this agent. After auditing how it would plug into hve-core's existing release pipeline, this agent is the direct enabler of Phase C in the autonomy architecture I just posted on #1220 — the two issues are complementary tracks (tooling here, workflow there), not overlapping.

### Answers to the open questions (from the workflow-consumer perspective)

**Q1: Output location — `docs/security/reports/` vs `.copilot-tracking/security/vex/`?**

Both, with different roles:

| Path                                      | Lifecycle               | Role                                    | Contents                                                             |
|-------------------------------------------|-------------------------|-----------------------------------------|----------------------------------------------------------------------|
| `.copilot-tracking/security/vex/{date}/`  | Ephemeral, git-ignored  | `/vex-scan` working dir                 | Raw Trivy JSON, enriched CVE profiles, per-CVE subagent transcripts, draft prose |
| `docs/security/reports/{report-name}/`    | Persistent, git-tracked | Final consumer-facing artifacts         | summary.md, yyyy-mm-dd-report.md, vex.json                          |
| `security/vex/{product}.openvex.json`     | Persistent, signed at release | The canonical VEX                  | Single source of truth, attested by release-stable.yml               |

The third row is what #1220 attests and uploads. The agent should write its draft VEX to the canonical path so the PR it opens directly modifies the file the release pipeline already knows about — no extra "promotion" step.

**Q2: Confidence thresholds for mandatory human review — yes, and they should be enforced in the agent instructions**

Recommend the 5-band routing table from #1220 §4 as the canonical contract. Key invariants the agent should enforce:

- **Forbidden**: drafting `not_affected` at low or unknown reachability confidence
- **Default for ambiguity**: `under_investigation` (safe, fully retractable in OpenVEX)
- **Required evidence for `not_affected`**: at least one code citation (file + line range) or explicit mitigation reference
- **Required evidence for `affected`**: reachable execution path or runtime invocation evidence

This makes the agent's output safely mergeable even when reviewers skim — the worst case is over-conservative `under_investigation`, never a false `not_affected`.

**Q3: SPDX SBOM input for `/vex-triage` — yes, strongly**

hve-core already produces SPDX-JSON via `anchore/sbom-action@v0.24.0` in `release-stable.yml`. Mode 2 should accept SPDX SBOM as a first-class input alongside Trivy JSON.

This is what makes the agent usable by the #1220 workflow without any extra translation step — `vex-detect.yml` already has the SBOM, just hands it off.

Suggested input precedence:

1. Trivy JSON report (richest — has scan-time vulnerability data already)
2. OSV-Scanner JSON (second-richest — package + advisory data)
3. SPDX-JSON SBOM (requires the agent to fetch advisories from OSV/NVD itself)

**Q4: Optional `security-reviewer` invocation — defer to a follow-up**

Keep the initial agent focused on dependency CVEs. Application-level OWASP findings have a different evidence model (code patterns vs. version ranges) and a different VEX product identifier (the application itself, not its dependencies). Worth a separate proposal once both agents are stable.

### Design refinements suggested by the workflow review

**Licensing posture for CVE data sources** — already covered by the no-MCP, REST-only design, but worth making explicit in `vex-generation.instructions.md`:

| Source                   | License                       | Safe usage                                                    |
|--------------------------|-------------------------------|---------------------------------------------------------------|
| OSV.dev                  | CC0                           | Drafted summaries, references, affected ranges — safest source |
| NVD API 2.0              | US Gov public domain          | CVSS, CWE — safe                                              |
| GitHub Advisory DB        | CC-BY-4.0 (attribution required) | Use only as a `references[]` URL, do not quote prose         |

Recommend the agent prefers OSV.dev for any drafted prose to sidestep attribution requirements entirely.

**Forbidden-transitions list** — formalize in `cve-analyzer.agent.md`:

- `unknown reachability` → `not_affected` ❌
- `unknown reachability` → `affected` ❌
- Either uncertain case must produce `under_investigation`

**Author-of-record contract** — when the agent's PR is merged, the merge commit author becomes the OpenVEX accountable author, and the release workflow's Sigstore identity is the trust anchor. The agent itself is never the author of record. Worth documenting in `vex-generation.instructions.md` so the contract is explicit.

**Maturity ramp** — concur with shipping at `experimental`. Suggested promotion criteria for `stable`:

- Used successfully against ≥3 real codebases (one of which should be hve-core itself, closing the loop with #1220)
- False-positive rate on `not_affected` ≤5% based on human review feedback
- OpenVEX output validates against spec on every run

### Suggested phasing alignment with #1220

| #1221 Phase                          | #1220 Phase needed in parallel            | Notes                                                                 |
|--------------------------------------|-------------------------------------------|-----------------------------------------------------------------------|
| Phase 1: Core agent + skill          | (none)                                    | #1221 prerequisite                                                    |
| Phase 2: Prompts + collection integration | (none)                               | #1221 prerequisite                                                    |
| Phase 3: Testing                     | #1220 Phase A (plumbing), #1220 Phase B (detection) | Can run in parallel — Phase A/B don't depend on the agent            |
| Phase 4: Documentation               | #1220 Phase C (AI drafting)               | Phase C lights up automatically once #1221 reaches experimental maturity |

This means #1220 Phase A and B can ship immediately without blocking on #1221, and #1221 can develop independently. The two tracks converge at #1220 Phase C.

### Bottom line

> This agent is the right tool, with the right architecture (no MCP, local CLI + REST, hve-core-native), to make #1220 fully autonomous on the drafting side. With the confidence-routing contract enforced in agent instructions and OSV-preferred prose, the human-touch budget for hve-core's own VEX maintenance drops to PR review only — estimated ~20 min/month at current Dependabot volume.
>
> Happy to help draft `vex-generation.instructions.md` with the confidence-routing rules, forbidden-transitions list, and licensing posture if that's useful.

Cross-references: #1220 (VEX Workflow — the consuming release pipeline integration).
