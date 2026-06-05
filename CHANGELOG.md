# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).
Log every update here **before committing** — the changelog entry and the code change go in the same commit (see CLAUDE.md Rule 5).

## [Unreleased]

### Added
- `docs/research/Architecture_design.md` — consolidated Software Architecture Document (SAD) for the IDP system, a buildable reference for developers. Synthesizes all five research docs (workflow/governance/scope/cost-finops/operations-llmops) into 14 sections: principles, conceptual/context, logical components & event/data contracts, data & storage, execution & messaging, security, runtime governance (LLMOps), cost (FinOps), integration & API contracts (fills the integration gap with concrete REST endpoints), deployment, NFRs, unified phased roadmap, and ADR-1..16 + glossary. Based on workflow.md; preserves ADR-10 (deterministic) and ADR-11 (full HITL). Authored via parallel section-writer agents [idp]
- `docs/research/operations-llmops.md` — runtime/operational governance (LLMOps) for the IDP system, derived from the 2026 AI-operations market report; enumerates the parts that need governance/operations (control plane, observability, eval gates, change control/kill-switch, registry, standards mapping) with handling for each, reconciled to existing architecture (Gateway as control plane, full-HITL as guardian, ADR-10 routing deferral) and outsource-project altitude. Includes operational reference diagram + SRE runbook checklist. Distinct from data-governance `governance.md` [idp]
- `docs/research/cost-finops.md` — financial constraints & FinOps cost-control for the IDP system, derived from the 2026 AI cost-control market report; translates enterprise FinOps down to outsource-project altitude, links cost to the accuracy–cost Pareto tradeoff, and maps cost levers onto the existing architecture (Gateway enforcement, caching, region-type dispatch) while reconciling with ADR-10 [idp]
- `.claude/settings.json` PreToolUse hook that blocks `git commit` when `CHANGELOG.md` is not staged — Claude Code equivalent of the Kiro `check-changelog` hook [all]

### Changed
- Refined CLAUDE.md guideline: changelog must be updated **before commit** (same commit), and "Adding a New Agent" reworked into "Adding a New Feature" [all]
- Moved `CLAUDE.md` into the IDP project root [all]
