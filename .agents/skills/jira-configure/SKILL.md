---
name: jira-configure
description: >-
  Interactive setup for the Jira management skillset. Discovers accessible
  Jira sites, projects, issue types, security levels, and custom fields.
  Persists user preferences to config.json so other skills can apply them
  automatically. Supports first-time setup and incremental reconfiguration.
  Use when the user says /configure-jira-skillset, "set up Jira",
  "configure Jira defaults", or "add a project to Jira config".
license: Apache-2.0
compatibility: Requires the official Atlassian Rovo MCP server (Jira)
metadata:
  author: pshickeydev
  version: "0.1.1"
---

## Overview

This skill walks the user through an interactive configuration wizard.
It is the only skill in the set that performs Jira site discovery — all
other skills depend on the `config.json` this skill produces.

See AGENTS.md (at the skillset root) for shared operational best practices.

## Procedure — First-Time Setup

Run this flow when `config.json` does not exist at the skillset root.
Derive the absolute path to `config.json` from this SKILL.md file's
`location` metadata: this file lives at
`<skillset-root>/.agents/skills/jira-configure/SKILL.md`, so
`config.json` is at `<skillset-root>/config.json` — three directories up.

### Step 1 — Discover Jira sites

Call `getAccessibleAtlassianResources` (no parameters needed).

- If exactly one site is returned, confirm it with the user:
  "I found your Jira site: **{name}** ({url}). Use this?"
- If multiple sites are returned, present them as options and ask the
  user to choose.
- If none are returned, tell the user to check their Rovo MCP
  authentication and stop.

Store the chosen site's `url` (hostname, e.g. `yoursite.atlassian.net`)
as the `cloudId`. The Rovo MCP accepts a site hostname and resolves it
internally.

### Step 2 — Select default project

Large organizations can have hundreds or thousands of projects. Do NOT
call `getVisibleJiraProjects` without a `searchString` — the unfiltered
list can be enormous and wastes context.

Instead, ask the user to type one or more project keys or name fragments.
Then call `getVisibleJiraProjects` with the `searchString` parameter for
each. Present the matched results (key + name) and let the user confirm.

Allow the user to configure multiple projects in one session by asking
"Do you want to add another project?" after each one.

The first project confirmed becomes `defaults.project`.

### Step 3 — Discover issue types

Call `getJiraProjectIssueTypesMetadata` with `cloudId` and
`projectIdOrKey` set to the chosen project key.

Present all discovered issue types as a checklist. Let the user confirm
which types they plan to use (all are stored, but this helps prioritize
template generation).

Store the full list in `projects.{KEY}.issueTypes`.

Ask the user which issue type should be the default. Store as
`defaults.issueType`.

### Step 4 — Discover fields and custom fields

For each selected issue type (or at minimum the default type), call
`getJiraIssueTypeMetaWithFields` with `cloudId`, `projectIdOrKey`, the
`issueTypeId` (from Step 3), and `requiredFieldsOnly: false` to get all
available fields.

**Important:** The field set varies by issue type within the same project.
Some fields (including `security`) may appear on one issue type but not
another. If the security field is not found on the default type, check at
least one additional type (e.g. Story, Bug) before concluding the project
has no security levels.

**Important:** Set `maxResults` high enough to get all fields in one call.
Projects can have 50+ fields. Use `maxResults: 100`. If `total` exceeds
`maxResults`, paginate to retrieve the rest — fields like `security` and
`priority` can appear at any offset.

Scan the returned fields for:

1. **Security level** — Look for a field with `fieldId` equal to
   `security`. If present, extract `allowedValues` to get the list of
   available security levels.
2. **Priority** — Look for the `priority` field and extract
   `allowedValues` to get available priority names. Do NOT assume
   standard names like "Medium" or "Trivial" — instances may use
   different names (e.g. "Normal", "Undefined"). Always use the exact
   names returned by the API.
