---
name: commit-report
description: Commits, pushes changes, or creates a non-technical daily work report. Triggers on "code push", "commit and push", "work report", "report bana de", "aaj ka report".
allowed-tools: Bash, Read, Write
---

# Workflow

1. Check repo state via `git status --porcelain` and unpushed commits.
2. **Branching Logic:**
   - **Dirty / Unpushed:** Inspect diff -> prevent committing `.env`/secrets -> commit with clear imperative message -> push to upstream.
   - **Clean & Up-to-date:** Do not commit/push. Read today's history via `git log --since=midnight --oneline`.
3. Save/append report to `docs/work-reports/YYYY-MM-DD.md` using current local date.
4. Print the final block in chat for direct copy-paste.

# Report Guidelines

- **Audience:** Non-technical manager / client.
- **Strictly banned words:** `DAO`, `JDBC`, `DTO`, `Entity`, `Collation`, `PR`, `Migration`, `Refactor`, `jakarta`.
- Describe only end-user capability, security, or speed impact.

# Output Schema

# <Project Name> Work Report - <DD Mon YYYY>

## <Feature / Task Name>

What changed:
- <Outcome: What is now enabled, blocked, or visible to the user>
- <Outcome: What performance, reliability, or safety issue was fixed>

Status: <Done | In progress>
Next: <One line next step>