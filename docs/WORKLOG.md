# Worklog

<!-- Reverse-chronological. Latest entry at top. -->
<!-- Archive to docs/WORKLOG_archive.md when entry count exceeds 10. -->

## CROSS_MODEL_PROVIDERS | 2026-09-07 | Session 001
**Objective:** Add an option to run some model passes through Microsoft Copilot, generalized alongside the existing DeepSeek cross-model lens, across all five `eg-*` Claude commands.

### Investigated
- Confirmed DeepSeek's cross-model lens (added in `4dbd655`) was the only existing precedent — a single-shot `curl` to an OpenAI-compatible chat-completions endpoint, gated by a `secret-tool` lookup, wired only into `eg-brainstorm` Step 2.5 and not yet mirrored anywhere else.
- Clarified with the user which "Microsoft Copilot" was meant: not GitHub Copilot CLI (shell-command suggest/explain only, no open-ended prompt API) or Azure OpenAI, but Microsoft 365 Copilot / Copilot Studio.
- Copilot Studio has no single-call chat-completions equivalent to DeepSeek's. The practical scripting path is the **Direct Line 3.0** protocol against a published Copilot Studio agent's channel secret: start conversation → post activity → poll for a reply (3 calls, asynchronous), keyed by a Direct Line secret rather than a bare API key.
- User confirmed scope should be broader than brainstorm-only: a general "which provider runs this pass" setting across all `eg-*` commands, Claude adapter only (Codex/Gemini not touched — DeepSeek hadn't been mirrored there either).

### Changed
- `claude/cross-model-providers.md` — new shared reference: provider registry (DeepSeek, Microsoft Copilot), the reusable `AskUserQuestion` "which provider" template, both providers' invocation recipes, and the fail-soft skip policy. Single source of truth so a new provider or a fix only needs updating in one place.
- `claude/commands/eg-brainstorm.md` — generalized the old "always run DeepSeek" Step 2.5 into a Q3.5 provider choice (DeepSeek / Microsoft Copilot / both / none); updated the CROSS-MODEL CHECK section and final report to be provider-agnostic.
- `claude/commands/eg-prd.md` — added Q3.5 optional cross-model research lens into the Wave 2 research goldfish step.
- `claude/commands/eg-fix-bug.md` — added optional cross-model second-opinion diagnosis alongside the Step 2 Claude diagnosis goldfish.
- `claude/commands/eg-new-feature.md` — added optional cross-model design critic as "Pass D," explicitly informational (doesn't gate the design ready/not-ready decision, same treatment as the comprehension pass).
- `claude/commands/eg-precommit-review.md` — added optional cross-model reviewer pass, one-shot on round 1 only, findings merged into the existing triage ledger tagged `[Cross-model — <Provider>]`.
- `README.md` — new "Fork note: optional cross-model passes" section documenting setup (`secret-tool store` for both providers) and pointing at `claude/cross-model-providers.md`.
- `CLAUDE.md` — created (repo previously had none); `docs/WORKLOG.md`, `docs/DECISIONS.md`, `docs/vscode-snippets.md` — created via worklog `/init`.

### Decided

| ID | Decision | Alternatives Rejected | Rationale |
|----|----------|-----------------------|-----------|
| D-001 | Implement "Microsoft Copilot" as Copilot Studio's Direct Line channel, not GitHub Copilot CLI or a full Graph/AAD OAuth app | GitHub Copilot CLI (rejected — no open-ended prompt API, only shell-command suggest/explain); full M365 Graph Copilot OAuth app registration (rejected — too heavy a setup for a fail-soft opt-in pass) | Direct Line needs only a per-bot secret (same shape as DeepSeek's API key), matches the existing fail-soft `secret-tool` pattern, and is the standard way to script against a published Copilot Studio agent |
| D-002 | Centralize provider mechanics in one new shared file (`claude/cross-model-providers.md`) instead of duplicating `curl`/`jq` blocks in each command | Duplicating the DeepSeek-style block inline in all five commands (rejected — five copies to keep in sync, exactly the drift risk the DeepSeek-only precedent already showed) | Single source of truth; each command only needs a short reference + its own prompt content |
| D-003 | Cross-model passes are opt-in per run via `AskUserQuestion`, never auto-run by default (departs from `eg-brainstorm`'s old "always run DeepSeek" behavior) | Keeping DeepSeek auto-run in brainstorm while making Copilot opt-in only (rejected — inconsistent UX, and auto-running an external paid/quota-limited call by default doesn't generalize safely across 5 commands) | Consistency across all five hook points; the user should decide per invocation whether the extra latency/cost of an external call is worth it |
| D-004 | In `eg-new-feature`, the cross-model critic pass is informational only and does not affect the design-ready gate | Making it a third gating vote (rejected — would let an unconfigured/unreliable external provider block implementation, and conflicts with the existing "critic AND readiness" gate definition) | Matches how the comprehension pass is already treated; the value is an independent read, not a veto |
| D-005 | Created a root `CLAUDE.md` + `docs/WORKLOG.md`/`DECISIONS.md` for this template repo itself, which previously had neither | Skipping worklog setup entirely (rejected — user explicitly asked to use the worklog skill); hand-writing worklog files without CLAUDE.md, bypassing the skill's precondition (rejected — user chose the "create CLAUDE.md first" option) | User picked this option directly when asked; keeps the meta-repo's own history in the same format it ships to consumers |

### Blocked / Open Questions
- The Copilot Studio invocation in `claude/cross-model-providers.md` is unverified against a real bot/tenant — channel naming ("Custom channel" vs "Mobile app channel") varies by Copilot Studio release and is left as a `[BOOTSTRAP: ...]` note for the user to confirm.
- DeepSeek's cross-model lens (and now Copilot's) has not been mirrored into the Codex or Gemini adapters yet — explicitly deferred per the user's scope answer, not forgotten.

### Next Steps
- [ ] If/when the user has real Copilot Studio credentials, test the Direct Line invocation end-to-end and correct the channel-name `[BOOTSTRAP: ...]` note in `claude/cross-model-providers.md`.
- [ ] Decide whether to mirror the cross-model provider mechanism into `codex/` and `gemini/` adapters (would need adapter-specific invocation syntax per `PROMPTS.md`'s "Propagate Pattern Update" prompt).