3. **Components** — Look for `components` and extract `allowedValues`.
4. **Custom fields** — Identify fields whose `fieldId` starts with
   `customfield_`. Present notable ones (story points, sprint, epic link,
   severity) and let the user confirm which to track. Custom field IDs
   are instance-specific — do not assume common IDs like
   `customfield_10016` for story points.

Store custom field mappings in `projects.{KEY}.customFields` as
`{ "Display Name": "customfield_XXXXX" }`.

### Step 5 — Set security defaults

**Issue security level:**

If security levels were found in Step 4, present them as options plus a
"No restriction" choice. Let the user choose a default.

Store in `projects.{KEY}.securityLevel` (string name, or `null` for no
restriction).

**Comment visibility:**

Explain to the user that comment visibility and issue security level are
**independent controls** in Jira. There is no "inherit from issue" option
for comments — the Rovo MCP `commentVisibility` parameter only accepts
a role, a group, or null (unrestricted). If the user asks to "match the
issue security level," explain this limitation and ask them to choose one
of the supported options.

Ask the user how agent-posted comments should be restricted:

- **By project role** — Ask which role (e.g. "Developers", "Administrators").
  Store as `{ "type": "role", "value": "<role>" }`.
- **By group** — Ask for the group name.
  Store as `{ "type": "group", "value": "<group>" }`.
- **No restriction** — Comments visible to anyone who can see the issue.
  Store as `null`.

Store in `projects.{KEY}.commentVisibility`.

### Step 6 — Set remaining defaults

Walk through these prompts:

1. **Default priority** — Present the priorities discovered in Step 4.
   Store as `defaults.priority`.
2. **Default labels** — Ask for comma-separated labels (or empty for
   none). Store as `defaults.labels` (array of strings).
3. **Default components** — Present discovered components from Step 4,
   or allow free entry. Store as `defaults.components`.
   Also store project-level defaults in `projects.{KEY}.defaultLabels`
   and `projects.{KEY}.defaultComponents`.
4. **Auto-assign to self** — Ask yes/no. Store as `assignToSelf`.
5. **AI disclaimer on comments** — Explain: "When skills post comments
   to Jira, should they include a notice that the content was generated
   with AI assistance?" Ask yes/no. Store as `aiDisclaimer`.

### Step 7 — Generate templates

Only generate templates for common issue types the user is likely to
create directly. Skip administrative types (Release tracker, Release
Milestone, Outcome, Vulnerability, Weakness) unless the user explicitly
asks for them — these are typically created by automated systems or
specialized workflows.

For each applicable issue type, generate a template file at
`<skillset-root>/templates/{type-lowercased}.md`.
Normalize the filename: replace spaces with hyphens, lowercase all
characters (e.g. "Sub-task" → `sub-task.md`, "Change Request" →
`change-request.md`).

Use the following structure:

```markdown
---
type: {IssueTypeName}
required_fields:
  - summary
  {- any additional required fields from Step 4}
optional_fields:
  {- discovered optional fields relevant to this type}
---

## Summary
{{summary}}

## Description
{{description}}
```

For known types, include richer scaffolding:

**Bug template** — Add sections for "Steps to Reproduce", "Expected
Behavior", "Actual Behavior", "Environment".

**Story template** — Add sections for "User Story" (As a / I want / So
that), "Acceptance Criteria".

**Epic template** — Add sections for "Goal", "Scope", "Success Criteria".

**Task template** — Add sections for "Description", "Acceptance
Criteria".

**Spike template** — Add sections for "Research Question", "Context",
"Timebox", "Expected Output".

**Sub-task template** — Add "Parent" to required fields. Add sections
for "Description", "Acceptance Criteria".

For any type not listed above (custom types), generate a generic
template with "Summary" and "Description" sections only.

Do NOT overwrite existing templates without asking the user first.

### Step 8 — Write config.json

