---
name: qa-pipeline
description: "Orchestrator: runs checklist-generator and guide-generator in parallel for a feature branch, cross-checks the two outputs for drift, then optionally runs checklist-enricher when notes= is given. Use when you want the full test checklist and verification guide in one command.\n\n<example>\nuser: \"@qa-pipeline branch=feature/PROJ-123-new-signup\"\nassistant: \"Running checklist-generator and guide-generator in parallel for feature/PROJ-123-new-signup…\"\n</example>\n\n<example>\nuser: \"@qa-pipeline branch=feature/PROJ-123 notes=.claude/qa/notes/PROJ-123.md\"\nassistant: \"Running both generators in parallel, then enriching with the QA notes…\"\n</example>"
model: inherit
---

# QA Pipeline Orchestrator

Runs the two generators concurrently — they read the same diff but write
different documents, so there is no reason to serialize them — then checks the
two results agree.

## Invocation

```
@qa-pipeline branch=<branch> [repo=<path>] [notes=<path>] [lang=<code>]
```

| Argument | Required | Default | Example |
|---|---|---|---|
| `branch` | yes | — | `feature/PROJ-123-new-signup` |
| `repo` | no | `project.repo_path` from config | `../backend` |
| `notes` | no | — | `.claude/qa/notes/PROJ-123.md` |
| `lang` | no | `project.output_language` from config | `ru` |

Announce: *"Running the QA pipeline for `{branch}` ({project.name})…"*

---

## Step 0 — Load configuration

Read `.claude/qa-config.yml` so you can pass a consistent context to both
sub-agents and resolve the artifact paths afterwards. If it is missing, say so
once here rather than letting each sub-agent report it separately.

---

## Phase 1 — Generators, in parallel

Dispatch **both agents in a single message** so they run concurrently.

Agent 1:
```
Run checklist-generator for branch={branch} repo={repo} lang={lang}.
Working directory: {CWD}.
Follow the instructions in .claude/agents/checklist-generator.md.
```

Agent 2:
```
Run guide-generator for branch={branch} repo={repo} lang={lang}.
Working directory: {CWD}.
Follow the instructions in .claude/agents/guide-generator.md.
```

Wait for both to finish.

If either fails, report which one and why, then continue with whatever
succeeded — a checklist without a guide is still useful, and so is the reverse.

---

## Phase 2 — Cross-check the two documents

Both agents read the same diff independently. Where they disagree, one of them
missed something — and that is worth knowing *before* a tester starts work.

Read both saved files and compare:

| Check | Warning when it fails |
|---|---|
| Every flow in the checklist appears in the guide's `## Flow scope` | flow `{N}` has no verification criteria — a tester can run it but cannot confirm the result |
| Every flow in the guide's `## Flow scope` has a `## Flow N` block in the checklist | flow `{N}` is verifiable but nobody is asked to run it |
| Every endpoint in a checklist flow's `Trigger:` line appears under the guide's `## Endpoints` | endpoint `{path}` is exercised but undocumented |
| Every endpoint in the guide appears in some checklist flow's `Trigger:` line | endpoint `{path}` is documented but untested |

A flow's `Trigger:` line may name a screen action, a schedule or a webhook
instead of an HTTP endpoint — UI-only and job-only flows legitimately have no
endpoint. Apply the last two checks only to triggers that actually look like an
endpoint (a method plus a path). Do not warn about a missing endpoint for a flow
that never had one.

Likewise, a flow whose evidence is entirely client-side — browser storage,
rendering, local state — has no server-side criteria by nature. When the guide
says so explicitly for that flow, treat it as agreement, not drift.

Collect the remaining warnings. An empty list means the two views of the diff
agree — say so, it is a real signal.

---

## Phase 3 — Enrichment (only when `notes=` is given)

Runs after Phase 1 because it edits those outputs. Dispatch sequentially:

```
Run checklist-enricher for branch={branch} notes={notes} repo={repo} lang={lang}.
Working directory: {CWD}.
Follow the instructions in .claude/agents/checklist-enricher.md.
```

Wait for it to finish. Skip this phase entirely when `notes=` is absent.

---

## Report

Read the saved files to count what you report — do not estimate.

- `branch-slug` — branch with any leading `feature/`, `bugfix/`, `claude/`
  segment removed, remaining `/` replaced by `-`
- `N` — scenario rows across all flows in the checklist
- `M` — flows
- `T` — tables in the guide
- `E` — log events in the guide
- `K` — rows tagged `[QA-NOTE]`, when the enricher ran

```markdown
## QA pipeline — {branch}

✅ Checklist: `{artifacts_dir}/checklists/checklist-{branch-slug}.md` — {N} scenarios in {M} flows
✅ Guide:     `{artifacts_dir}/guides/api-db-guide-{branch-slug}.md` — {T} tables, {E} log events
```

When the enricher ran, mark the checklist line `(enriched)` and append
`({K} from notes)`.

When Phase 2 produced warnings, add them — this is the part people act on:

```markdown
⚠️ Checklist/guide drift:
- {warning}
```

Close with the next step:

```markdown
Next: run the tests manually, then verify each one:
@flow-verifier id=<identifier> flow=<F1-01>
```

---

## Scope

This orchestrator does not call `flow-verifier` — that runs after a human has
actually performed the test, which is not something the pipeline can wait for.
