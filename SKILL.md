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
  config.md                  # Project aliases, ACLI settings (optional)
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
5. **ACLI integration is optional.** Everything works without it. ACLI just enriches data.

## Conversation Start Procedure

Run this at the beginning of every conversation. Keep it fast and silent.

### Step 1: Ensure Data Directory Exists

```bash
mkdir -p ~/.config/work-log/archive
```

If `wip.md` doesn't exist, create it with:

```markdown
# Work In Progress
```

If the current year's work log doesn't exist, create it with:

```markdown
# YYYY Work Log
```

### Step 2: Read Current State

Read `~/.config/work-log/wip.md` to know what's currently in flight.

### Step 3: Detect Git Context

```bash
git rev-parse --show-toplevel 2>/dev/null     # repo root
git rev-parse --abbrev-ref HEAD 2>/dev/null   # branch name
git rev-parse --git-common-dir 2>/dev/null    # detects worktree
basename "$(git rev-parse --show-toplevel)"   # repo name
```

If in a worktree, also capture the worktree path:

```bash
pwd    # worktree working directory
```

If not in a git repo, skip to Step 5.

### Step 4: Match to WIP Item

Compare the detected repo + branch against existing entries in `wip.md`.

**Match found:** Resume tracking silently. Output:
```
[work-tracker] Tracking: <Project Name> (<Ticket>)
```

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

That's it. Move on to whatever the user actually wants to do.

## Session Opt-Out

The user can disable tracking for the current conversation at any time. This is purely
in-memory, per-conversation state. It is never persisted to any file. Other conversations
in other tabs are completely unaffected.

### Pausing

If the user says anything like "don't track this," "pause work tracker," "skip tracking,"
"no tracking for this conversation," "stop tracking," etc., immediately stop all tracking
for this conversation:

- No further reads of `wip.md`
- No writes to the work log
- No one-liner confirmations
- No context detection
- Completely invisible

Output exactly:
```
[work-tracker] Tracking paused for this conversation.
```

Then go fully silent. Do not mention work-tracker again unless the user re-enables it.

### Resuming

If the user later says "resume tracking," "turn tracking back on," "start tracking again,"
etc., re-run the Conversation Start Procedure from Step 1 and resume normal behavior.

Output:
```
[work-tracker] Tracking resumed. Tracking: <Project> (<Ticket>) | X items in progress
```

### Rules

- Opt-out is **never written** to `wip.md`, `config.md`, or any file. It exists only in
  the current conversation's memory.
- Closing the tab or starting a new session always starts fresh with tracking enabled.
- The user does not need to use exact phrasing. Any natural language expressing "stop tracking
  for now" should be understood.
- If the user pauses tracking and then asks "what's in progress?" -- that's a direct request
  for WIP status, so answer it. But do not resume automatic tracking unless they explicitly
  ask to resume.

## Logging During a Conversation

### When to Log

Write to the work log and update `wip.md` immediately after any of these events:

- A git commit is made
- A PR/MR is created or merged (capture the MR/PR number into the `MR` field of the WIP item)
- A meaningful code change is completed (file edits, refactors)
- A test suite or build is run
- The user describes work they've done (Confluence edits, presentations, etc.)
- The user explicitly says "log this"

### How to Log

**Every time, before writing:**

1. Re-read `~/.config/work-log/wip.md` (fresh, no cache)
2. Re-read `~/.config/work-log/YYYY-work-log.md` to find today's section

**Update `wip.md`:**
- Update the `Status` field with a brief description of where things stand now
- Update `Last touched` to today's date
- If the branch or worktree changed, update those fields

**Append to `YYYY-work-log.md`:**
- Find or create today's date section (`## YYYY-MM-DD`)
- Find or create the project subsection (`### Project Name (Ticket)`)
- Append a brief bullet describing what was done

**Output:**
```
[work-tracker] Updated: <Project Name> (<Ticket>)
```

One line. Nothing more.

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
- **MR:** !1234
- **Status:** Migrated token refresh logic. Integration tests still pending.
- **Started:** 2026-02-15
- **Last touched:** 2026-03-07
```

Field rules:
- `Worktree` line: only include if the work is in a worktree. Omit for regular branches.
- `Branch` line: only include for git-tracked items. Omit for non-code work.
- `Repo` line: only include for git-tracked items.
- `MR` line: only include when a merge request or pull request exists. Use `!number` for GitLab MRs, `#number` for GitHub PRs. Capture this when a PR/MR is created.
- `Status`: always brief -- one or two sentences describing where things stand.
- For non-code work, add a `Type` field instead of Repo/Branch:

