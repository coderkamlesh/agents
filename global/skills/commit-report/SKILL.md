---
name: commit-report
description: Commits, pushes changes, or creates a non-technical daily work report. Triggers on "code push", "commit and push", "work report", "report bana de", "aaj ka report".
allowed-tools: Bash, Read, Write
---

# Workflow

1. **Check Repository State:**
   - Run `git status --porcelain` and check unpushed commits.
2. **Commit & Push (Code Only):**
   - **CRITICAL RULE:** NEVER stage, commit, or push files under `docs/work-reports/`. Reports must remain strictly local.
   - Do NOT use blanket `git add .` or `git add -A`. Stage only relevant source code files.
   - If `docs/work-reports/` is accidentally staged, immediately unstage it via `git reset docs/work-reports/` before committing.
   - Inspect diff -> verify no `.env` or secret leaks -> commit with clear imperative message -> push to upstream.
   - **If Clean & Up-to-date:** Skip commit/push. Read today's history via `git log --since=midnight --oneline`.
3. **Generate Local Work Report:**
   - Write or append the report to `docs/work-reports/YYYY-MM-DD.md` using current local date.
   - Double check `git status` to ensure the report file remains untracked/unstaged in Git.
4. **Output:**
   - Print the final block in chat for direct copy-paste.

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
