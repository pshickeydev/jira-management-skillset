---
name: jira-create
description: >-
  Create Jira issues of any type with template-driven descriptions.
  Applies configured defaults for project, priority, security level,
  labels, and assignee. Discovers available issue types at runtime.
  Use when asked to create a Jira issue, file a bug, write a story,
  create a task, or open a ticket.
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

### Step 1 — Determine project and issue type

Parse the user's input for:
- **Project key** — If specified (e.g. "create a bug in PROJ"), use it.
  Otherwise use `config.defaults.project`.
- **Issue type** — If specified (e.g. "file a bug", "create a story"),
  map it to the Jira type name. Otherwise use `config.defaults.issueType`.

Verify the project exists in `config.projects`. If not, warn the user
and offer to proceed anyway (the issue will be created without
project-specific defaults like security level).

**Discover issue types at runtime:** Call `getJiraProjectIssueTypesMetadata`
with `cloudId` and the target project key. Verify the requested issue
type exists in the returned list. If not, present available types and
ask the user to choose.

Store the `issueTypeId` (numeric string) from the response — this is
needed if custom field metadata must be looked up.

### Step 2 — Gather required fields

**Summary** is always required. If the user provided it in their request,
use it. Otherwise ask.

**Description** — Check if a template exists at
`<skillset-root>/templates/{type-lowercased}.md` (normalize: lowercase,
replace spaces with hyphens). If it does, read it and use its body sections
as scaffolding for the description. Present the template structure to the
user and ask them to fill in the sections, or accept a freeform
description.

If no template exists, accept a freeform description.

**Parent** — If creating a Sub-task, a parent issue key is required. Ask
if not provided.

### Step 3 — Apply defaults

Build the `additional_fields` object by applying configured defaults.
For each field, use the user-provided value if given, otherwise fall
back to config:

1. **Priority** — `config.defaults.priority` unless overridden.
   Pass as `{ "priority": { "name": "<value>" } }`.
2. **Labels** — Merge `config.defaults.labels`,
   `config.projects.{KEY}.defaultLabels` (if the project is in config),
   and any user-specified labels. Pass as `{ "labels": [...] }`.
3. **Components** — Use project-level `defaultComponents` from
   `config.projects.{KEY}` if available. Pass as
   `{ "components": [{ "name": "..." }, ...] }`.
4. **Security level** — Use `config.projects.{KEY}.securityLevel`.
   If not null, pass as `{ "security": { "name": "<value>" } }`.
   If null or project not in config, omit.
5. **Assignee** — If `config.assignToSelf` is true, resolve the current
   user's account ID by calling `atlassianUserInfo` and use the returned
   `account_id` as the `assignee_account_id` parameter. Cache the ID
   for the session.

### Step 4 — Confirm before creating

Present the complete issue that will be created and ask for approval:

```
Ready to create {issueTypeName} in {projectKey}:

  Summary:    {summary}
  Priority:   {priority}
  Assignee:   {assignee or "Unassigned"}
  Labels:     {labels or "None"}
  Components: {components or "None"}
  Security:   {securityLevel or "None"}
  Parent:     {parent or "None"}

  Description:
  {first ~200 chars of description}...

Create this issue?
```

Wait for explicit approval. If the user wants changes, go back to the
relevant step. Do NOT call any write tool until the user confirms.

### Step 5 — Create the issue

Only after the user confirms, call `createJiraIssue` with:
- `cloudId` from config
- `projectKey`
- `issueTypeName` (the display name, e.g. "Bug", not the numeric ID)
- `summary`
- `description` with `contentFormat: "markdown"` — append the skill
  attribution line as the last line of the description body:
  `\n\n_Created with jira-create v0.1.2_`
- `assignee_account_id` (if auto-assign)
- `parent` (if Sub-task)
- `additional_fields` assembled in Step 3

### Step 6 — Report

Print confirmation:
```
Created {issueTypeName}: {KEY}-{number}
  Summary:  {summary}
  Priority: {priority}
  Assignee: {assignee or "Unassigned"}
  Security: {securityLevel or "None"}
  Link:     https://{cloudId}/browse/{KEY}-{number}
```

## Gotchas

- Always discover issue types at runtime via
  `getJiraProjectIssueTypesMetadata`. Do not rely on the `issueTypes`
  list in `config.json` — it may be stale if project admins change
  the scheme.
- `createJiraIssue` uses `issueTypeName` (the display name string),
  not the numeric `issueTypeId`.
- Template filenames are normalized: lowercase, spaces to hyphens.
  "Sub-task" → `sub-task.md`, "Change Request" → `change-request.md`.
- If the user specifies a project not in `config.projects`, the issue
  can still be created — just skip project-specific defaults (security
  level, components, labels) and warn the user.
- The AI disclaimer prefix is only for comments, not issue descriptions.
  The skill attribution line is separate — it is always appended to the
  description. See AGENTS.md for the attribution format.
- Use `contentFormat: "markdown"` for descriptions so the user can
  write natural markdown without constructing ADF.
