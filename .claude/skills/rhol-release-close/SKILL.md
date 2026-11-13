---
name: rhol-release-close
description: >
  Use when the user asks to close, ship, or release a RHOL z-stream release in Jira after its
  ART shipment MR merges — e.g. "close 6.5.4", "release Logging 6.5.4". Takes a shipment MR
  URL or bare version number (finds the MR by title), extracts the prod errata URL from the
  shipment image YAML, and updates the Logging X.Y.Z Jira release: description, release date,
  released status, and offers to close the release tracking issue. Optionally creates the
  next z-stream version (start date = release date + 1 day) and bumps the operator versions
  (CLO, loki-operator) on their release-X.Y branches via rhol-version-incrementer.
---

# RHOL Release Close

Close out a RHOL z-stream release in Jira once its ART shipment MR has merged: put the errata
URL on the release, set the release date, and mark the version released.

## Role

You are closing the Jira **project version** (release) named `Logging <X.Y.Z>` in project LOG.
The version — not an issue — holds the description, release date, and released status.

## Inputs

The user gives either a **shipment MR URL** in the `ocp-shipment-data` repository (e.g.
`https://gitlab.cee.redhat.com/hybrid-platforms/art/ocp-shipment-data/-/merge_requests/1159`)
or a **bare version number** ("close 6.5.4" — run Step 0 to find the MR). If neither is
given, ask — do not guess a version or construct an MR URL.

## Step 0 — Find the MR from a version number (only when no MR URL was given)

List the project's merge requests matching the shipment title pattern
`Shipment for openshift-logging <version> (ship date: YYYY-Mmm-DD)`:

```bash
PROJECT="hybrid-platforms%2Fart%2Focp-shipment-data"
VERSION="6.5.4"

curl -s "https://gitlab.cee.redhat.com/api/v4/projects/$PROJECT/merge_requests?state=opened&search=openshift-logging%20$VERSION&per_page=20" | \
python3 -c "import json,sys; [print(m['iid'], m['state'], '-', m['title']) for m in json.load(sys.stdin)]"
```

- The `search` parameter is a case-insensitive substring match — always print the titles and
  **verify each matches the exact pattern** `Shipment for openshift-logging <version> (ship
  date: ...)`; the search can return other products or nearby versions (`6.5.4` matches
  `6.5.40`).
- Search `state=opened` first, then `state=merged` (then `state=all`) — the matching MR is
  often already merged. Report which state you found it in.
- One match → proceed (skip the Step 1 re-fetch if you already have the JSON). Multiple →
  list them and ask. None → stop; do not close a release without a shipment MR.

## Prerequisites

- Network access to `gitlab.cee.redhat.com` (anonymous read works from the corp network; on
  401/403/404 tell the user rather than retrying)
- Atlassian MCP tools; cloudId for redhat.atlassian.net:
  `2b9e35e3-6bd3-4cec-b838-f4249ee02432`

## Instructions

### Step 1 — Fetch the MR

Parse the URL as `https://gitlab.cee.redhat.com/<project-path>/-/merge_requests/<iid>`,
URL-encode the project path, and fetch the MR metadata:

```bash
PROJECT="hybrid-platforms%2Fart%2Focp-shipment-data"
MR_IID="1159"

curl -s "https://gitlab.cee.redhat.com/api/v4/projects/$PROJECT/merge_requests/$MR_IID"
```

From the MR JSON extract:

- **title** — e.g. `Shipment for openshift-logging 6.5.4 (ship date: 2026-Nov-12)`.
  It yields the **version** (`6.5.4`) and the **ship date** (`YYYY-Mmm-DD`). Convert the ship
  date to ISO with Python `strptime(..., '%Y-%b-%d')` — GNU `date -d` rejects this form.
- **state** — normally `merged`. If it is `opened` (e.g. the user is closing the release while
  the shipment MR is still open), warn them that the MR has not merged yet and get explicit
  confirmation before proceeding; do not close the release early on your own judgment.

### Step 2 — Extract the errata URL from the shipment files

Fetch the MR diffs:

```bash
curl -s "https://gitlab.cee.redhat.com/api/v4/projects/$PROJECT/merge_requests/$MR_IID/diffs"
```

