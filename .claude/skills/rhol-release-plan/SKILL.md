---
name: rhol-release-plan
description: >
  Use when the user asks to create, open, or plan a Red Hat OpenShift Logging (RHOL) z-stream
  release ticket in Jira — e.g. "create the 6.6.3 release ticket", "set up the next z-stream
  release", "plan release 6.2.15", or any request referencing the release template LOG-4933.
  Creates the LOG release tracking issue (dates, fix version, due date, sprint, assignee) and the
  ART release request issue (template ART-14539), linked to the LOG issue.
---

# RHOL Release Plan

Create the LOG Jira release tracking issue for an OpenShift Logging z-stream release, based on the
template **LOG-4933** ("[TEMPLATE] OpenShift Logging z-stream: 6.y.z"), plus the matching ART
release request issue (template **ART-14539**).

All Jira work goes through the **Atlassian MCP tools** (cloudId for redhat.atlassian.net:
`2b9e35e3-6bd3-4cec-b838-f4249ee02432`). Do not use `acli` for descriptions or custom fields —
see the `acli-jira-quirks` memory: MCP converts markdown to proper ADF and is the only reliable
path for labels, fix versions, due dates, sprints, and assignee.

## Inputs — ask the user before creating anything

Ask (in one batch, via AskUserQuestion or a single question):

1. **Release version** (e.g. `6.6.3`) — becomes the issue title and usually the fix version
2. **Release Freeze date** (YYYY-MM-DD)
3. **Candidate Build date** (YYYY-MM-DD)
4. **Release Date** (YYYY-MM-DD)
5. **Fix version** to set — defaults to `Logging <version>` (e.g. `Logging 6.2.14`); confirm
   with the user if they want something different

The sprint is derived, not asked: pick the sprint whose date range contains the Release Date.
The assignee is always the requesting user.

## Steps

### 1. Create the issue (clone of LOG-4933)

Fetch LOG-4933 with `getJiraIssue` (view: full) to get its current description, labels, priority,
and issue type (Task). Then `createJiraIssue` in project LOG with:

- **Summary:** `OpenShift Logging z-stream: <version>` (no `[TEMPLATE]` prefix)
- **Type:** Task, **Priority:** same as template (Major)
- **Labels:** copy the template's labels, **remove `template`** if present (expected set:
  `groomed`, `no-errata`, `no-rn`, `z-stream`)
- **Description:** the template description verbatim, with the three date placeholders in the
  "Important Dates" table replaced:
  - `Release Freeze Date` → user's freeze date
  - `Candidate Build Ready Date` → user's candidate build date
  - `Release Date` → user's release date

Add a Cloners link to the template: `editJiraIssue` is unreliable for links — use `discover` to
find the remote/link create operation, or note the lineage in the first comment if linking fails.

### 2. Subtasks (only if any exist)

If the created issue (or the clone source) has subtasks whose summaries start with `CLONE `,
strip the `CLONE ` prefix from each summary via `editJiraIssue`. LOG-4933 normally has **no**
subtasks — if there are none, skip this step silently; component subtasks are filed later per the
description's instructions.

### 3. Due date, fix version

Via `editJiraIssue` on the new issue:

- **Due date** = Release Date **+ 1 day**: `{"duedate": "YYYY-MM-DD"}`
- **Fix versions**: `{"fixVersions": [{"name": "<fix version>"}]}` — the version name follows
  the pattern `Logging <version>` (e.g. `Logging 6.2.14`)

If the fix version does not exist in the LOG project, the edit fails — check with
`getJiraProjectVersions` (executeRead); if missing, create it with `manageJiraProjectVersion`
(executeWrite, action create), then retry. Do not invent a version name the user did not give.

### 4. Sprint

1. `listJiraBoards` (executeRead, projectKeyOrId: LOG) to get the LOG board id.
2. `listJiraBoardSprints` (executeRead, boardId, state: all) and pick the sprint whose
   start/end dates contain the Release Date. Sprint names follow the pattern
   `Logging - Sprint 297` — use the name to sanity-check the pick.
3. Add the issue: `manageJiraSprint` (executeDestructive, action `addIssues`,
   sprintId, issueIdsOrKeys: [new issue key]).

If no sprint covers the Release Date, tell the user and ask which sprint to use — do not guess.

### 5. Assign to the user

`editJiraIssue` with `additional_fields: {"assignee": {"accountId": "<user's accountId>"}}`.
Get the accountId from `atlassianUserInfo` (the requesting user). Do **not** transition the issue
to any other status.

### 6. Create the ART release issue

Template: **ART-14539** ("[template] Operator release request") — if the user says "ART-1453",
that is an unrelated Bug; ART-21925's actual clone parent is ART-14539. See **ART-21925**
("Release Logging 6.5.2 [2026-Aug-18]") as a filled-in example.

**First check for a duplicate:** search the ART project for
`summary ~ "Release Logging <version>"` — these issues are often filed ahead of time. If one
exists, report it and skip creation.

Create via `createJiraIssue` in project ART:

- **Summary:** `Release Logging <version> [<release date>]` — date format `YYYY-Mmm-DD`, e.g.
  `Release Logging 6.6.1 [2026-Sep-10]`