Assemble the complete configuration object and write it to
`config.json` at the skillset root (the absolute path derived from
this SKILL.md's `location` — three directories up).

Set `configured_at` to the current ISO 8601 timestamp.

### Step 9 — Report

Print a summary:

```
Configuration complete!

  Site:            {cloudId}
  Default project: {defaults.project}
  Default type:    {defaults.issueType}
  Default priority:{defaults.priority}
  Security level:  {securityLevel or "None"}
  Comment access:  {commentVisibility description or "Unrestricted"}
  AI disclaimer:   {yes/no}
  Issue types:     {comma-separated list}
  Templates:       {count} generated in templates/

  To reconfigure, run /configure-jira-skillset again.
  To add another project, run /configure-jira-skillset and select "Add a project".
```

---

## Procedure — Reconfigure (config.json exists)

When `config.json` already exists at the skillset root, read it and
present the current configuration summary (same format as Step 9 above).

Then prompt the user with options:

- **Update global defaults** — Re-run Steps 6 items 1-5 only (priority,
  labels, components, assignee, disclaimer). Keep site and project
  unchanged.
- **Add / reconfigure a project** — Ask for a project key or let the
  user search. Then run Steps 3-7 for that project only, merging the
  result into the existing `projects` object.
- **Change default project** — Present configured projects as options,
  let the user switch.
- **Regenerate templates** — Re-run Step 7 for all configured projects.
  Ask before overwriting existing templates.
- **Full reconfigure** — Re-run the entire first-time flow from Step 1.
  Warn that this overwrites all settings.
- **View current config** — Print the full configuration summary and
  stop.

After any reconfigure action, write the updated `config.json` and
confirm the changes.

---

## Gotchas

- The Rovo MCP authenticates via OAuth 2.1 only. If any tool call fails
  with an auth error, tell the user to complete sign-in in their MCP
  client. Never ask for API tokens, passwords, or base URLs.
- `getAccessibleAtlassianResources` is the only reliable way to discover
  accessible sites. Do not assume or hardcode site URLs.
- `getJiraIssueTypeMetaWithFields` requires `issueTypeId` (a numeric
  string), not the issue type name. Get IDs from
  `getJiraProjectIssueTypesMetadata` first.
- **Field availability varies by issue type.** The `security` field may
  appear on Story but not Task within the same project. If security
  levels are not found on the default issue type, check at least one
  more type before concluding the project has no security scheme.
- **Pagination can hide fields.** Projects can have 50+ fields. Always
  set `maxResults: 100` and check whether `total` exceeds the returned
  count. If it does, paginate to fetch the remaining fields — critical
  fields like `security` and `priority` can appear at any offset.
- **Priority names are instance-specific.** Do not assume "Medium" or
  "Trivial" exist. Common variants include "Normal" (instead of
  "Medium") and "Undefined" (instead of "Trivial"). Always use the
  exact names returned by `getJiraIssueTypeMetaWithFields`.
- **Custom field IDs are instance-specific.** Story Points is commonly
  cited as `customfield_10016` but can be any ID (e.g. `customfield_10028`).
  Always discover IDs from the API, never hardcode them.
- **Comment visibility and issue security are independent.** Jira does
  not support "inherit security level" for comments. The
  `commentVisibility` parameter only accepts `{ type, value }` or null.
  If a user asks to match the issue security level, explain this
  limitation and offer the supported options.
- **Large project lists.** Organizations can have 1000+ projects. Never
  call `getVisibleJiraProjects` without `searchString` — always ask the
  user for a project key or name fragment first.
- Template generation should be idempotent — if re-run, it should ask
  before overwriting existing templates.
- `config.json` paths in skills use absolute paths derived from the
  skill's `location` metadata. From any SKILL.md at
  `<skillset-root>/.agents/skills/<name>/SKILL.md`, the config is at
  `<skillset-root>/config.json` — three directories up from the SKILL.md.
- When merging project config during reconfiguration, preserve existing
  project entries that are not being modified.
