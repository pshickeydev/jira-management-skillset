---
name: jira-search
description: >-
  Search for Jira issues using JQL or natural language queries. Builds
  JQL from user intent, paginates results, and presents formatted
  tables. Use when asked to find, search, list, or query Jira issues,
  or when asked "what are my open issues" or "show bugs in PROJ".
license: Apache-2.0
compatibility: Requires the official Atlassian Rovo MCP server (Jira)
metadata:
  author: pshickeydev
  version: "0.1.1"
---

## Prerequisites

Derive the absolute path to `config.json` from this SKILL.md file's
`location` metadata — three directories up from this file (see
AGENTS.md § Configuration Dependency). Read it. If missing, tell the
user: "Run /configure-jira-skillset to set up your Jira defaults
first." and stop.

See AGENTS.md for shared operational best practices.

## Procedure

### Step 1 — Build the JQL query

Parse the user's input and construct JQL:

**If the input contains JQL operators** (`=`, `!=`, `AND`, `OR`,
`ORDER BY`, `IN`, `NOT IN`, `~`, `IS`, `IS NOT`), treat it as raw JQL
and use it as-is.

**If natural language**, translate to JQL using these patterns:

| User says | JQL |
|-----------|-----|
| "my open issues" | `assignee = currentUser() AND status != Done AND status != Closed ORDER BY updated DESC` |
| "my open bugs" | `assignee = currentUser() AND issuetype = Bug AND status != Done AND status != Closed ORDER BY priority ASC, updated DESC` |
| "bugs in PROJ" | `project = PROJ AND issuetype = Bug AND status != Done AND status != Closed ORDER BY priority ASC, created DESC` |
| "issues updated this week" | `assignee = currentUser() AND updated >= startOfWeek() ORDER BY updated DESC` |
| "PROJ backlog" | `project = PROJ AND status = "To Do" ORDER BY priority ASC, created ASC` |
| "unassigned tasks in PROJ" | `project = PROJ AND issuetype = Task AND assignee IS EMPTY AND status != Done ORDER BY created DESC` |

If no project is specified, use `config.defaults.project`.

If the query is ambiguous, show the generated JQL and ask the user to
confirm or refine before executing.

### Step 2 — Execute the search

Call `searchJiraIssuesUsingJql` with:
- `cloudId` from config
- `jql` from Step 1
- `fields: ["summary", "status", "priority", "issuetype", "assignee", "updated"]`
- `maxResults: 25` (keep initial result set manageable)
- `responseContentFormat: "markdown"`

### Step 3 — Present results

Format as a table:

```
Found {total} issues:

| Key | Type | Status | Priority | Assignee | Summary |
|-----|------|--------|----------|----------|---------|
| PROJ-123 | Bug | Open | Major | @user | Fix the thing |
| PROJ-456 | Task | In Progress | Normal | Unassigned | Review config |
```

If there are more results than returned (check `nextPageToken`), offer:
"Showing {count} of {total}. Load more?"

### Step 4 — Follow-up actions

After presenting results, offer relevant actions:

- **Load more** — Paginate using `nextPageToken`
- **Refine query** — Adjust the JQL and re-run
- **Open an issue** — Fetch full details of a specific result
- **Bulk action** — Suggest using `jira-bulk` for batch operations

## Gotchas

- `searchJiraIssuesUsingJql` paginates via `nextPageToken`, NOT
  `startAt`/`limit`. Pass the token from the previous response to
  get the next page.
- `maxResults` caps at 100 per call. Start with 25 to keep context
  manageable, then paginate if the user wants more.
- JQL field names are case-insensitive but values may not be. Status
  names must match exactly (e.g. "In Progress" not "in progress").
- `currentUser()` is a JQL function — it does not need a value, and
  the Rovo MCP resolves it to the authenticated user.
- When translating natural language to JQL, prefer `ORDER BY updated
  DESC` for recency or `ORDER BY priority ASC, created DESC` for
  triage views.
- If the JQL is invalid, the API returns an error message with the
  position of the syntax error. Present this to the user so they can
  fix it.
