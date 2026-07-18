# Field Mappings Reference

Default mappings used across the Jira management skills. The
`/configure-jira-skillset` skill populates project-specific custom field
IDs in `config.json`.

## Priority Mapping

**Priority names are instance-specific.** The table below shows common
names, but your instance may differ (e.g. "Normal" instead of "Medium",
"Undefined" instead of "Trivial"). Always use the exact names stored in
`config.json`, which were discovered from the API by `/configure-jira-skillset`.

Use with `additional_fields: { "priority": { "name": "<value>" } }`.

| Common Name | Typical Severity | Notes |
|-------------|-----------------|-------|
| Blocker | Highest | Prevents all progress |
| Critical | Highest | System down or data loss |
| Major | High | Significant impact, workaround may exist |
| Normal / Medium | Medium | Default for most work items |
| Minor | Low | Low impact, deferrable |
| Undefined / Trivial | Lowest | Cosmetic or informational |

The `/configure-jira-skillset` skill discovers available priorities from
the project's field metadata and stores the default in `config.json`.

## Standard Fields

Fields available on most issue types via `createJiraIssue`:

| Field | Parameter | Notes |
|-------|-----------|-------|
| Summary | `summary` (top-level) | Required for all types |
| Description | `description` (top-level) | Use with `contentFormat: "markdown"` |
| Issue Type | `issueTypeName` (top-level) | e.g. "Bug", "Story" |
| Project | `projectKey` (top-level) | e.g. "PROJ" |
| Assignee | `assignee_account_id` (top-level) | Account ID from `lookupJiraAccountId` |
| Parent | `parent` (top-level) | Issue key, for subtasks/child issues |
| Priority | `additional_fields.priority` | `{ "name": "Medium" }` |
| Labels | `additional_fields.labels` | `["label1", "label2"]` |
| Components | `additional_fields.components` | `[{ "name": "Backend" }]` |
| Security Level | `additional_fields.security` | `{ "name": "Level Name" }` |
| Fix Version | `additional_fields.fixVersions` | `[{ "name": "v1.2" }]` |

## Common Custom Fields

**Custom field IDs are instance-specific and MUST NOT be assumed.**
The `/configure-jira-skillset` skill discovers the actual IDs and stores
them in `config.json` under `projects.{KEY}.customFields`. Always read
IDs from config, never hardcode them.

| Common Name | Type | Notes |
|-------------|------|-------|
| Story Points | number | Often customfield_100XX — varies by instance |
| Sprint | array | Greenhopper sprint field |
| Epic Link | string (issue key) | Links issue to an Epic |
| Severity | select | Critical/Important/Moderate/Low/Informational |
| Start date | date (YYYY-MM-DD) | |
| Target start | date (YYYY-MM-DD) | Advanced Roadmaps field |
| Target end | date (YYYY-MM-DD) | Advanced Roadmaps field |

## Security Levels and Comment Visibility

Issue security level and comment visibility are **independent controls**.
Jira does not support inheriting comment visibility from the issue's
security level.

- **Issue security:** Set via `additional_fields: { "security": { "name": "..." } }`.
  Applied when creating or editing issues. Value stored per-project in
  `config.json` as `projects.{KEY}.securityLevel`.
- **Comment visibility:** Set via the `commentVisibility` parameter on
  `addCommentToJiraIssue`. Value stored per-project in `config.json` as
  `projects.{KEY}.commentVisibility`. Accepts `{ type, value }` or `null`.
- If either value is `null` in config, omit the parameter (no restriction).
