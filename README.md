# QA agents for Claude Code

Five Claude Code subagents that turn a feature branch into a testable
deliverable: a black-box test checklist, a verification guide, and an
evidence-backed PASS/FAIL verdict once a human has run the test.

They are stack-agnostic. A single `.claude/qa-config.yml` tells them how to read
*your* codebase — which files are entrypoints, which hold business rules, where
the schema lives, how to reach your database and logs. The same five agents run
against a Java backend and a React SPA without a line of the agents changing.

---

## The problem they solve

A feature branch lands and someone has to test it. Two things reliably go wrong:

1. **The checklist is written from the code**, so it reads like the code —
   "verify `DiscountResolver` returns `BigDecimal.ZERO`" — and only its author
   can execute it.
2. **"Tested, works"** is asserted with no evidence attached, so nobody can
   re-check it later, and a passing test and an untested one look identical in
   the ticket.

These agents attack both. Every scenario is written in black-box language and
gated on that rule. Every verdict prints the log lines and query results it was
derived from, and returns **INCONCLUSIVE** rather than PASS when the evidence
isn't there.

---

## The agents

| Agent | Input | Output |
|---|---|---|
| `qa-pipeline` | a branch | orchestrates the two generators in parallel, then cross-checks them |
| `checklist-generator` | a branch diff | `checklist-<branch>.md` — prioritized manual scenarios |
| `guide-generator` | a branch diff | `api-db-guide-<branch>.md` — where to look to confirm a result |
| `checklist-enricher` | a checklist + free-form notes | scenarios added for notes the code confirms; the rest reported as skipped |
| `flow-verifier` | an identifier + a flow id | PASS / FAIL / INCONCLUSIVE with evidence |

```
                 ┌─ checklist-generator ─┐
   branch ──────►│                        ├──► cross-check ──► [notes?] ──► enricher
                 └─ guide-generator ──────┘
                                                                     │
   human runs the test ──────────────────────────────────────────────┘
                                                                     ▼
                                                              flow-verifier
```

### Design rules the agents follow

- **Black-box or it doesn't ship.** Scenario text may not contain class names,
  function names, table or column names. Each agent carries the rule and a
  rewrite table.
- **Evidence gates invention.** `checklist-enricher` turns a note into a
  scenario only when it finds the code that backs it, and lists the notes it
  rejected. `flow-verifier` never fabricates a query result.
- **Absence is a verdict.** A missing tool, an empty table, or an unreachable
  log source produces INCONCLUSIVE with a stated next action — never a
  silent PASS.
- **Disagreement is a signal.** The two generators read the same diff
  independently; `qa-pipeline` reports where they diverge, because that gap is
  usually where a flow has no way to be verified.

---

## Install

Clone this repo, copy the agents into the project you want to test, then pick
the config closest to your stack:

```bash
git clone https://github.com/azeezello/claude-qa-agents.git
cd claude-qa-agents

mkdir -p <your-project>/.claude/agents
cp agents/*.md <your-project>/.claude/agents/

# pick one of config/example-*.yml
cp config/example-react-supabase.yml <your-project>/.claude/qa-config.yml
```

Then edit `qa-config.yml` for your codebase. Start a new Claude Code session —
agents are loaded at startup, so an existing session will not see them.

**No config?** The agents auto-detect the stack from `pom.xml`, `package.json`,
`pyproject.toml`, `go.mod`, `supabase/config.toml` and tell you what they
inferred. You get a working run immediately; the config is how you make it
accurate.

---

## Configure

`config/qa-config.reference.yml` documents every key. The part that matters is
`stack.layers` — a set of globs plus a one-line hint per layer:

```yaml
stack:
  layers:
    logic:
      globs: ["app/src/services/**/*.js"]
      hint: >-
        Every database query lives here — screens never query directly.
        Note .eq() / .filter() chains: they define which rows a user can
        see, and each one is a candidate negative or security scenario.
```

The globs say *where to look*. The hint says *what matters there* — it is
prose, read by a model, so write it the way you would brief a new tester.