- **Type:** Story; **Labels:** `art:release:schedule`
- **Description:** the ART-14539 template description with:
  - **GROUP** filled in: `logging-<major.minor>` (the ocp-build-data branch, e.g. `logging-6.5`)
  - **ASSEMBLY** filled in: `<version>` (e.g. `6.5.2`)
  - **Build Information** and **JIRA issues** sections left as template placeholders — they are
    filled in later once builds and the shipping issue list are known
- **Due date** = Release Date (matches ART-21925; ART needs 5 working days lead time)
- **Reporter** = the user: pass the user's accountId (from `atlassianUserInfo`) as the `reporter`
  on `createJiraIssue`. If the create call rejects the reporter field, retry without it and then
  set it via `editJiraIssue` with `additional_fields: {"reporter": {"accountId": "..."}}`.

Then link it to the LOG issue created above: a **Related** link, direction "is related to", from
the ART issue to the LOG issue. Use `discover` to find the issue-link create operation
(e.g. "link two jira issues").

### 6a. Fill the ART issue's JIRA issues list (later, near code freeze)

When it's time to populate the ART issue's **JIRA issues** section, **run the
`release-issue-auditor` skill** with the release version. It queries the Jira MCP server with:

```text
project = LOG AND fixVersion in ("Logging <X.Y.z>", "Logging <X.Y.Z>") AND status not in (New, "To Do", Assigned, Post, "Code Review", "In Progress") ORDER BY issuetype, key
```

and audits the z-stream checklist: QE-ready statuses (Modified for bugs, Review for tasks),
`/hold` on open PRs of Code Review/Post issues, release notes or `no-rn`, private-issue handling,
one issue per release for multi-release changes, and sprint assignment. **Issues that are not
ready are resolved by asking the user** how to handle each one — never auto-fix or silently
exclude.

Take the surviving list from the audit and update the ART issue description's **JIRA issues**
section via `editJiraIssue`, grouped `# CVEs` then `# Fixes` (see ART-21925 for the format).

### 6b. Fill the ART issue's Build Information (later, at candidate selection)

Identify the FBC builds and the eventrouter image to be used for the release. Component and FBC
builds live in Konflux (dots in the stream become dashes in the application name). URL patterns
(replace `X-Y`, e.g. `6-5`):

- Components: `https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-logging-tenant/applications/logging-<X-Y>/components`
- FBC: `https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-logging-tenant/applications/fbc-logging-<X-Y>/components`

Steps:

1. **Check recent component builds** happened and were successful (PipelineRuns for the release,
   e.g. `logging-6.6`).
2. **Check recent staging releases** were successful and are **newer than the component builds**
   (releases page, e.g. `logging-6.6`). This matters mostly for the **eventrouter** image — it is
   not part of the operators and thus the FBCs, so it must be mentioned separately in the release.
   The last staging release was probably triggered by a bundle build (CLO or LO), unless the last
   change was to eventrouter.
3. **Pick the eventrouter candidate:** the most recent eventrouter release. Open it and note the
   NVR for the eventrouter image from its **Release Artifacts** page, e.g.
   `ose-logging-eventrouter-container-6.5.2-202608131800.p2.ge2c2ae9.assembly.stream.el9`.
4. **Check the FBC builds** for the release (e.g. `fbc-logging-6.6`). Releases usually target
   4 OCP versions and consist of two operators, so there are **two groups of four runs** per
   release. The operators build independently — the four FBC builds for one operator may be older
   than the other operator's four. Identify the latest successful **group of four runs** for each
   operator.
5. In each PipelineRun, note the **FBC tag** from the "additional tags" list in the "apply-tags"
   step, e.g.
   `v4.19__operator_nvr__ose-cluster-logging-operator-container-6.5.2-202608132150.p2.g75664fd.assembly.stream.el9`.
6. You now have one eventrouter image and **eight FBC tags**. Prefix each FBC tag with the
   repository to form the full image name:
   `quay.io/redhat-user-workloads/ocp-art-tenant/art-fbc:<tag>`
7. Write both into the ART issue's **Build Information** section via `editJiraIssue`: the eight
   FBC pullspecs in the FBC block (grouped per operator, CLO then Loki, like ART-21925) and the
   eventrouter NVR under extra images.

### 7. Report

Give the user the new issue keys and URLs (LOG and ART), and a summary of what was set: titles,
labels, dates, due dates, fix version, sprint name, assignee. List anything skipped or failed
(e.g. missing fix version, no covering sprint, pre-existing ART issue) rather than silently
omitting it.

## Common Mistakes

- Copying the `[TEMPLATE]` prefix or `YYYY-MM-DD` placeholders into the new issue
- Leaving a `template` label on the release issue
- Setting the due date to the Release Date instead of Release Date + 1
- Using `acli` for the description (stores raw markdown as one ADF text node)
- Assigning to a sprint that merely is "current" instead of the one containing the Release Date
- Creating subtasks — this skill never creates them; they are filed per-request later
- Filling in the ART issue's Build Information or JIRA issues sections at creation time (6a/6b
  happen later, near code freeze)
- Mixing FBC runs from different "groups of four" — each operator's four runs must come from the
  same latest successful group
- Using a staging release that is older than the component builds it should contain
- Cloning ART-1453 (an unrelated Bug) instead of the real template ART-14539
- Creating a duplicate ART issue — always search for `Release Logging <version>` first
