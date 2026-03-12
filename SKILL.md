---
name: work-tracker
description: Tracks work in progress across git branches, worktrees, and non-code tasks. Maintains a persistent work log and WIP dashboard. Use this skill at the START of EVERY conversation to detect context and resume tracking. Also use it whenever the user asks about work status, says something is done, mentions working on something new, references JIRA tickets or Confluence docs, or asks for an annual summary. This skill should activate even if the user doesn't explicitly mention "work tracker" -- any conversation where code work or task work is happening should be tracked.
---

# Work Tracker

Track work in progress across branches, worktrees, and non-code tasks. Maintain a persistent work log
for annual performance reviews.

**Announce at start:** A single quiet line. Nothing more.

## Directory Structure

All data lives in `~/.config/work-log/`. Create this directory and any missing files on first run.

```
~/.config/work-log/
  wip.md                     # Active work items dashboard
  config.md                  # Project aliases, GitLab settings (optional)
  2025-work-log.md           # Calendar year logs (one per year)
  2026-work-log.md
  archive/
    2025-annual-summary.md   # Past year executive summaries
```

## Core Principles

1. **Log eagerly, not lazily.** Write to disk immediately after each meaningful action. Never wait for
   "conversation end." Conversations may stay open for weeks across multiple terminal tabs.
2. **Be quiet.** Output a single one-liner confirmation after each update. No narration, no chatter,
   no "I'm now reading your WIP file."
3. **Re-read before every write.** Always read `wip.md` fresh before updating it. Multiple concurrent
   conversations in different tabs may be writing to the same file.
4. **Projects are the unit of identity, not repos.** The same project can span multiple repos and tickets.
   Unrelated work in the same repo is tracked as separate projects.

## Conversation Start Procedure

Run this at the beginning of every conversation. Keep it fast and silent.

### Step 1: Ensure Data Directory Exists

```bash
mkdir -p ~/.config/work-log/archive
```

If `wip.md` doesn't exist, create it with `# Work In Progress`.
If the current year's work log doesn't exist, create it with `# YYYY Work Log`.

### Step 2: Read Current State

Read `~/.config/work-log/wip.md` to know what's currently in flight.

### Step 3: Detect Git Context

```bash
git rev-parse --show-toplevel 2>/dev/null     # repo root
git rev-parse --abbrev-ref HEAD 2>/dev/null   # branch name
git rev-parse --git-common-dir 2>/dev/null    # detects worktree
basename "$(git rev-parse --show-toplevel)"   # repo name
```

If in a worktree, also capture `pwd` for the worktree working directory.
If not in a git repo, skip to Step 5.

### Step 4: Match to WIP Item

Compare the detected repo + branch against existing entries in `wip.md`.

**Match found:** Resume tracking silently.

**New branch in a known repo:** Ask briefly:
```
[work-tracker] New branch `feature/new-api` in `backend`. Is this part of an existing project or something new?
```
Wait for the user's answer. Then create or update the WIP item accordingly.

**Unknown repo entirely:** Ask briefly:
```
[work-tracker] First time seeing repo `new-service`. What project is this for?
```

**Ticket number extraction:** If the branch name contains a ticket pattern (e.g., `feature/XYZ-1234-description`),
extract the ticket number. If ACLI is configured, fetch the ticket title and parent epic
(see `references/acli-integration.md`). If not, just record the ticket number and ask the user
for a project name if one isn't already mapped in `config.md`.

### Step 5: No Git Context

If not in a git repo:
- Check `wip.md` for non-git items.
- Do NOT ask the user what they're working on unprompted. Wait for them to mention it or ask for status.
- If the user mentions a task ("I'm preparing the Q2 presentation"), create a WIP item for it.

### Step 6: Output

One line, always:
```
[work-tracker] Tracking: <Project Name> (<Ticket or "no ticket">) | X items in progress
```

Or if no match was made and no question needed:
```
[work-tracker] Ready | X items in progress
```

## Session Opt-Out

The user can pause tracking for the current conversation at any time. This is purely in-memory,
per-conversation state — never persisted to any file. Other tabs are unaffected.