```markdown
## <no ticket> | Q2 Architecture Presentation
- **Type:** Documentation
- **Status:** Outline complete, working on diagrams.
- **Started:** 2026-03-01
- **Last touched:** 2026-03-08
```

## On-Demand Commands

Respond to these when the user asks. These are natural language -- the user won't use exact phrasing.

### "What's in progress?" / "Show my WIP" / "Status?"

Read `wip.md` and display all active items in this format:

```
## Work In Progress

XYZ-1234 | Auth Service Rewrite
  Branch: feature/oauth2-migration
  Worktree: ~/projects/.worktrees/oauth2-migration
  Status: Migrated token refresh logic. Integration tests still pending.

XYZ-1301 | API Rate Limiting
  Branch: feature/rate-limits
  Status: Initial middleware scaffolding done. Need to add Redis backend.

<no ticket> | Q2 Architecture Presentation
  Type: Non-code
  Status: Outline complete, working on diagrams.
```

Rules:
- Show the Worktree line only if the item has a worktree path.
- Show the Branch line only for git-tracked items.
- Status is always the brief summary from `wip.md`.

### "This is done" / "Mark X as finished" / "I finished X"

1. Identify which WIP item is being finished. If ambiguous, ask.
2. Ask: "Is the whole project done, or just this ticket/task?" -- because finishing one ticket
   doesn't necessarily mean the project is complete if other tickets are still open.
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

Example log entry:

```markdown
### Auth Service Rewrite (XYZ-1234)
- Updated architecture runbook on Confluence
```

Or standalone:

```markdown
### Onboarding Documentation (Confluence)
- Rewrote the new hire setup guide
```

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

### API Platform v2 (May - Aug YYYY)
Related tickets: XYZ-1301, XYZ-1315
Repos: mycompany/backend

Designed and built rate limiting middleware with Redis backend.
Introduced tiered API access for external partners.

## Other Contributions
- Q2 Architecture Presentation (Mar YYYY)
- Onboarding documentation updates (Jun YYYY)
- Incident response: production DB failover (Jul YYYY)

## Summary Statistics
- X projects completed
- Active across Y repositories
- Spanning Z JIRA tickets
```

Keep the narrative brief and impact-focused. This is for a performance review -- emphasize
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
3. Check if the branch still exists on the remote:
   ```bash
   git ls-remote --heads origin <branch>
   ```
   Empty output means the branch is gone.
4. If the item has an `MR` field, also check MR/PR state:
   - GitLab (`!number`): `glab mr view <number>` — look at the `state:` line
   - GitHub (`#number`): `gh pr view <number>` — look at the state
   - If the CLI tool isn't available or the command fails, skip this check silently.
5. Classify the item:
   - **Merged**: branch gone from remote, or MR state is `merged`
   - **Closed**: MR state is `closed` (branch may or may not exist)
   - **Active**: branch exists on remote and MR is still open (or no MR)
   - **Unknown**: couldn't reach remote — skip silently
6. Skip non-git items (no `Branch` field). Mention them at the end: "N non-git items skipped (can't auto-check)."
7. Present findings **one item at a time**. For each stale item, show:
   ```
   <ticket> | <Project Name>
     Branch: <branch> — not found on remote
     MR: !1234 — merged
     Remove from WIP and log as completed? (y/n)
   ```
   Wait for the user's answer before proceeding to the next item.
8. For confirmed removals: write a `[COMPLETED]` entry to the work log, remove from `wip.md`.
9. For items the user wants to keep: leave them in `wip.md` unchanged.

**Output when done:**
```
[work-tracker] Cleanup: removed N items, M remain
```

**Rules:**
- Never auto-remove without asking. Every removal requires explicit confirmation.
- If all items are active, output: `[work-tracker] Cleanup: all N items still active`
- Run `git fetch origin` once per repo before checking, to ensure remote refs are current.

## Project Grouping

Projects are identified by name. The name is established when the user first answers
"what project is this?" and persists in `wip.md` and `config.md`.