Each entry has `new_path` and `diff`. The MR touches two kinds of files:

- `prod/<version>.fbc.<ocp-ver>.<timestamp>.yaml` — FBC catalog files; **ignore these**
- `prod/<version>.image.<timestamp>.yaml` — the **image advisory file**; this is the one to read

From the image file's diff, read `environments.prod.advisory.url`, e.g.
`https://access.redhat.com/errata/RHSA-2026:99823` — the **errata URL**. Take it from `prod`,
**not** `stage` (host `access.stage.redhat.com`); the prod host is exactly
`access.redhat.com`, and the advisory id contains a **colon** — don't write a regex that
stops at the digits:

```bash
curl -s "https://gitlab.cee.redhat.com/api/v4/projects/$PROJECT/merge_requests/$MR_IID/diffs" | \
python3 -c "
import json,sys,re
diffs=json.load(sys.stdin)
imgs=[f for f in diffs if re.search(r'/prod/[^/]*\.image\.[0-9]+\.yaml$', f['new_path'])]
assert len(imgs)==1, f'expected 1 image file, got {len(imgs)}'
d=imgs[0]['diff']
asm=re.search(r'assembly: (\S+)', d).group(1)
url=re.search(r'url: (https://access\.redhat\.com/errata/RH[A-Z]+-[0-9]+:[0-9]+)', d).group(1)
print('assembly:', asm); print('errata:', url)
"
```

Sanity checks — stop and tell the user instead of proceeding if any fails:

- The image file's `shipment.metadata.assembly` (e.g. `6.5.4`) matches the version from the MR
  title
- There is exactly one image file (if several, they must all carry the same prod advisory URL;
  if they differ, report the conflict)
- The URL matches `https://access.redhat.com/errata/RH[A-Z]+-`

### Step 3 — Find the Jira release

The Jira version name is `Logging <version>` (e.g. `Logging 6.5.4`). Look it up with
`getJiraProjectVersions` (executeRead, projectIdOrKey `LOG`, query `Logging <version>`) and
keep its `versionId`.

If the version does not exist in the LOG project, stop and tell the user — do not create it
(rhol-release-plan owns creation).

If the version is already `released: true`, report that and ask the user before overwriting
anything.

### Step 4 — Summarize and confirm

Present a summary and get the user's confirmation (AskUserQuestion) before writing: the
release version name and versionId; the errata URL that will become the description; the
release date (ship date from the MR title, ISO); that the version will be marked released.

The established description convention is the **bare errata URL** — no label, no extra prose
(see released versions like `Logging 6.5.3` or `Logging 6.5.2` for examples).

### Step 5 — Update the release

Apply all three changes in one `manageJiraProjectVersion` call (executeDestructive),
action `update`:

```json
{
  "cloudId": "2b9e35e3-6bd3-4cec-b838-f4249ee02432",
  "action": "update",
  "versionId": "<versionId>",
  "description": "<errata URL>",
  "releaseDate": "<YYYY-MM-DD>",
  "released": true
}
```

`released` must be a real boolean. Use action `update` (not `release`) so the description and
release date land in the same call. Only pass `moveUnfixedIssuesTo` if the user explicitly asks
to sweep unresolved issues into a later version — never decide that yourself.

### Step 6 — Report

Report what was written: version name, errata URL, release date, released status. Mention
anything skipped or failed (unmerged MR, missing version, already released) rather than
silently omitting it.

### Step 7 — Offer to create the next z-stream release

After a successful close, offer to create the next z-stream release version in the LOG
project, e.g. closing `6.5.4` → create `Logging 6.5.5`. Do not create it unasked — ask the
user first (AskUserQuestion), then:

- **Version name:** `Logging <X.Y.(Z+1)>` — bump only the patch component of the version just
  closed. If the user asks for a different version, use theirs.
- **Start date:** the closed release's **release date + 1 day** (`datetime.date.fromisoformat`
  + `timedelta(days=1)`)
- **Release date:** leave unset — it is not known yet and comes from the next release's
  planning (see the `rhol-release-plan` skill, which owns the full release ticket workflow;
  this step only creates the Jira version).

