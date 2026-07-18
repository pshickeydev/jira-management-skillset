# Rovo MCP Tool Quick Reference

Signatures and usage notes for the Jira tools available via the Atlassian
Rovo MCP server. Skills should reference this file when they need to recall
parameter names or constraints.

## Platform Tools

### getAccessibleAtlassianResources
List all Atlassian sites accessible to the authenticated user. Returns
`cloudId` values needed by every other tool. Only used by `jira-configure`.
```
(no parameters)
```
Returns array of `{ id, url, name, scopes }`.

### atlassianUserInfo
Get the current authenticated user's account information. Use this to
resolve the user's `accountId` for auto-assignment.
```
(no parameters)
```
Returns `{ account_id, name, email, ... }`. The `account_id` value is
what you pass to `assignee_account_id` on `createJiraIssue` or to the
`assignee` field on `editJiraIssue`.

## Read Tools

### getJiraIssue
Fetch a single issue by key or ID.
```
cloudId:              string (required)
issueIdOrKey:         string (required)  — e.g. "PROJ-123" or "10000"
fields:               string[] (optional) — field names to return
                        default: summary, description, status, issuetype,
                        priority, labels, components, assignee, reporter,
                        created, updated, resolution, project
                        use ["*all"] for everything including custom fields
                        include "comment" to get comments
responseContentFormat: "markdown" | "adf" (optional)
```

### getJiraProjectIssueTypesMetadata
List issue types available in a project.
```
cloudId:          string (required)
projectIdOrKey:   string (required)
maxResults:       number (optional, default 50, max 200)
startAt:          number (optional, default 0)
```

### getJiraIssueTypeMetaWithFields
Get create-field metadata (required/optional fields) for a project + issue type.
```
cloudId:              string (required)
projectIdOrKey:       string (required)
issueTypeId:          string (required)  — from getJiraProjectIssueTypesMetadata
requiredFieldsOnly:   boolean (optional, default true)
                        set false to get all fields
maxResults:           number (optional)
startAt:              number (optional)
```

### getTransitionsForJiraIssue
List available workflow transitions for an issue.
```
cloudId:       string (required)
issueIdOrKey:  string (required)
```
Returns array of `{ id, name, to: { name, id } }`. Transition IDs differ
across projects — always look up, never hardcode.

### getVisibleJiraProjects
List projects the user can access.
```
cloudId:           string (required)
searchString:      string (optional)
expandIssueTypes:  boolean (optional, default true)
maxResults:        number (optional, default 50, max 50)
startAt:           number (optional, default 0)
action:            "view" | "browse" | "edit" | "create" (optional, default "create")
```

### lookupJiraAccountId
Find user account IDs by name or email.
```
cloudId:       string (required)
searchString:  string (required)
```

### getIssueLinkTypes
List available issue link types.
```
cloudId:  string (required)
```
Returns types with `inward` and `outward` descriptions.

## Write Tools

### createJiraIssue
Create a new issue.
```
cloudId:           string (required)
projectKey:        string (required)
issueTypeName:     string (required)  — e.g. "Bug", "Story", "Task"
summary:           string (required)
description:       string | ADF object (optional)
contentFormat:     "markdown" | "adf" (optional)
assignee_account_id: string (optional)
parent:            string (optional)  — parent issue key for subtasks
additional_fields: object (optional)  — REQUIRED for:
                     priority:   { "priority": { "name": "Medium" } }
                     labels:     { "labels": ["label1", "label2"] }
                     components: { "components": [{ "name": "Backend" }] }
                     security:   { "security": { "name": "Level Name" } }
                     custom:     { "customfield_XXXXX": value }
```

### editJiraIssue
Update fields on an existing issue.
```
cloudId:              string (required)
issueIdOrKey:         string (required)
fields:               object (required) — field names/IDs as keys
                        set value to null to clear a field
contentFormat:        "markdown" | "adf" (optional)
responseContentFormat: "markdown" | "adf" (optional)
```

### transitionJiraIssue
Perform a workflow transition.
```
cloudId:       string (required)
issueIdOrKey:  string (required)
transition:    { id: string } (required)  — from getTransitionsForJiraIssue
fields:        object (optional)
```

### addCommentToJiraIssue
Add or update a comment.
```
cloudId:           string (required)
issueIdOrKey:      string (required)
commentBody:       string (required)
contentFormat:     "markdown" | "adf" (optional)
commentId:         string (optional)  — set to update existing comment
commentVisibility: { type: "role"|"group", value: string } (optional)
```

### addWorklogToJiraIssue
Add or update a time-tracking worklog.
```
cloudId:       string (required)
issueIdOrKey:  string (required)
timeSpent:     string (required)  — e.g. "2h", "30m", "4d"
commentBody:   string (optional)
contentFormat: "markdown" | "adf" (optional)
started:       string (optional)  — ISO 8601
worklogId:     string (optional)  — set to update existing worklog
visibility:    { type: "role"|"group", value: string } (optional)
```

### createIssueLink
Create a relationship between two issues.
```
cloudId:       string (required)
type:          string (required)  — link type name from getIssueLinkTypes
inwardIssue:   string (required)  — issue key (e.g. "PROJ-1")
outwardIssue:  string (required)  — issue key (e.g. "PROJ-2")
comment:       string (optional)  — comment on the outward issue
contentFormat: "markdown" | "adf" (optional)
```
Directionality: for "A blocks B", set inwardIssue=A, outwardIssue=B.

## Search Tools

### searchJiraIssuesUsingJql
Search issues with JQL.
```
cloudId:              string (required)
jql:                  string (required)
fields:               string[] (optional)  — same defaults as getJiraIssue
maxResults:           number (optional, max 100)
nextPageToken:        string (optional)  — from previous response
responseContentFormat: "markdown" | "adf" (optional)
```
Pagination: uses `nextPageToken`, NOT `startAt`/`limit`.
