---
name: jira-transition
description: >-
  Transition Jira issues through workflow states. Discovers available
  transitions at runtime — never hardcodes transition IDs. Can
  optionally add a comment when transitioning. Use when asked to
  close, reopen, start, move, or change the status of a Jira issue.
license: Apache-2.0
compatibility: Requires the official Atlassian Rovo MCP server (Jira)
metadata:
  author: pshickeydev
  version: "0.1.2"
---

## Prerequisites

Derive the absolute path to `config.json` from this SKILL.md file's
`location` metadata — three directories up from this file (see
AGENTS.md § Configuration Dependency). Read it. If missing, tell the
user: "Run /configure-jira-skillset to set up your Jira defaults
first." and stop.

See AGENTS.md for shared operational best practices.

## Procedure

### Step 1 — Identify the issue

Parse the issue key from the user's input. If not provided, ask.

### Step 2 — Fetch current state and available transitions

Make both calls in parallel:

1. `getJiraIssue` with `cloudId`, `issueIdOrKey`, and
   `fields: ["summary", "status", "assignee"]`,
   `responseContentFormat: "markdown"`
2. `getTransitionsForJiraIssue` with `cloudId`, `issueIdOrKey`

Present the current status and available transitions:
```
{KEY}: {summary}
  Current status: {status.name}

  Available transitions:
    1. {transition.name} → {transition.to.name}
    2. {transition.name} → {transition.to.name}
    ...
```

### Step 3 — Select transition

If the user's request implies a specific transition (e.g. "close
PROJ-123"), match it against the available transition names:
- "close" → look for "Close", "Closed", "Done", "Resolve"
- "start" → look for "In Progress", "Start Progress"
- "reopen" → look for "Reopen", "Re-open"
- "done" → look for "Done", "Closed", "Resolve"

Use case-insensitive substring matching against transition names or
target status names. If exactly one match is found, use it. If
multiple match, present them as options. If none match, show all
available transitions and ask.

**Critical:** Use the `transition.id` from the
`getTransitionsForJiraIssue` response. Transition IDs are per-project
and per-workflow — never hardcode or reuse IDs across issues.

### Step 4 — Optional comment

Ask the user if they want to add a comment with the transition.

If yes, compose the comment text. Use `addCommentToJiraIssue` as a
**separate call** — do NOT use the `comment` parameter on
`transitionJiraIssue` (it causes Atlassian Document Format errors).

If adding a comment:
- Apply `commentVisibility` from config
- Apply AI disclaimer prefix if `config.aiDisclaimer` is true
- Append the skill attribution line as the last line of the comment:
  `\n\n_Created with jira-transition v0.1.2_`

### Step 5 — Confirm before executing

Present the complete action and ask for approval:

```
Transition {KEY}:
  Current status: {current status}
  New status:     {transition.to.name}
  Transition:     {transition.name} (ID: {transition.id})
  {If comment:}
  Comment:        "{comment text}"
  Visibility:     {commentVisibility description or "Unrestricted"}

Proceed with this transition?
```

Wait for explicit approval. Do NOT call any write tool until the user
confirms.

### Step 6 — Execute the transition

Only after the user confirms, call `transitionJiraIssue` with:
- `cloudId`
- `issueIdOrKey`
- `transition: { "id": "<transition_id>" }`

If the user also requested a comment, make the comment call in
parallel with the transition call for efficiency.

### Step 7 — Report

```
Transitioned {KEY}:
  {old status} → {new status}
  {comment added if applicable}
```

## Gotchas

See AGENTS.md for transition ID, comment, and confirmation
rules. Additional skill-specific notes:

- The available transitions depend on the issue's current status and
  the project's workflow. An issue in "Done" may not have a "Close"
  transition — it might only have "Reopen".
- If a transition requires fields (e.g. resolution), the
  `getTransitionsForJiraIssue` response indicates this. Pass required
  fields in the `fields` parameter of `transitionJiraIssue`.
- When closing and commenting in parallel, both calls are independent
  and can fail independently. Report each result separately.
- Always append the skill attribution line to transition comments.
  See AGENTS.md for the attribution format.