Before creating, check the version does not already exist:
`getJiraProjectVersions` (executeRead, projectIdOrKey `LOG`, query `Logging <new version>`).
If it exists, report it and skip creation.

Create it with `manageJiraProjectVersion` (executeDestructive), action `create`:
`projectIdOrKey: "LOG"`, `name: "Logging <X.Y.(Z+1)>"`, `startDate: "<release date + 1 day>"`
(same cloudId). Then report the new version name, its versionId, and its start date.

### Step 8 — Offer to bump the operator versions for the new release

After creating the next z-stream release (or when the user asks to "bump the versions" for
it), offer to bump the **operators only** — cluster-logging-operator and loki-operator —
using the **`rhol-version-incrementer`** skill. Do not bump unasked, and do not bump the
other components (LFME, eventrouter) as part of release close.

Both operators have a branch per z-stream: the bump for an `X.Y.Z` release runs against the
matching `release-X.Y` branch of the operator's repo, and the PR targets that branch (never
`main`):

| Operator | Repository | Bump branch (for X.Y.Z) |
|---|---|---|
| cluster-logging-operator | `github.com/openshift/cluster-logging-operator` | `release-X.Y` (e.g. `release-6.5` for 6.5.5) |
| loki-operator | Red Hat fork `github.com/openshift/loki` (**not** the workspace `loki/` clone, which tracks `grafana/loki`) | `release-X.Y` (same stream as the release, e.g. `release-6.5` for 6.5.5) |

Also bump **ocp-build-data** (per
[openshift-eng/ocp-build-data#13169](https://github.com/openshift-eng/ocp-build-data/pull/13169)):
on its per-stream branch `logging-X.Y`, edit the top-level `group.yml` `version:` line to the
target version — nothing else changes. There is no workspace clone: fetch the branch into a
worktree and PR against `logging-X.Y` (not `main`). Commit message and PR title:
`chore: update logging version to <X.Y.Z>`.

Notes:

- The target version equals the new release's `<X.Y.Z>` for both operators and the
  ocp-build-data group — the two operators do **not** stream independently; both are bumped
  to the same version on their same-stream `release-X.Y` branches.
- Check out a worktree of the matching `release-X.Y` branch and pass it as the bump script's
  repo-path; create PRs with `--base release-X.Y`. Dry-run first, show the user the diff, and
  get confirmation before any commit/push/PR.

### Step 9 — Offer to close the release tracking issue (always last)

Only after everything else is done, offer to close the LOG **release tracking issue**
(summary `OpenShift Logging z-stream: <version>`, created by `rhol-release-plan`; e.g.
LOG-10532 for 6.5.4). If its key is not already known, ask the user for it — or search
`project = LOG AND summary ~ "OpenShift Logging z-stream: <version>"` with
`searchJiraIssuesUsingJql`. Never guess an issue key.

With the user's confirmation, transition it to **Closed**: get the transition whose
`to.name` is `Closed` via `listJiraIssueTransitions` (executeRead), then apply it with
`transitionJiraIssue` (same cloudId). Report the final status.

## Common Mistakes

- Reading the advisory URL from `stage` instead of `prod`, or from an `.fbc.` file instead of
  the `.image.` file
- Treating the release as a Jira issue — it is a project version; `editJiraIssue` and
  `transitionJiraIssue` do not apply
- Marking the version released before the MR has merged without warning the user and getting
  confirmation; or overwriting an already-released version without asking
- Trusting the GitLab `search` substring match blindly — `6.5.4` also matches `6.5.40`;
  always re-verify the full title pattern
- Writing to Jira without showing the user the summary first
- Creating the next z-stream version unasked, giving it a release date, or bumping the minor
  instead of the patch
- Bumping LFME or eventrouter as part of release close — only the two operators (on their
  `release-X.Y` branch) and the ocp-build-data group (on its `logging-X.Y` branch) are
  bumped; never PR these against `main`
- Running the loki-operator bump against the workspace `loki/` clone (it tracks grafana/loki),
  or bumping the two operators to different versions — they always move together to the
  release's `<X.Y.Z>` on their same-stream `release-X.Y` branches
