---
name: jira-bulk
description: >-
  Apply batch operations across multiple Jira issues — bulk close,
  bulk label, bulk reassign, bulk transition, or bulk comment. Searches
  for matching issues, presents a plan, and executes after confirmation.
  Use when asked to bulk update, close all, label multiple issues, or
  apply changes to a set of Jira issues.
license: Apache-2.0
compatibility: Requires the official Atlassian Rovo MCP server (Jira)
metadata:
  author: pshickeydev
  version: "0.1.3"
---

## Prerequisites

Derive the absolute path to `config.json` from this SKILL.md file's
`location` metadata — three directories up from this file (see
AGENTS.md § Configuration Dependency). Read it. If missing, tell the
user: "Run /configure-jira-skillset to set up your Jira defaults
first." and stop.

See AGENTS.md for shared operational best practices.

## Procedure

### Step 1 — Identify the target issues

Parse the user's input for:
- **JQL query or natural language** describing which issues to target
- **Action** to apply (close, label, reassign, transition, comment)

Build the JQL following the same patterns as `jira-search`:
- If raw JQL, use as-is
- If natural language, translate (e.g. "close all my bugs in PROJ" →
  `project = PROJ AND assignee = currentUser() AND issuetype = Bug
  AND status != Done AND status != Closed`)
- Default project: `config.defaults.project`

### Step 2 — Fetch matching issues

Call `searchJiraIssuesUsingJql` with:
- `cloudId`
- The JQL from Step 1
- `fields: ["summary", "status", "priority", "issuetype", "assignee"]`
- `maxResults: 50`
- `responseContentFormat: "markdown"`

Continue paginating with `nextPageToken` until all results are
collected. **Safety cap: 100 issues.** If more than 100 match, warn
the user and ask them to narrow the query.

### Step 3 — Present the plan

Show what will be affected:

```
Found {count} issues matching your query.

Action: {describe action}

| Key | Type | Status | Priority | Summary |
|-----|------|--------|----------|---------|
| PROJ-1 | Bug | Open | Major | Fix auth flow |
| PROJ-2 | Bug | Open | Normal | Update error messages |
...

Apply this action to all {count} issues?
```

**Always require explicit confirmation before executing.** Never
apply bulk changes without the user saying yes. Do NOT call any write
tool until the user confirms. If the user wants to adjust the query or
action, go back to the relevant step.

### Step 4 — Execute the bulk operation

Only after the user confirms, proceed. For each supported action:

**Bulk transition (e.g. close all):**
1. For the first issue, call `getTransitionsForJiraIssue` to discover
   available transitions. Find the one matching the desired target
   state.
2. Group issues by project key — transition IDs differ across
   projects. For each unique project, call `getTransitionsForJiraIssue`
   on one issue from that project to get the correct transition ID.
3. Call `transitionJiraIssue` for each issue with the project-specific
   transition ID.

**Bulk field update (e.g. label, reassign, change priority):**
1. Build the `fields` object once. If the update includes a description
   change:
   - If `config.aiDisclaimer` is true, prepend:
     `_This content was generated with AI assistance._\n\n`
   - Append the skill attribution line as the last line:
     `\n\n_Created with jira-bulk v0.1.3_`
2. Call `editJiraIssue` for each issue.

**Bulk comment:**
1. Build the comment body once.
2. If `config.aiDisclaimer` is true, prepend:
   `_This content was generated with AI assistance._\n\n`
3. Append the skill attribution line as the last line of the comment:
   `\n\n_Created with jira-bulk v0.1.3_`
4. Apply `commentVisibility` from config for each issue's project.
5. Call `addCommentToJiraIssue` for each issue.

**Execution strategy:**
- Process issues in batches of 5 parallel calls to avoid
  overwhelming the API.
- Track successes and failures separately.
- Do not stop on individual failures — continue with remaining
  issues and report failures at the end.

### Step 5 — Report

```
Bulk operation complete:
  Succeeded: {count}
  Failed:    {count}

{If failures:}
  Failed issues:
  | Key | Error |
  |-----|-------|
  | PROJ-5 | Transition not available |
```

## Gotchas

See AGENTS.md for confirmation, transition ID, comment, and
security level rules. Additional skill-specific notes:

- **Safety cap at 100 issues.** If the query matches more than 100,
  ask the user to narrow it. This prevents accidental mass updates.
- **Process in batches of 5.** Parallel calls are faster but
  hammering the API with 50+ simultaneous requests is risky. Batch
  into groups of 5.
- **Transition IDs differ across projects.** If bulk-closing issues
  from multiple projects, discover the transition ID separately for
  each project. Do NOT reuse a transition ID from one project on
  another.
- Apply `commentVisibility` per-project — extract the project key
  from each issue key and look up the config for that project.
- Some transitions require fields (e.g. resolution). If a transition
  fails due to a required field, report it and offer to retry with
  the field populated.
- **Do not clear security levels during bulk edits.** When building
  the `fields` object for `editJiraIssue`, only include fields the
  user explicitly asked to change. Omitting the `security` field
  preserves the existing value — setting it to `null` would remove it.
- Always append the skill attribution line to bulk comments and
  bulk description updates.
  See AGENTS.md for the attribution format.
