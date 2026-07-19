# Jira Management Skills — Operational Guide

Shared best practices for all skills in this skillset. Individual SKILL.md
files reference this document with "see AGENTS.md."

**Maintainer note:** When making changes to this file that affect
user-facing behavior (new skills, changed defaults, new configuration
options, or updated best practices), reflect those changes in `README.md`
at the skillset root so that human readers stay informed.

## Confirmation Before Write

- **Every skill must show the user exactly what will be sent to Jira and
  receive explicit approval before making any write call.** This applies
  to `createJiraIssue`, `editJiraIssue`, `transitionJiraIssue`,
  `addCommentToJiraIssue`, `addWorklogToJiraIssue`, and `createIssueLink`.
- The confirmation must include all material details: issue key, field
  values being set or changed, comment text, transition target, link
  direction — whatever is relevant to the operation.
- Never assume the user's intent is sufficient approval. Even if the user
  said "close PROJ-123", confirm the exact transition and target status
  before executing.
- For bulk operations, show the full list of affected issues and the
  action before proceeding.
- Read-only operations (search, fetch) do not require confirmation.

## Authentication

- The Rovo MCP uses OAuth 2.1 exclusively. Never ask users for API tokens,
  passwords, or base URLs. If an auth error occurs, instruct the user to
  complete sign-in in their MCP client.
- Use the `cloudId` stored in `config.json`. Only the `jira-configure` skill
  performs initial discovery via `getAccessibleAtlassianResources`.

## Configuration Dependency

- Every skill except `jira-configure` requires `config.json` in the skillset
  root. If missing, tell the user: "Run /configure-jira-skillset to set up
  your Jira defaults first." and stop.
- Read `config.json` relative to the SKILL.md location:
  `../../config.json` from any `.agents/skills/<name>/SKILL.md`.

## Data Handling

- Do not include personally identifiable information (PII) or sensitive data
  in issue descriptions, comments, or any content sent to AI-backed tools.
- Review all AI-generated content before submitting it to Jira.
- When posting comments via an agent, prepend an "AI-generated" notice to
  the comment body if the `aiDisclaimer` flag in `config.json` is `true`.
  Example prefix: `_This comment was generated with AI assistance._\n\n`

## Security Levels

- Always apply the project's configured `securityLevel` from `config.json`
  when creating or editing issues. Pass it as
  `additional_fields: { "security": { "name": "<level>" } }`.
- Always apply the project's configured `commentVisibility` from
  `config.json` when adding comments or worklogs.
- If either value is `null`, omit the parameter (no restriction).
- If the target project is not in `config.json`, warn the user and offer
  to run `/configure-jira-skillset` for that project before proceeding.

## Auto-Assignment

- When `config.json` has `assignToSelf: true`, skills that create issues
  should set the assignee to the current user.
- To get the current user's account ID, call `atlassianUserInfo` and use
  the returned `account_id` as the `assignee_account_id` parameter on
  `createJiraIssue`.
- Cache the account ID within a session — it does not change between calls.

## Rovo MCP Scope

- Rovo searches Jira, Confluence, and JSM based on the authenticated user's
  permissions. Third-party connectors may not be available at your
  organization — do not assume access to external data sources.
- Set `contentFormat` to `"markdown"` on descriptions and comments for
  readability.

## Tool-Specific Patterns

- **Transition IDs are per-project.** Always call `getTransitionsForJiraIssue`
  before transitioning — never hardcode transition IDs.
- **Comments:** Use `addCommentToJiraIssue`, not the `comment` parameter on
  `transitionJiraIssue` (the latter causes Atlassian Document Format errors).
- **Pagination:** `searchJiraIssuesUsingJql` paginates via `nextPageToken`,
  not `startAt`/`limit`. `maxResults` caps at 100 per call.
- **Custom fields:** Use `additional_fields` on `createJiraIssue` for
  priority, labels, components, and custom fields — they do not have
  dedicated top-level parameters.
- **Issue links:** `createIssueLink` uses directional semantics —
  `inwardIssue` is the blocker, `outwardIssue` is the blocked issue.
  Call `getIssueLinkTypes` first to discover available link types.

## Versioning

Skills use [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`).
The version is stored in each skill's YAML frontmatter at
`metadata.version`.

### When to bump each field

- **PATCH** — Any change that does not add new features or break
  compatibility. Examples: bug fixes, wording improvements, gotcha
  additions, adding attribution lines, adjusting defaults, refining
  instructions.
- **MINOR** — Adding a new capability or backward-compatible behavior.
  Examples: supporting a new action (e.g. adding worklog support to
  `jira-update`), adding a new optional config field, supporting a new
  issue type pattern.
- **MAJOR** — Any change that breaks an existing user's workflow or
  config. Examples: removing or renaming a supported action, changing
  the meaning of an existing config field, adding a required config
  field that existing `config.json` files do not have.

### Scope of a bump

- When a change to a skill's own `SKILL.md` is made, bump that skill's
  version according to the criteria above.
- When a change to `AGENTS.md` introduces a new shared requirement that
  affects skill behavior (e.g. the Skill Attribution rule), bump every
  skill that must comply with the new requirement. The bump level is
  determined by the nature of the change — a new shared requirement
  that only adds to output (like attribution) is a PATCH; a new shared
  requirement that changes how skills interact with Jira would be MINOR
  or MAJOR depending on impact.
- Read-only skills are still versioned. Bump them when their procedure,
  output format, or behavior changes.

### Attribution line versions

The skill attribution line (see below) embeds the skill's version.
When bumping a version, update the hardcoded version string in the
skill's attribution instruction to match the new `metadata.version`.

## Skill Attribution

- **Every skill that creates or modifies Jira content must append a skill
  attribution line to the body of that content.** This applies to issue
  descriptions, comments, and any other text body sent to Jira.
- This is unconditional — it does not depend on the `aiDisclaimer` config
  option. The AI disclaimer is a user-facing notice about AI involvement;
  the attribution line identifies which skill and version produced the
  content.
- The attribution line is always the **last line** of the content body.
  If an AI disclaimer prefix is also present, the disclaimer goes at the
  top and the attribution goes at the bottom.
- Format: `\n\n_Created with {skill-name} v{version}_`
  - `{skill-name}` is the `name` field from the skill's YAML frontmatter
    (e.g. `jira-create`, `jira-update`).
  - `{version}` is the `metadata.version` field from the skill's YAML
    frontmatter (e.g. `0.1.1`).
  - Example: `_Created with jira-create v0.1.1_`
- For bulk operations, use the bulk skill's own name and version (e.g.
  `jira-bulk v0.1.1`), not the name of the underlying operation.

## Content Format

- Use `contentFormat: "markdown"` for `createJiraIssue`, `editJiraIssue`,
  and `addCommentToJiraIssue`. This allows natural markdown in descriptions
  and comments without needing to construct ADF (Atlassian Document Format)
  JSON.
- When reading issue content with `getJiraIssue`, use
  `responseContentFormat: "markdown"` for human-readable output.