### How Grouping Works

1. User tells you: "That's the API Platform v2 project." The skill uses "API Platform v2" as the
   project name for all future matching.
2. Multiple tickets accumulate under the same project name across conversations.
3. Multiple repos can map to the same project.
4. `config.md` stores persistent aliases:

```markdown
## Project Aliases
# XYZ-1234, XYZ-1235, XYZ-1240, auth-service/*  ->  Auth Service Rewrite
# XYZ-1301, XYZ-1315, backend/feature/api-*    ->  API Platform v2
```

5. When ACLI is available, the skill can also look up the parent epic to suggest groupings
   (see `references/acli-integration.md`).

### Updating config.md

When a new ticket or branch is associated with a project (through the user's answer or ACLI
epic lookup), add it to the project aliases section in `config.md`. This makes future matching
automatic -- the skill won't need to ask again for known mappings.

## Edge Cases

### Branch Switching Without New Conversation

If the user switches branches mid-conversation (e.g., `git checkout other-branch`), the skill
won't automatically detect this since it checks git context at conversation start. If the user
mentions they switched branches, or if a git command reveals a different branch than expected,
re-run the matching logic and update tracking accordingly.

### Multiple Worktrees, Same Project

A project may have multiple worktrees for different sub-tasks. Track them all under the same
WIP item. The `wip.md` entry should reflect the most recently active worktree.

### Stale WIP Items

If a WIP item hasn't been touched in a long time (30+ days), don't proactively nag about it.
Only mention staleness if the user asks for WIP status -- then note:
```
XYZ-1234 | Auth Service Rewrite
  Branch: feature/oauth2-migration
  Status: Migrated token refresh logic. Integration tests still pending.
  (Last touched 45 days ago — run cleanup to check remote status)
```

To detect and remove items whose branches have been merged or deleted, use the **cleanup**
on-demand command (see "Clean up WIP" under On-Demand Commands).

### No Config File

If `config.md` doesn't exist, the skill works fine. Project grouping relies on the user's
answers stored in `wip.md`. Create `config.md` when the first alias mapping is established.

### ACLI Not Installed or Not Authenticated

If ACLI commands fail, silently fall back to manual mode. Never error out or nag about ACLI.
The skill is fully functional without it.

## ACLI Integration (Optional)

**IMPORTANT:** Before running any ACLI commands, read `references/acli-integration.md` for the
correct command syntax. Do not guess command structures -- the reference file has the exact
invocations needed.

For JIRA and Confluence enrichment via the Atlassian CLI, see `references/acli-integration.md`.

This is entirely optional. The skill works without ACLI. When ACLI is available and configured,
it provides:
- Auto-populated ticket titles and status from JIRA
- Parent epic lookup for automatic project grouping
- Confluence page title lookup by page ID

## Quick Reference

### One-liner outputs:

| Event | Output |
|---|---|
| Conversation start, match found | `[work-tracker] Tracking: Project (Ticket) \| X items in progress` |
| Conversation start, no match | `[work-tracker] Ready \| X items in progress` |
| After logging work | `[work-tracker] Updated: Project (Ticket)` |
| New item created | `[work-tracker] New item: Project (Ticket)` |
| Item completed | `[work-tracker] Completed: Project (Ticket)` |
| Confluence/non-code logged | `[work-tracker] Logged: description` |
| Cleanup completed | `[work-tracker] Cleanup: removed N items, M remain` |
| Cleanup, nothing stale | `[work-tracker] Cleanup: all N items still active` |

### File write checklist:

Every time you write to `wip.md` or `YYYY-work-log.md`:
1. Re-read the file fresh (do not use cached content)
2. Make the update
3. Write the file
4. Output the one-liner confirmation

### Never:

- Wait for conversation end to log work
- Output more than one line of confirmation for routine updates
- Nag about ACLI not being installed
- Proactively ask what the user is working on when there's no git context
- Create WIP items without the user's knowledge (except when auto-matching a known mapping)
- Modify the work log retroactively (it's append-only)

### Always:

- Read `wip.md` fresh before every write
- Log immediately after meaningful actions
- Ask before creating a new project ("Is this part of X or something new?")
- Ask "whole project done or just this ticket?" when marking things finished
- Keep status descriptions brief (1-2 sentences max)