Three worked profiles ship in `config/`:

| File | Stack | Also demonstrates |
|---|---|---|
| `example-react-supabase.yml` | React 19 + Vite · Supabase Postgres · Deno edge functions | agents living in the code repo; DB and logs from one MCP server |
| `example-java-spring.yml` | Java · Spring Boot · SQL database · message queue | agents in a **separate QA repo** (`repo_path: "../my-service"`); a non-`main` base branch |
| `example-python-fastapi-postgres.yml` | Python · FastAPI · SQLAlchemy + Alembic | no MCP at all — SQL through a CLI client, logs read from a file |

Other keys worth knowing:

- `project.base_branch` — what feature branches are diffed against
- `project.repo_path` — `"."` when the agents live in the code repo, or
  `"../service"` when they live in a separate QA repo
- `project.output_language` — any IETF tag; IDs, tags and code identifiers stay
  untranslated regardless
- `data.provider` / `logs.provider` — a **capability, not a product**: `mcp`
  (name the tool, the agent resolves its real prefixed name at runtime), `cli`
  (a command-line client), `file` (logs only), or `none`. Set `none` and
  `flow-verifier` still runs — it reports that channel as unchecked instead of
  guessing. Any MCP server that can run a query or search logs works; no agent
  is tied to a particular database or log platform.

---

## Use

```bash
# Both documents at once
@qa-pipeline branch=feature/PROJ-123-new-signup

# With free-form QA notes folded in
@qa-pipeline branch=feature/PROJ-123 notes=.claude/qa/notes/PROJ-123.md

# Individually
@checklist-generator branch=feature/PROJ-123
@guide-generator branch=feature/PROJ-123
@checklist-enricher branch=feature/PROJ-123 notes=.claude/qa/notes/PROJ-123.md

# After running a test by hand
@flow-verifier id=user-42 flow=F1-01          # one scenario
@flow-verifier id=user-42 flow=F1             # every F1-xx scenario
@flow-verifier traceId=abc123 flow=F2-03      # resolve entities from a trace first
@flow-verifier id=user-42 flow=F1-01 range=48 # widen the log window
```

Output lands in `{artifacts_dir}`:

```
.claude/qa/
├── checklists/checklist-<branch-slug>.md
├── guides/api-db-guide-<branch-slug>.md
└── notes/<your notes>.md
```

---

## Worked example

`examples/venue-persistence/` holds a full set of output for `readclub`, a
React + Postgres reading-groups app. The branch under test made venue selection
persistent and let signed-in members add venues to a catalogue everyone reads.

| File | What it shows |
|---|---|
| `checklist-venue-persistence.md` | 55 scenarios in 6 flows, prioritized, with regression scope and boundary cases |
| `api-db-guide-venue-persistence.md` | the tables, access rules and log statements that confirm each flow |
| `notes-venue-persistence.md` | sample notes, deliberately mixing verifiable observations with guesses |

Things worth pointing at in that output:

- The new `INSERT` permission on the shared `venues` table produced its own flow
  with five security scenarios — including the one for the member who *cannot*
  write, which is the case that is normally forgotten.
- The three-saved-venues cap produced boundary rows at 2, 3 and 4.
- Two flows are marked as having **no server-side evidence** — browser storage
  and canvas rendering. The guide says so explicitly, so `flow-verifier`
  returning INCONCLUSIVE there is understood as correct rather than broken.
- The guide notes that a row-level-security violation in the logs is a **PASS**
  for the scenario that tests a signed-out visitor being blocked. Evidence needs
  interpretation, and the guide is where that interpretation is recorded.
- The notes file contains a push-notification idea and a rate-limiting
  assumption that no code supports. The enricher reports them as skipped rather
  than writing scenarios for features that don't exist.

---

## Requirements

- Claude Code
- `git` — every diff is read through `git -C <repo>`
- For `flow-verifier` only: an MCP server for your database and/or logs. Without
  one, the first four agents work fully and `flow-verifier` reports its channels
  as unchecked.
