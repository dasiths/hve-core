# VEX Implementation Plan — Index

This directory contains the complete plan, validation spec, and source issue details for implementing VEX (Vulnerability Exploitability eXchange) support across hve-core. The work spans two GitHub issues that form complementary tracks: a workflow track (#1220) and a tooling track (#1221).

## Files

| File | Purpose |
|------|---------|
| [vex-workflow-and-agent-plan.instructions.md](vex-workflow-and-agent-plan.instructions.md) | Master implementation plan with 7 phases, dependency graph, design decisions, and deliverables |
| [vex-validation-spec.md](vex-validation-spec.md) | Checkable validation spec (100+ criteria) for verifying each phase's output |
| [issue-1220-vex-workflow.md](issue-1220-vex-workflow.md) | Full issue description and WilliamBerryiii's technical review for #1220 (VEX Workflow) |
| [issue-1221-vex-generation-agent.md](issue-1221-vex-generation-agent.md) | Full issue description and WilliamBerryiii's technical review for #1221 (VEX Generation Agent) |

## How to Use

### Distributing work

The implementation plan defines 7 phases, each scoped for a single developer or agent. Check the **Phase Summary** table and **Phase Dependency Graph** to identify which phases can run in parallel. Phases 1 and 2 have no dependencies and can start immediately. Assign each phase to a developer or agent with a reference to the plan file and the corresponding validation spec section.

### Implementing a phase

1. Read the phase section in `vex-workflow-and-agent-plan.instructions.md` for the task list and deliverables.
2. Read the matching issue file (`issue-1220-vex-workflow.md` or `issue-1221-vex-generation-agent.md`) for WilliamBerryiii's design decisions and constraints.
3. Complete each task item in order within the phase.
4. Validate your work against the matching section in `vex-validation-spec.md` (e.g., Phase 1 tasks validate against "Phase 1: VEX Foundation" criteria).

### Validating a phase

Open `vex-validation-spec.md` and locate the section for the completed phase. Each criterion is a checkbox. Run the specified `npm run` commands and manually verify structural requirements. The "Cross-Phase Validation" section at the bottom covers consistency checks that apply after multiple phases are complete.

### Quick reference

- **Parallel start**: Phases 1 + 2
- **Critical path**: Phase 2 → 3 → 4 → 7
- **Convergence point**: Phase 6 (requires both Phases 3 and 5)
- **Source issues**: [#1220](https://github.com/microsoft/hve-core/issues/1220) (workflow), [#1221](https://github.com/microsoft/hve-core/issues/1221) (agent)
- **Milestone**: v3.10.0
- **Maturity**: All new artifacts ship as `experimental`
