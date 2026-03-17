---
name: work-tracker
description: Tracks work in progress across git branches, worktrees, and non-code tasks. Maintains a persistent work log and WIP dashboard. Use this skill at the START of EVERY conversation to detect context and resume tracking. Also use it whenever the user asks about work status, says something is done, mentions working on something new, references JIRA tickets or Confluence docs, or asks for an annual summary. This skill should activate even if the user doesn't explicitly mention "work tracker" -- any conversation where code work or task work is happening should be tracked.
---

# Work Tracker

Track work in progress across branches, worktrees, and non-code tasks. Maintain a persistent work log
for annual performance reviews.

**Announce at start:** A single quiet line. Nothing more.

## CLI Tool

The `wt` command handles all file I/O, git detection, markdown parsing, and data manipulation.
Location: `~/.agents/skills/work-tracker/scripts/wt`

The LLM never reads or writes work-tracker data files directly. Always use `wt`.

## Data Directory

All data lives in `~/.config/work-log/`:

```
wip.md                     # Active work items dashboard
config.md                  # Project aliases, GitLab settings (optional)
YYYY-work-log.md           # Calendar year logs (one per year)
archive/YYYY-annual-summary.md  # Past year executive summaries
```

## Core Principles

1. **Log eagerly, not lazily.** Call `wt log` immediately after each meaningful action. Never wait for
   "conversation end." Conversations may stay open for weeks across multiple terminal tabs.
2. **Be quiet.** Output a single one-liner confirmation after each update. No narration, no chatter.
3. **Projects are the unit of identity, not repos.** The same project can span multiple repos and tickets.

## Conversation Start Procedure

Run at the beginning of every conversation. Keep it fast and silent.

```bash
~/.agents/skills/work-tracker/scripts/wt init && ~/.agents/skills/work-tracker/scripts/wt context
```

The `init` output confirms data directory state. The `context` output is JSON:

```json
{
  "in_git_repo": true,
  "repo": "group/repo-name",
  "branch": "feature/XYZ-1234-thing",
  "is_worktree": false,
  "worktree_path": null,
  "ticket": "XYZ-1234",
  "match": { "found": true, "item_id": "XYZ-1234", "project": "...", "status": "..." },
  "total_items": 4
}
```

**Interpret the result:**

- **`match.found == true`**: Resume tracking silently.
  Output: `[work-tracker] Tracking: <project> (<ticket or "no ticket">) | N items in progress`

- **`match.found == false`, `known_repo == true`**: New branch in a known repo. Ask briefly:
  `[work-tracker] New branch '<branch>' in '<repo>'. Is this part of an existing project or something new?`
  Wait for user's answer. Then run `wt add` or `wt update` accordingly.

- **`match.found == false`, `known_repo == false`**: Unknown repo. Ask briefly:
  `[work-tracker] First time seeing repo '<repo>'. What project is this for?`

- **`in_git_repo == false`**: No git context. Do NOT ask what they're working on. Output:
  `[work-tracker] Ready | N items in progress`

**Ticket enrichment:** If `context.ticket` is non-null and ACLI is configured, fetch the ticket title
and parent epic (see `references/acli-integration.md`). Use the ticket title as the project name if
no existing project matches.

**MR discovery:** After context detection, run:

```bash
~/.agents/skills/work-tracker/scripts/wt discover
```

If `untracked` is non-empty, present each untracked MR briefly and ask if it should be added:

```
[work-tracker] Found 2 untracked MRs you authored:
  1. "Fix OAuth token refresh" (!456, opened 3 days ago) — branch: fix/oauth-refresh
  2. "Update CI pipeline" (!789, merged 2 days ago) — branch: ci-update
Add any of these to WIP? (numbers, all, or skip)
```

For confirmed items, run `wt add` with the MR metadata (title as project name, branch, repo, MR URL).
If the user says "skip" or similar, move on silently.
If discover returns an empty list or fails, skip silently — do not mention it.

## Session Opt-Out

Purely in-memory, per-conversation state — never persisted. Other tabs are unaffected.

**Pausing:** If the user says "stop/pause/skip tracking" in any phrasing, stop all tracking for this
conversation. Output: `[work-tracker] Tracking paused for this conversation.`

**Resuming:** Re-run the Conversation Start Procedure.
Output: `[work-tracker] Tracking resumed. Tracking: <Project> (<Ticket>) | N items in progress`

