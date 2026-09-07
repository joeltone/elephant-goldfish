# VS Code Snippets

Snippets for this project. Add each block to the relevant VS Code user snippets file
via `Ctrl+Shift+P` → "Snippets: Configure User Snippets".

## Worklog Entry (`markdown.json`)

```json
{
  "Worklog Entry": {
    "prefix": "wlog",
    "description": "Insert a new worklog session entry",
    "body": [
      "---",
      "",
      "## ${1:WORKSTREAM} | ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE} | Session ${2:NNN}",
      "**Objective:** ${3:What you set out to accomplish}",
      "",
      "### Investigated",
      "- ${4:What was explored, found, or ruled out}",
      "",
      "### Changed",
      "- `${5:path/to/file}` — ${6:what and why}",
      "",
      "### Decided",
      "",
      "| ID | Decision | Alternatives Rejected | Rationale |",
      "|----|----------|-----------------------|-----------|",
      "| D-${7:NNN} | ${8:Decision} | ${9:Alternatives} | ${10:Rationale} |",
      "",
      "### Blocked / Open Questions",
      "- ${11:Unresolved items}",
      "",
      "### Next Steps",
      "- [ ] ${12:Immediate next action}",
      "log"
    ]
  }
}
```

Type `wlog` in any `.md` file and trigger IntelliSense (`Ctrl+Space`) to insert
the scaffold. Tab through fields to fill them in.
