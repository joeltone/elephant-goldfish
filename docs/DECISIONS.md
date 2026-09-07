# Decisions

<!-- Aggregated from WORKLOG.md. One H2 section per workstream. -->
<!-- Sync via /log command — do not edit manually. -->

## CROSS_MODEL_PROVIDERS

| ID | Decision | Alternatives Rejected | Rationale |
|----|----------|-----------------------|-----------|
| D-001 | Implement "Microsoft Copilot" as Copilot Studio's Direct Line channel, not GitHub Copilot CLI or a full Graph/AAD OAuth app | GitHub Copilot CLI (rejected — no open-ended prompt API, only shell-command suggest/explain); full M365 Graph Copilot OAuth app registration (rejected — too heavy a setup for a fail-soft opt-in pass) | Direct Line needs only a per-bot secret (same shape as DeepSeek's API key), matches the existing fail-soft `secret-tool` pattern, and is the standard way to script against a published Copilot Studio agent |
| D-002 | Centralize provider mechanics in one new shared file (`claude/cross-model-providers.md`) instead of duplicating `curl`/`jq` blocks in each command | Duplicating the DeepSeek-style block inline in all five commands (rejected — five copies to keep in sync, exactly the drift risk the DeepSeek-only precedent already showed) | Single source of truth; each command only needs a short reference + its own prompt content |
| D-003 | Cross-model passes are opt-in per run via `AskUserQuestion`, never auto-run by default (departs from `eg-brainstorm`'s old "always run DeepSeek" behavior) | Keeping DeepSeek auto-run in brainstorm while making Copilot opt-in only (rejected — inconsistent UX, and auto-running an external paid/quota-limited call by default doesn't generalize safely across 5 commands) | Consistency across all five hook points; the user should decide per invocation whether the extra latency/cost of an external call is worth it |
| D-004 | In `eg-new-feature`, the cross-model critic pass is informational only and does not affect the design-ready gate | Making it a third gating vote (rejected — would let an unconfigured/unreliable external provider block implementation, and conflicts with the existing "critic AND readiness" gate definition) | Matches how the comprehension pass is already treated; the value is an independent read, not a veto |
| D-005 | Created a root `CLAUDE.md` + `docs/WORKLOG.md`/`DECISIONS.md` for this template repo itself, which previously had neither | Skipping worklog setup entirely (rejected — user explicitly asked to use the worklog skill); hand-writing worklog files without CLAUDE.md, bypassing the skill's precondition (rejected — user chose the "create CLAUDE.md first" option) | User picked this option directly when asked; keeps the meta-repo's own history in the same format it ships to consumers |