**Edge case:** If paused but user asks "what's in progress?" — run `wt status` and answer, but do
not resume automatic tracking unless they explicitly ask.

## When to Log

Call `wt log` and optionally `wt update` immediately after any of these:

- A git commit is made
- A PR/MR is created or merged — also capture the URL: `wt update <id> --mr "<url>"`
- A meaningful code change is completed (file edits, refactors)
- A test suite or build is run
- The user describes work they've done (Confluence edits, presentations, etc.)
- The user explicitly says "log this"

**How:**

```bash
wt log "<item-id>" "<brief description of what was done>"
wt update "<item-id>" --status "<1-2 sentence current state>"
```

To capture an MR URL when `gh pr create` or `glab mr create` runs:

```bash
wt update "<item-id>" --mr "<full URL from command output>"
```

If MR was created outside this conversation and only a number is known, construct the URL from the
repo's remote origin and the MR/PR number, then pass it to `wt update --mr`.

Output: `[work-tracker] Updated: <Project Name> (<Ticket>)`

## On-Demand Commands

Respond to these when the user asks. These are natural language — the user won't use exact phrasing.

### "What's in progress?" / "Show my WIP" / "Status?"

```bash
wt status --json
```

**IMPORTANT: Status output has TWO mandatory parts. Always do both.**

**Part 1 — WIP list.** Render as **grouped blocks**, one per item. Do NOT use a markdown table —
tables break URL detection in terminals when cells wrap across lines.

Format each item as:

```
### 1. Project Name (or Ticket · Project Name)
- **Branch:** `branch-name` (+ worktree path if exists)
- **MR:** [!number](url)  (or "None" if no MR)
- **Status:** Brief 1-line summary
```

Rules:
- **Project heading**: `### N. ` + ticket (if any) + project name, e.g. `### 1. XP-5919 · Hero Grid Banner`
- **Branch**: Show as `` `branch-name` ``. If a worktree exists, append ` — worktree: path`.
- **MR**: Render as a clickable markdown link `[!number](url)` on its own line so the terminal can detect the full URL. Use `None` if no MR exists.
- **Status**: Brief 1-line summary of current state.

**Part 2 — Pending reviews.** Always run immediately after the table:

```bash
wt reviews
```

If reviews are returned, display them after the WIP table under a `## Pending Reviews` heading
as a bullet list with title, linked URL, and age. Example:

```
## Pending Reviews

- [Fix token refresh for OAuth2 flow](https://gitlab.com/.../merge_requests/456) — opened 2 days ago
```

If no reviews or the command returns an empty list, output: `No pending reviews.`
If reviews fail silently (no glab, no config), skip the section entirely.

**Never show Part 1 without Part 2.** They are always displayed together.

### "This is done" / "Mark X as finished" / "I finished X"

1. Identify which WIP item. If ambiguous, ask.
2. Ask: "Is the whole project done, or just this ticket/task?"
3. If just a ticket: `wt update <id> --status "<updated status>"` to reflect the ticket is done.
4. If the whole project:
   ```bash
   wt done "<item-id>" "<completion summary>"
   ```
5. Output: `[work-tracker] Completed: <Project Name> (<Ticket>)`

### "I'm working on X" / "track: X"

Ask clarifying questions if needed ("Does this have a JIRA ticket?", "Is this related to an existing project?"), then:

```bash
wt add "<Project Name>" --ticket "<XYZ-1234>" --repo "<repo>" --branch "<branch>" [--worktree "<path>"] [--type "<Non-code>"]
```

For non-git tasks, use `--type` instead of `--repo`/`--branch`.

Output: `[work-tracker] New item: <Project Name> (<Ticket or "no ticket">)`

### Confluence-related work

Log it as part of an existing project if related, or create a standalone non-code entry. If ACLI
is configured and the user provides a page ID, fetch the page title (see `references/acli-integration.md`).

```bash
wt log "<item-id>" "Updated Confluence page: <title>"
```

Output: `[work-tracker] Logged: <description>`

### "Generate my annual summary" / "Generate summary for YYYY"

```bash
wt summary <year>
```

This returns structured JSON with projects grouped, date ranges, ticket numbers, repos, and all
entries. Use this data to write an executive-style narrative:

