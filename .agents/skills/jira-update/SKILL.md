---
name: jira-update
description: >-
  Update existing Jira issues — edit fields, change priority, add or
  update comments, modify labels, reassign, or update descriptions.
  Fetches current issue state at runtime before making changes.
  Use when asked to update, edit, modify, comment on, or change a
  Jira issue.
license: Apache-2.0
compatibility: Requires the official Atlassian Rovo MCP server (Jira)
metadata:
  author: pshickeydev
  version: "0.1.0"
---

## Prerequisites

Read `../../config.json` relative to this SKILL.md. If missing, tell
the user: "Run /configure-jira-skillset to set up your Jira defaults
first." and stop.

See `../../AGENTS.md` for shared operational best practices.

## Procedure

### Step 1 — Identify the issue

Parse the issue key from the user's input (e.g. "update PROJ-123",
"add a comment to ACME-456"). The key must match the pattern
`^[A-Z]+-\d+$`.

If no key is provided, ask the user for it.

### Step 2 — Fetch current state

Call `getJiraIssue` with `cloudId`, `issueIdOrKey`, and
`responseContentFormat: "markdown"` to get the current issue state.

Present a concise summary to the user:
```
{KEY}: {summary}
  Type:     {issuetype.name}
  Status:   {status.name}
  Priority: {priority.name}
  Assignee: {assignee.displayName or "Unassigned"}
  Labels:   {labels or "None"}
```

### Step 3 — Determine the update action

If the user's request is clear (e.g. "change priority to Major"),
proceed directly. Otherwise, prompt with options:

- **Edit fields** — Change priority, labels, components, assignee,
  summary, or other fields
- **Update description** — Replace or append to the description
- **Add a comment** — Post a new comment to the issue
- **Update a comment** — Modify an existing comment

Allow multiple actions in sequence.

### Step 4 — Confirm before writing

Present the exact changes that will be made and ask for approval:

**For field edits**, show:
```
Update {KEY}:
  {field}: {current value} → {new value}
  {field}: {current value} → {new value}
  ...

Apply these changes?
```

**For comments**, show:
```
Add comment to {KEY}:
  Visibility: {commentVisibility description or "Unrestricted"}
  Body:
  {full comment text}

Post this comment?
```

Wait for explicit approval. Do NOT call any write tool until the user
confirms. If the user wants changes, revise and re-confirm.

### Step 5 — Execute the update

Only after the user confirms, proceed with the write calls.

**For field edits**, call `editJiraIssue` with:
- `cloudId`, `issueIdOrKey`
- `fields` object containing the changed fields
- `contentFormat: "markdown"` if description is being changed
- `responseContentFormat: "markdown"`

Common field patterns:
- Priority: `{ "priority": { "name": "Major" } }`
- Labels (replace): `{ "labels": ["label1", "label2"] }`
- Assignee: `{ "assignee": { "accountId": "..." } }`
- Assignee (clear): `{ "assignee": null }`
- Summary: `{ "summary": "New summary text" }`
- Description: `{ "description": "New description" }`
- Security level: `{ "security": { "name": "..." } }`

**For adding a comment**, call `addCommentToJiraIssue` with:
- `cloudId`, `issueIdOrKey`
- `commentBody` with `contentFormat: "markdown"`
- `commentVisibility` from `config.projects.{KEY}.commentVisibility`
  (omit if null)
- If `config.aiDisclaimer` is true, prepend:
  `_This comment was generated with AI assistance._\n\n`

**For updating a comment**, call `addCommentToJiraIssue` with the same
parameters plus `commentId` set to the existing comment's ID. To find
comment IDs, fetch the issue with `fields: ["comment"]`.

### Step 6 — Report

Print what changed:
```
Updated {KEY}:
  {field}: {old value} → {new value}
```

Or for comments:
```
Added comment to {KEY}:
  "{first 80 chars of comment}..."
```

## Gotchas

See `../../AGENTS.md` for comment, confirmation, and content format
rules. Additional skill-specific notes:

- Always fetch the current issue state before editing. This prevents
  stale assumptions and lets you show the user what they're changing.
- To clear a field, set its value to `null` in the `fields` object.
- Apply the AI disclaimer prefix only to comments, not to descriptions
  or field edits.
- The project key can be extracted from the issue key (everything
  before the hyphen) to look up project-specific config.
- `editJiraIssue` returns the updated issue. Use
  `responseContentFormat: "markdown"` for readable output.
