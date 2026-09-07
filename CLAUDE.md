# CLAUDE.md

This is the **elephant-goldfish** template repo: a set of reusable elephant/goldfish
workflow commands/skills (`eg-brainstorm`, `eg-prd`, `eg-new-feature`, `eg-fix-bug`,
`eg-precommit-review`) mirrored across three AI adapters — `claude/` (Claude Code
commands), `codex/` (Codex skills), and `gemini/` (Gemini CLI skills). Consumer repos
bootstrap from here (see `README.md` → "How to install"); this repo itself is the
source of truth those bootstraps pull from, not a bootstrapped consumer project.

See `README.md` for the full pattern explanation, `PROMPTS.md` for the
update/propagate maintenance prompts, and `claude/cross-model-providers.md` for the
optional DeepSeek/Microsoft Copilot cross-model pass mechanics used by the Claude
adapter's commands.

## Working conventions

- Changes to one adapter's logic (e.g. `claude/commands/eg-*.md`) should generally be
  propagated to the other adapters (`codex/skills/eg-*/`, `gemini/commands/eg-*.md`)
  per the "For Maintainers: Syncing Adapters" prompt in `PROMPTS.md` — unless the
  change is explicitly adapter-specific (e.g. a Claude-only feature still being
  proven out).
- Commit messages end with `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>`.

## Session Context

- Read `docs/WORKLOG.md` at session start to regain context on recent work
- Active workstreams are identified by H2 headers in `docs/WORKLOG.md`

### Commands

**`/log [workstream]`**
Log the current session. Steps:
1. Append a new entry to `docs/WORKLOG.md` (reverse-chronological, latest at top)
2. Infer session number from existing entry count + 1 (zero-pad to 3 digits)
3. Use free-text workstream name supplied by user (e.g., `/log BRAND_FUND`)
4. Before assigning any D-xxx IDs, scan both `docs/WORKLOG.md` and
   `docs/DECISIONS.md` for the highest existing D-xxx number. New IDs must
   be globally unique and increment from that maximum (e.g., if D-007 is the
   highest found, next ID is D-008). Never reuse or guess — always scan first.
5. Sync any new D-xxx rows from the new entry into `docs/DECISIONS.md` under the
   matching H2 workstream section (create section if it doesn't exist)
6. Check entry count in `docs/WORKLOG.md`. If > 10: move entries 6–10 to
   `docs/WORKLOG_archive.md` (append, preserve order), keep 5 most recent,
   add footer comment: `<!-- Archived entries in docs/WORKLOG_archive.md -->`
7. If the trigger phrase was "log and commit this" (or similar — the user's
   wording explicitly asks for a commit): stage all session-changed/new files
   and create a git commit after the entry is written. If the trigger was
   plain "/log" or "log session" with no mention of committing, write the
   entry only — do not commit.
8. Output confirmation summary to chat (see format below)

Entry format:
```
## [WORKSTREAM] | YYYY-MM-DD | Session NNN
**Objective:** [one line]

### Investigated
- [bullet, one line each — what was explored, what was found, what was ruled out]
- [dead ends count — "Tried X, didn't work because Y" is a valid entry]

### Changed
- `path/to/file` — [one-line reason]

### Decided

| ID | Decision | Alternatives Rejected | Rationale |
|----|----------|-----------------------|-----------|
| D-NNN | [decision] | [what was rejected] | [why] |

### Blocked / Open Questions
- [unresolved items that will cost time next session]

### Next Steps
- [ ] [immediate next action]
```

Confirmation summary format (output to chat after writing files):
```
Logged: [WORKSTREAM] Session NNN (YYYY-MM-DD)
Scanned: max existing ID was D-NNN, new entries start at D-NNN | no existing decisions found
Decisions synced: [N] (IDs: D-NNN, D-NNN) | none
Archive triggered: yes — entries 6–10 moved to WORKLOG_archive.md | no
Committed: yes (<short sha>) | no — plain /log, no commit requested
```

**`/status`**
Report current worklog state without writing anything:
1. Count entries in `docs/WORKLOG.md`
2. Report last session date and workstream
3. Count total D-xxx rows in `docs/WORKLOG.md`
4. Count total D-xxx rows in `docs/DECISIONS.md`
5. Flag if counts differ (decisions in log not yet synced to DECISIONS.md)
6. Report whether archive threshold (10 entries) has been hit

Output format:
```
Worklog status:
  Entries: N (threshold: 10)
  Last session: [WORKSTREAM] | YYYY-MM-DD | Session NNN
  Decisions in worklog: N | in DECISIONS.md: N [IN SYNC | X UNSYNCED]
  Archive: not triggered | triggered — WORKLOG_archive.md exists
```
