---
name: guide-generator
description: "Auto-generates a verification guide from a feature branch diff: which database tables to query, which log events to look for, and which endpoints to call when verifying a manual test. Stack-agnostic — reads .claude/qa-config.yml. Used as input by flow-verifier.\n\n<example>\nuser: \"Generate a guide for feature/PROJ-123-new-signup\"\nassistant: \"I'll use the guide-generator agent to analyze the branch and produce a verification guide.\"\n</example>"
model: inherit
---

# Verification Guide Generator

The checklist tells a tester **what to do**. This guide tells them — and
`flow-verifier` — **where to look afterwards**: the tables that should have
changed, the log lines that should have appeared, the endpoints that were hit.

## Invocation

```
@guide-generator branch=<branch> [repo=<path>] [lang=<code>]
```

| Argument | Required | Default | Example |
|---|---|---|---|
| `branch` | yes | — | `feature/PROJ-123-new-signup` |
| `repo` | no | `project.repo_path` from config | `../backend` |
| `lang` | no | `project.output_language` from config | `ru` |

Announce before starting: *"Generating verification guide for `{branch}`…"*

---

## Step 0 — Load configuration

Read `.claude/qa-config.yml` from the current working directory. If it is
missing, auto-detect the stack using the same table as `checklist-generator`
Step 0, print what you inferred, and continue.

You specifically need: `{project.repo_path}`, `{project.base_branch}`,
`{stack.layers}`, `{data.provider}`, `{data.schema}`,
`{data.identifier_fields}`, `{logs.provider}`, `{logs.sources}`.

Write the guide in `{project.output_language}`. Never translate identifiers,
SQL, log strings, table names, or endpoint paths — the guide's whole value is
that these are copy-pasteable.

---

## Step 1 — Resolve the repo and verify the branch

`REPO` = `{project.repo_path}` (default `.`).

```
git -C {REPO} branch -a --list "*{branch}*"
```

Resolve `REF` (prefer `origin/{branch}`) and `BASE` (from
`{project.base_branch}`). If the branch does not exist, stop and say so.

---

## Step 2 — Categorize the diff

```
git -C {REPO} diff {BASE}...{REF} --stat
```

Drop `{stack.skip_globs}`, then bucket by `{stack.layers}` globs. The layers map
onto guide sections:

| Layer | Feeds guide section |
|---|---|
| `data_model` | **Database** — tables, columns, access rules |
| `entrypoint` | **Endpoints** |
| `logic` | **Log events** |
| `async` | **Async triggers** |
| `ui` | flow names only — no guide section of its own |

---

## Step 3 — Extract database artifacts

Read every `data_model` file with `git -C {REPO} show {REF}:<path>`.

**From migrations** (SQL, XML, or code-based — whatever this stack uses):

- table creation → new table
- column addition → new column on an existing table
- column rename → record **both** names; existing rows still carry the old one
- nullability / default changes → these are where existing rows break
- **authorization rules** (`create policy`, `grant`, RLS, ACL) → record who can
  now read or write, and who can no longer. This is a verifiable fact: a query
  run as the wrong user should return zero rows.
- functions and triggers → record what fires them and what they write

**From ORM entities / models**, if the stack has them: the table name, the
identifier column, and columns matching `{data.identifier_fields}`.

For each table, note the identifier column that links a row back to one of
`{data.identifier_fields}` — `flow-verifier` needs it to build a `WHERE` clause.

---

## Step 4 — Extract log events

Read every `logic` file changed in the diff. Find logging calls at decision
points — successful outcomes, state transitions, and failure branches. The call
shape depends on the stack (`log.info`, `logger.warning`, `console.error`,
`slog.Info`, …); the useful part is the **literal message string**, because that
is what a log search will match.

Record for each: the exact string, the condition that produces it, and the
severity. Group events by the flow they belong to. If a checklist already exists
at `{project.artifacts_dir}/checklists/checklist-{branch-slug}.md`, read it and
reuse its flow numbering so both documents line up.

If `{logs.provider}` is `none`, skip this step and say so in the guide.

---

## Step 5 — Extract endpoints

Read every `entrypoint` file. Record:

- HTTP method and full path — including any prefix declared where the route is
  registered, not just the local fragment
- expected success status code, and every explicit error status
- request body shape and URL/query parameters
- authentication requirement — public, authenticated, or role-restricted
- whether it is hidden from public API docs (still testable; note it)

---

## Step 6 — Extract async triggers

Read every `async` file. Record the trigger (queue topic, schedule expression,
webhook source), what it processes, and the observable side effect. A scheduled
job with no visible output still writes something — find it, or the flow cannot
be verified.

---

## Step 7 — Merge with an existing guide

Glob `{project.artifacts_dir}/guides/api-db-guide-*.md`.

- A guide for this same branch-slug exists → **update it**: add newly found
  tables, events and endpoints; never delete existing content.
- Guides exist for other branches → create a new file; do not touch theirs.
- None → create from scratch.

---

## Step 8 — Write the guide

**File:** `{project.artifacts_dir}/guides/api-db-guide-{branch-slug}.md`

`branch-slug` = branch name with any leading `feature/`, `bugfix/`, `claude/`
segment removed and remaining `/` replaced by `-`.

Create the directory if needed. Omit any section that has no content — an empty
`## Kafka` heading is noise.

````markdown
# Verification guide — {branch}

**Branch:** {branch} | **Project:** {project.name} | **Date:** {today}
**Data:** {data.provider} | **Logs:** {logs.provider}

---

## Flow scope

| Flow | Log sources | Tables | Endpoints |
|---|---|---|---|
| F1 | {source} | {table1}, {table2} | `POST /api/…` |

---

## Database

### {table_name}

- **Schema:** {data.schema}
- **Identifier column:** {column}
- **Query:**
  ```sql
  SELECT * FROM {schema}.{table_name}
  WHERE {identifier_column} = '<identifier>'
  ORDER BY created_at DESC;
  ```
- **What to check:** {columns and the values expected after the test}
- **Access rule:** {who may read/write this row, if the diff changed it}

---

## Log events

### Flow F1 — {description}

- `"{exact log string}"` — appears when {condition}. Check: `{field}` = `{value}`

---

## Endpoints

### {METHOD} {path}

- **When called:** {condition}
- **Auth:** public / authenticated / role: {role}
- **Expected:** HTTP {status} — `{response example}`
- **Errors:** {status} when {condition}
- **Note:** {e.g. not present in public API docs}

---

## Async triggers

### {topic | schedule | webhook}

- **Fires when:** {condition}
- **Observable effect:** {what changes that a tester can see}
````

---

## Step 9 — Report

> Guide saved: `{path}` — {N} tables, {M} log events, {K} endpoints.

If `{data.provider}` or `{logs.provider}` is `none`, add a line naming which
verification channel is unavailable, so nobody expects `flow-verifier` to check
it later.
