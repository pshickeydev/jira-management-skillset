---
name: jira-link
description: >-
  Create relationships between Jira issues — blocks, is blocked by,
  duplicates, relates to, clones, and other link types. Discovers
  available link types at runtime. Use when asked to link issues,
  mark as duplicate, set a blocker, or relate Jira issues.
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

### Step 1 — Parse the request

Extract from the user's input:
- **Source issue key** — the first issue mentioned
- **Target issue key** — the second issue mentioned
- **Relationship type** — inferred from language (see mapping below)

If any are missing, ask.

### Step 2 — Discover available link types

Call `getIssueLinkTypes` with `cloudId` to get the list of available
link types. Each type has:
- `name` — the link type name (e.g. "Blocks")
- `inward` — the inward description (e.g. "is blocked by")
- `outward` — the outward description (e.g. "blocks")

### Step 3 — Match the relationship

Map the user's language to a link type and direction:

| User says | Link type | inwardIssue | outwardIssue |
|-----------|-----------|-------------|--------------|
| "A blocks B" | Blocks | A | B |
| "A is blocked by B" | Blocks | B | A |
| "A duplicates B" | Duplicate | A | B |
| "A is a duplicate of B" | Duplicate | A | B |
| "A relates to B" | Relates | A | B |
| "A clones B" | Cloners | A | B |
| "link A to B" | (ask user) | — | — |

**Directionality is critical.** `createIssueLink` uses:
- `inwardIssue` — the issue described by the inward text
- `outwardIssue` — the issue described by the outward text

For "Blocks": inward = "is blocked by", outward = "blocks".
So "A blocks B" means: `inwardIssue: A, outwardIssue: B`.

If the user's intent doesn't clearly match a link type, present the
available types with their inward/outward descriptions and ask.

### Step 4 — Verify both issues exist

Call `getJiraIssue` for both issue keys in parallel with
`fields: ["summary", "status"]` to verify they exist and show the
user what's being linked:

```
Linking:
  {KEY-1}: {summary1} ({status1})
    ↓ {link type outward description}
  {KEY-2}: {summary2} ({status2})

Create this link?
```

Wait for explicit approval. Do NOT call any write tool until the user
confirms. If the user wants to change the link type or direction, go
back to Step 3.

### Step 5 — Create the link

Only after the user confirms, call `createIssueLink` with:
- `cloudId`
- `type` — the link type name
- `inwardIssue` — issue key
- `outwardIssue` — issue key
- `comment` — optional comment text
- `contentFormat: "markdown"` (if adding a comment)

If adding a comment on the link:
- Apply AI disclaimer prefix if `config.aiDisclaimer` is true
- Note: `createIssueLink`'s `comment` parameter does not support
  `commentVisibility`. If restricted comments are needed, create
  the link without a comment, then use `addCommentToJiraIssue`
  separately with the appropriate `commentVisibility` from config.

### Step 6 — Report

```
Linked: {KEY-1} {outward description} {KEY-2}
```

## Gotchas

- **Discover link types at runtime.** Call `getIssueLinkTypes` every
  time — available types can change when admins modify the instance
  configuration.
- **Directionality matters.** "A blocks B" and "B is blocked by A"
  produce the same link but require different inward/outward
  assignments. Always map from the user's language to the correct
  direction.
- The `type` parameter on `createIssueLink` is the link type **name**
  (e.g. "Blocks"), not an ID.
- Linking is idempotent — creating a duplicate link does not error
  but has no effect.
- Both issues must exist. If either `getJiraIssue` call fails, report
  the error and stop.
