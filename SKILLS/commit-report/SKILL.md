---
name: commit-report
description: Commits and pushes code with a plain-language commit body, saves a dated
  work-report file under docs/work-reports, then prints a WhatsApp-ready status
  block. This skill should be used when the user says "code push", "code push kar de",
  "code push kar do", "push the code", "commit and push", "commit kar do", "aaj ka
  report bana de", "work report bana de", or asks for today's work summary.
agent_created: true
allowed-tools: Bash, Read, Grep, Glob, Write, Edit
---

# Commit and report

## Purpose

Commit the current work, push it, and produce a short update that a non-technical
reader understands without asking follow-up questions.

## When to use

Trigger when the user asks to commit, push, or share a status update about code.

## Steps

1. Inspect the repository state.
   - Run `git status --short` and `git rev-parse --abbrev-ref HEAD`.
   - If nothing is staged and nothing is modified, stop and report that there is
     nothing to commit. Never create an empty commit.

2. Understand the change before describing it.
   - Run `git diff` and `git diff --cached` and read the actual content.
   - Identify the user-visible outcome of the change, not just which files moved.

3. Write the commit message with the report in the body.
   - Subject in imperative English, maximum 72 characters, no trailing period.
   - Body is required and carries the plain-language report in plain text
     (no WhatsApp markup, no `*bold*`, no file names, no class names):
     What changed:
     - <outcome, one line>
     - <outcome, one line>
     Status: <Done | In progress>
     Next: <one line>
   - Keep the body outcome-focused for a non-technical reader.

4. Check guardrails before pushing.
   - Never use `--force`, `--force-with-lease`, or `--no-verify`.
   - If the current branch is `main` or `master`, confirm with the user first.
   - If the diff touches `.env`, credentials, keys, or secrets, stop and warn.
   - Do not run tests, build, or compile commands unless the user explicitly asks.

5. Commit and push.
   - Stage the intended files, commit with the subject plus the report body, then push.
   - If the branch has no upstream, use `git push -u origin <branch>`.
   - If the user asked only for a report ("aaj ka report bana de") without
     asking to commit or push, skip the commit and push. Build the report from
     `git log --since=midnight --oneline`, `git diff --stat`, and `git diff`
     instead.

6. Save the work-report file.
   - Path is `docs/work-reports/YYYY-MM-DD.md` relative to the repo root,
     using the local calendar date.
   - Create `docs/work-reports` when it does not exist.
   - When the file for today already exists, append a new entry with time and
     commit hash instead of overwriting.
   - Each entry contains the commit subject plus What changed, Status, and Next.

7. Print the report as one plain-text block so it can be copied in a single action.

## Report format

Plain text only, using WhatsApp syntax: `*bold*`, `_italic_`, triple backticks for
monospace. No markdown headings, no tables, no emoji. Maximum 12 lines.

````
*<Project name> - <DD Mon>*
```<commit subject>```

What changed:
- <outcome, one line>
- <outcome, one line>

Status: <Done | In progress>
Next: <one line>
````

## Writing for a non-technical reader

Describe what the change does for the person using the product, not how it was built.

| Instead of this | Write this |
| --- | --- |
| Refactored JwtAuthenticationFilter | Login is more secure - expired sessions now end on their own |
| Added index on orders.created_at | Order history loads faster |
| Fixed NPE in PaymentService | The payment screen no longer crashes on failed payments |
| Migrated schema v3 to v4 | Behind-the-scenes database cleanup, no action needed |
| Added DTO validation | Forms now show a clear error before submission |
| Bumped Spring Boot to 3.3 | Routine security and stability update |

Rules for the report:

- Lead with the outcome. Add the technical term in parentheses only if it builds trust.
- Replace jargon with the everyday word: endpoint becomes screen or page, deployment
  becomes released, validation becomes checks, refactor becomes cleanup.
- One idea per line. If a line needs a comma and a second clause, split it.
- State impact in plain words: faster, safer, fixed, removed, now works, no longer crashes.
- Never mention file names, class names, dependency names, or stack traces.
- If a change has no user-visible effect, write "Internal cleanup, no visible change".
- Keep the whole report under 12 lines.