**Pausing:** If the user expresses "stop/pause/skip tracking" in any natural phrasing, immediately
stop all tracking for this conversation (no reads, no writes, no confirmations, completely invisible).
Output: `[work-tracker] Tracking paused for this conversation.`

**Resuming:** If the user asks to resume, re-run the Conversation Start Procedure from Step 1.
Output: `[work-tracker] Tracking resumed. Tracking: <Project> (<Ticket>) | X items in progress`

**Edge case:** If the user pauses tracking but then asks "what's in progress?" — answer the status
query directly, but do not resume automatic tracking unless they explicitly ask to resume.

Closing the tab or starting a new session always starts fresh with tracking enabled.

## Logging During a Conversation

### When to Log

Write to the work log and update `wip.md` immediately after any of these events:

- A git commit is made
- A PR/MR is created or merged — capture the full MR/PR URL into the `MR` field of the WIP item.
  When `gh pr create` or `glab mr create` is run, the URL is printed in the output; capture it directly.
  If the MR was created outside this conversation and only a number is known, construct the URL from
  the repo's remote origin (e.g., `git remote get-url origin`) and the MR/PR number.
- A meaningful code change is completed (file edits, refactors)
- A test suite or build is run
- The user describes work they've done (Confluence edits, presentations, etc.)
- The user explicitly says "log this"

### How to Log

**Every time, before writing:**

1. Re-read `~/.config/work-log/wip.md` (fresh, no cache)
2. Re-read `~/.config/work-log/YYYY-work-log.md` to find today's section

**Update `wip.md`:** Update `Status`, `Last touched`, and any changed fields (branch, worktree).

**Append to `YYYY-work-log.md`:**
- Find or create today's date section (`## YYYY-MM-DD`)
- Find or create the project subsection (`### Project Name (Ticket)`)
- Append a brief bullet describing what was done

The work log is append-only. Never modify past entries retroactively.

Output: `[work-tracker] Updated: <Project Name> (<Ticket>)`

### Log Entry Format

In `YYYY-work-log.md`:

```markdown
## 2026-03-08

### Auth Service Rewrite (XYZ-1234)
- Repo: mycompany/auth-service, branch: feature/oauth2-migration
- Implemented retry logic for token refresh
- Fixed race condition in session invalidation

### Q2 Architecture Presentation
- Created sequence diagrams for the new auth flow
```

Keep entries brief. One line per meaningful action. Include the repo and branch on the first entry
of each day for that project.

## WIP Item Format

In `wip.md`:

```markdown
## XYZ-1234 | Auth Service Rewrite
- **Repo:** mycompany/auth-service
- **Branch:** feature/oauth2-migration
- **Worktree:** ~/projects/.worktrees/oauth2-migration
- **MR:** https://gitlab.com/mycompany/auth-service/-/merge_requests/1234
- **Status:** Migrated token refresh logic. Integration tests still pending.
- **Started:** 2026-02-15
- **Last touched:** 2026-03-07
```

Field rules:
- `Worktree`: only include if the work is in a worktree.
- `Branch`, `Repo`: only include for git-tracked items.
- `MR`: only when a merge request or pull request exists. Store the full URL so it is clickable in the terminal. For GitHub PRs this is the `https://github.com/...` URL; for GitLab MRs this is the `https://gitlab.com/...` URL. If the full URL cannot be determined, fall back to `!number` (GitLab) or `#number` (GitHub).
- `Status`: always brief — one or two sentences.

For non-code work, use a `Type` field instead of Repo/Branch:

```markdown
## <no ticket> | Q2 Architecture Presentation
- **Type:** Documentation
- **Status:** Outline complete, working on diagrams.
- **Started:** 2026-03-01
- **Last touched:** 2026-03-08
```

## On-Demand Commands

Respond to these when the user asks. These are natural language — the user won't use exact phrasing.

### "What's in progress?" / "Show my WIP" / "Status?"

Read `wip.md` and display all active items:

```
## Work In Progress

XYZ-1234 | Auth Service Rewrite
  Branch: feature/oauth2-migration
  Worktree: ~/projects/.worktrees/oauth2-migration
  MR: https://gitlab.com/mycompany/auth-service/-/merge_requests/1234
  Status: Migrated token refresh logic. Integration tests still pending.

<no ticket> | Q2 Architecture Presentation
  Type: Non-code
  Status: Outline complete, working on diagrams.
```

Only show Worktree/Branch/MR lines when the item has them. For items untouched 30+ days,
append: `(Last touched N days ago — run cleanup to check remote status)`

**Pending Reviews:** After displaying WIP items, check for MRs assigned to the user for review.

1. Read `config.md` for the `gitlab-group` setting under `## GitLab Settings`.
2. If `gitlab-group` is configured, run:
   ```bash
   glab mr list --reviewer=@me --group="<gitlab-group>" --output=json 2>/dev/null
   ```
3. If the command succeeds and returns results, display them after the WIP items:

```
## Pending Reviews

- Fix token refresh for OAuth2 flow
  https://gitlab.com/heb-engineering/projects/native-apps/ios/auth-service/-/merge_requests/456
  Opened 2 days ago

- Add unit tests for session manager
  https://gitlab.com/heb-engineering/projects/native-apps/ios/core-lib/-/merge_requests/789
  Opened 4 hours ago
```

4. Compute the "Opened X ago" value from the `created_at` field in the JSON response, relative to now.
   Use the most natural unit: minutes (< 1 hour), hours (< 1 day), or days.
5. If no MRs are pending review, output: `No pending reviews.`
6. If `gitlab-group` is not configured, or `glab` is not available, or the command fails — skip
   the Pending Reviews section silently. Do not warn or error.

### "This is done" / "Mark X as finished" / "I finished X"

1. Identify which WIP item is being finished. If ambiguous, ask.
2. Ask: "Is the whole project done, or just this ticket/task?"
3. If just a ticket: update the WIP item to remove that ticket, keep the project active.
4. If the whole project: write a final summary entry to the work log with a `[COMPLETED]` tag,
   then remove the item from `wip.md`.
5. Output: `[work-tracker] Completed: <Project Name> (<Ticket>)`

Work log entry for completion:

```markdown
### Auth Service Rewrite (XYZ-1234) [COMPLETED]
- Project completed. Migration from session-based auth to OAuth2/OIDC finished.
- Final PR merged, integration tests passing.
```

### "I'm working on X" / "track: X"

Create a new WIP item. Ask clarifying questions only if needed:
- "Does this have a JIRA ticket?"
- "Is this related to an existing project?"

For non-git tasks, create a `Type: Non-code` entry.

Output: `[work-tracker] New item: <Project Name> (<Ticket or "no ticket">)`

### "I updated X on Confluence" / Confluence-related work

Log it as part of an existing project if the user indicates a relationship, or as a standalone
non-code entry otherwise. If ACLI is configured and the user provides a page ID, fetch the page
title to enrich the entry (see `references/acli-integration.md`).

Output: `[work-tracker] Logged: <description>`

### "Generate my annual summary" / "Generate summary for YYYY"

1. Determine the year. Default to current calendar year. Accept an explicit year if provided.
2. Read `YYYY-work-log.md` for the target year.
3. Group entries by project (using the project name, not repo).
4. For each project, produce an executive-style summary:
   - Project name and date range (first entry to last entry)
   - Related tickets (all ticket numbers seen)
   - Repos involved
   - 2-3 sentence narrative: what was done and why it mattered
5. Add an "Other Contributions" section for smaller or non-code items.
6. Add summary statistics (projects completed, repos active in, tickets worked).
7. Write to `~/.config/work-log/archive/YYYY-annual-summary.md`.
8. Also display the summary to the user in the conversation.

Output format:

```markdown
# YYYY Annual Summary

## Major Projects

### Auth Service Rewrite (Feb - Apr YYYY)
Related tickets: XYZ-1234, XYZ-1235, XYZ-1240
Repos: mycompany/auth-service, mycompany/shared-libs

Led migration from legacy session-based auth to OAuth2/OIDC.
Implemented token refresh, session management, and integration test suite.
Reduced auth-related incidents by enabling token rotation.

## Other Contributions
- Q2 Architecture Presentation (Mar YYYY)
- Onboarding documentation updates (Jun YYYY)

## Summary Statistics
- X projects completed
- Active across Y repositories
- Spanning Z JIRA tickets
```

Keep the narrative brief and impact-focused. This is for a performance review — emphasize
outcomes over implementation details.

### "Clean up WIP" / "Cleanup" / "Prune stale items"

Checks all git-tracked WIP items against their remotes to find items that have been merged,
closed, or abandoned outside of a tracked conversation.

**Procedure:**

1. Read `wip.md`.
2. For each git-tracked item (has a `Branch` field), determine the repo directory:
   - If the item has a `Worktree` field, use that path.
   - Otherwise, if you're currently in the item's repo, use the current directory.
   - Otherwise, skip the item (can't reach its remote from here).
3. Run `git fetch origin` once per repo, then check if the branch exists on the remote:
   ```bash
   git ls-remote --heads origin <branch>
   ```
4. If the item has an `MR` field, also check MR/PR state:
   - If the MR field is a full URL, extract the MR/PR number from the URL path.
   - GitLab (URL contains `merge_requests` or field starts with `!`): `glab mr view <number>`
   - GitHub (URL contains `pull` or field starts with `#`): `gh pr view <number>`
   - If the CLI tool isn't available or the command fails, skip this check silently.
5. Classify: **Merged** (branch gone or MR merged), **Closed** (MR closed), **Active** (branch exists, MR open or none), **Unknown** (couldn't reach remote — skip silently).
6. Skip non-git items. Mention at the end: "N non-git items skipped (can't auto-check)."
7. Present findings **one item at a time**:
   ```
   <ticket> | <Project Name>
     Branch: <branch> — not found on remote
     MR: https://gitlab.com/mycompany/repo/-/merge_requests/1234 — merged
     Remove from WIP and log as completed? (y/n)
   ```
   Wait for the user's answer before proceeding to the next item.
8. For confirmed removals: write a `[COMPLETED]` entry to the work log, remove from `wip.md`.
   For items the user wants to keep: leave unchanged.

**Rules:**
- Never auto-remove without asking. Every removal requires explicit confirmation.
- If all items are active: `[work-tracker] Cleanup: all N items still active`
- When done: `[work-tracker] Cleanup: removed N items, M remain`

## Project Grouping

Projects are identified by name, established when the user first answers "what project is this?"
and persisted in `wip.md` and `config.md`.

- Multiple tickets and repos can map to the same project.
- `config.md` stores persistent aliases and settings (create the file when the first alias is established):

```markdown
## GitLab Settings
gitlab-group: heb-engineering/projects/native-apps/ios

## Project Aliases
# XYZ-1234, XYZ-1235, XYZ-1240, auth-service/*  ->  Auth Service Rewrite
# XYZ-1301, XYZ-1315, backend/feature/api-*    ->  API Platform v2
```

- When a new ticket or branch is associated with a project, add it to config.md for automatic
  future matching.
- When ACLI is available, look up the parent epic to suggest groupings
  (see `references/acli-integration.md`).

## Edge Cases

### Branch Switching Without New Conversation

If the user switches branches mid-conversation, the skill won't automatically detect this.
If the user mentions it or a git command reveals a different branch, re-run the matching logic.

### Multiple Worktrees, Same Project

Track them all under the same WIP item. The entry should reflect the most recently active worktree.

## ACLI Integration (Optional)

**IMPORTANT:** Before running any ACLI commands, read `references/acli-integration.md` for the
correct command syntax. Do not guess command structures.

ACLI is entirely optional — if commands fail, silently fall back to manual mode. When available:
- Auto-populated ticket titles and status from JIRA
- Parent epic lookup for automatic project grouping
- Confluence page title lookup by page ID