- Project name and date range (first entry to last entry)
- Related tickets
- Repos involved
- 2-3 sentence narrative: what was done and why it mattered
- "Other Contributions" section for smaller or non-code items
- Summary statistics (projects completed, repos active, tickets worked)

Write to `~/.config/work-log/archive/YYYY-annual-summary.md` and display to the user.

Keep the narrative brief and impact-focused. This is for a performance review — emphasize
outcomes over implementation details.

### "Clean up WIP" / "Cleanup" / "Prune stale items"

```bash
wt cleanup
```

### "Find my MRs" / "Discover" / "Am I missing anything?"

```bash
wt discover [--days N]
```

Finds MRs authored by the current user on GitLab that aren't tracked in WIP. Checks open MRs
and MRs merged within the last N days (default 14). Returns JSON with untracked MRs including
title, URL, branch, repo, state, and age.

Present findings as a numbered list and ask which to add:

```
Found 2 untracked MRs:
  1. "Fix OAuth token refresh" (!456, opened 3 days ago) — branch: fix/oauth-refresh
  2. "Update CI pipeline" (!789, merged 2 days ago) — branch: ci-update
Add any of these to WIP? (numbers, all, or skip)
```

For confirmed items: `wt add "<title>" --branch "<branch>" --repo "<repo>" --mr "<url>"`
For merged items the user wants to log but not track: run `wt add` then immediately `wt done`.
If empty: `[work-tracker] No untracked MRs found.`

This checks all git-tracked items against their remotes and returns JSON with classifications:
`merged`, `closed`, `active`, or `unknown`.

Present findings **one item at a time**:

```
XP-1234 | Auth Service Rewrite
  Branch: feature/oauth2 — not found on remote
  MR: .../merge_requests/123 — merged
  Remove from WIP and log as completed? (y/n)
```

Wait for the user's answer before proceeding to the next item.

- For confirmed removals: `wt done "<item-id>" "<summary>"`
- For items the user wants to keep: leave unchanged.
- Never auto-remove. Every removal requires explicit confirmation.
- If all items are active: `[work-tracker] Cleanup: all N items still active`
- When done: `[work-tracker] Cleanup: removed N items, M remain`

## Project Grouping

Projects are identified by name, established when the user first answers "what project is this?"
and persisted in `wip.md` and `config.md`.

- Multiple tickets and repos can map to the same project.
- `config.md` stores persistent aliases and settings:

```markdown
## GitLab Settings
gitlab-group: heb-engineering/projects/native-apps/ios

## Project Aliases
# XYZ-1234, XYZ-1235, auth-service/*  ->  Auth Service Rewrite
```

- When a new ticket or branch is associated with a project, add it to config.md.
- When ACLI is available, look up the parent epic to suggest groupings
  (see `references/acli-integration.md`).

## Edge Cases

**Branch switching mid-conversation:** If the user switches branches or a git command reveals a
different branch, re-run `wt context` and re-match.

**Multiple worktrees, same project:** Track under the same WIP item. Update with:
`wt update <id> --worktree "<most recently active path>"`

## ACLI Integration (Optional)

**IMPORTANT:** Before running any ACLI commands, read `references/acli-integration.md` for the
correct command syntax. Do not guess command structures.

ACLI is entirely optional — if commands fail, silently fall back to manual mode. When available:
- Auto-populated ticket titles and status from JIRA
- Parent epic lookup for automatic project grouping
- Confluence page title lookup by page ID

## wt Command Reference

```
wt init                          Ensure data directory and files exist
wt context                       Detect git context, match against WIP (JSON)
wt status [--json]               Display WIP dashboard (pretty or JSON)
wt log <id> "<msg>"              Append entry to work log, bump last_touched
wt update <id> [--status/--mr/--branch/--worktree/--repo ".."]
                                 Update WIP item fields
wt add "<name>" [--ticket/--repo/--branch/--worktree/--type/--status/--mr ".."]
                                 Add new WIP item
wt done <id> "<summary>"         Mark completed: log [COMPLETED], remove from WIP
wt cleanup                       Check remote status of git items (JSON)
wt discover [--days N]           Find authored MRs not tracked in WIP (JSON)
wt reviews                       Pending MR reviews from GitLab (JSON)
wt summary [year]                Gather work log data grouped by project (JSON)
```

Item `<id>` can be: ticket number (`XP-5210`), project slug (`peek-testing-skill`),
project name substring (`Peek`), or 1-based index (`1`).
