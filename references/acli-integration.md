# ACLI Integration (Optional)

This document covers how the work-tracker skill uses the Atlassian CLI (`acli`) for JIRA and
Confluence enrichment. Everything here is optional -- the skill works fully without ACLI.

## Prerequisites

- `acli` installed and on PATH
- Authenticated: `acli auth`

## Checking ACLI Availability

At conversation start, if the skill wants to use ACLI features, first verify it works:

```bash
acli jira workitem view TEST-1 2>&1
```

If this fails (not installed, not authenticated, network error), silently disable ACLI features
for the session. Do not warn or nag the user.

## JIRA Commands

### Fetch Ticket Details

When a ticket number is detected (from a branch name like `feature/XYZ-1234-description` or
from the user mentioning it), fetch the title and status:

```bash
acli jira workitem view XYZ-1234 --fields summary,status,issuetype --json
```

Use the `summary` field to auto-populate the WIP item name instead of asking the user.

### Fetch Parent Epic for Grouping

To discover project groupings automatically:

```bash
acli jira workitem view XYZ-1234 --fields summary,parent --json
```

If the ticket has a parent epic, use the epic's summary as a suggested project name:

```
[work-tracker] XYZ-1234 "Implement token refresh" belongs to epic XYZ-1200 "Auth Service Rewrite".
               Tracking under: Auth Service Rewrite
```

If the epic name matches an existing project in `wip.md`, auto-associate. If it's new, ask
the user to confirm.

### Search Recently Worked Tickets

To find tickets you've been working on (useful for catching up after time away):

```bash
acli jira workitem search --jql "assignee = currentUser() AND updated >= -7d" --fields key,summary,status --json
```

Only use this when the user asks for a status overview or when trying to match context.
Do not run this automatically on every conversation start.

### Check Ticket Status

To detect if a ticket has been closed (which might mean the WIP item is done):

```bash
acli jira workitem view XYZ-1234 --fields status --json
```

If status is "Done", "Closed", or equivalent, mention it when displaying WIP status:

```
XYZ-1234 | Auth Service Rewrite
  Branch: feature/oauth2-migration
  Status: Migrated token refresh logic. Integration tests still pending.
  JIRA Status: Done (ticket was closed -- mark as finished?)
```

## Confluence Commands

### Fetch Page Title by ID

When the user mentions a Confluence page by ID:

```bash
acli confluence page view <page-id>
```

Use the page title to enrich the log entry:

```markdown
### Platform Architecture Overview (Confluence, page 12345678)
- Updated with new auth service diagrams
```

### Limitations

The Confluence CLI currently supports viewing pages by ID but does not have a "recently modified
by me" search. Confluence work is primarily tracked through natural language -- the user tells
the skill what they updated. ACLI just helps look up page titles when an ID is provided.

## Fallback Behavior

If any ACLI command fails during a session:

1. Log the work with whatever information is available (ticket number without title, etc.)
2. Do not retry the failed command
3. Do not inform the user that ACLI failed unless they explicitly asked for JIRA/Confluence data
4. Continue operating in manual mode for the rest of the session

The skill should never block or degrade because of ACLI issues.

## config.md ACLI Settings

When ACLI is available, the user may optionally add settings to `~/.config/work-log/config.md`:

```markdown
## JIRA
# jira_project: XP
# This helps the skill recognize ticket patterns like XYZ-1234 in branch names
```

This is not required. Without it, the skill still extracts ticket numbers from branch names
using common patterns (ALPHA-DIGITS, e.g., `XYZ-1234`, `AUTH-567`).
