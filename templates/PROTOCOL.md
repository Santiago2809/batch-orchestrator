# Protocol: {{BATCH_TITLE}} (blackboard)

How the sessions of this batch coordinate. Read this once at session start; read `BOARD.md` at every checkpoint.

Lives outside the repos ({{ROOT}}): local only, never committed.

## Roles

| Role | Session | Writes | Reads |
| --- | --- | --- | --- |
| **Orchestrator** | the user's coordinating session | `BOARD.md` only | everything |
| **Agent `<id>`** | one session per ticket, opened at the project root ({{ROOT}}), working only in its worktree(s) via absolute paths and `git -C` | `agents/<id>.md` only | `PROTOCOL.md`, `BOARD.md`, other agents' files |
| **User** | — | answers through the orchestrator | everything |

**Single writer per file.** Never edit a file you don't own. If you need something in another file, say so in your own
file (Questions or Notes) and nudge the orchestrator.

Shared files are changed with **Edit only, never Write** (Edit fails if the text changed since you read it; Write
silently overwrites). If an Edit fails, re-read the file and redo the edit.

## Files

```
batch-{{SLUG}}/
  PROTOCOL.md       this file (orchestrator-owned, rarely changes)
  BOARD.md          orchestrator-owned: scope, order, branches, MR plan, decision log, answers, status summary, sessions
  agents/<id>.md    agent-owned: status, current task, last commit, blockers, questions, notes for other agents
  locks/            shared-resource locks (directories)
  prompts/          start prompts for agent sessions
```

## Checkpoints (when an agent reads `BOARD.md`)

- At session start and after every compaction.
- Before starting each task.
- After each commit (then update your own file: task, commit hash).
- Before stopping for any reason.
- When nudged by the orchestrator.

## Questions and decisions

The standing rule: whenever a decision is needed (deviation from legacy, scope, naming, design choice, anything a
skill/spec marks as a question) or something genuinely blocks, **stop**. Never pick a default and continue; never start
the next step "in the meantime".

1. Agent adds a row under **Questions** in its own file: id `Q-<id>-<n>` (e.g. `Q-123-3`), the question, options,
   recommendation, status `open`. Sets its **State** to `blocked: Q-<id>-<n>`.
2. Agent nudges the orchestrator (`SendMessage`, see below) and stops.
3. Orchestrator presents the question to the user, records the answer in `BOARD.md` → **Answers** (same id) and, if it
   is a lasting decision, in the **Decision log**.
4. Orchestrator nudges the agent. Agent reads the answer, marks its question `answered`, continues.

An agent never treats a question as answered until the answer is in `BOARD.md`.

## Nudges (`SendMessage`)

Files are the source of truth; a message is only a pointer to them. Keep it to one line, e.g.
`Q-123-3 open in agents/123.md` or `Answer to Q-123-3 in BOARD.md`.

- Find session names with `ListAgents`; the **Sessions** table in `BOARD.md` maps roles to names.
- Agent → orchestrator: when blocked, when a package is done, when a lock is stale, when another agent's work affects
  yours.
- Orchestrator → agent: when an answer is recorded, or `BOARD.md` changed something the agent depends on.
- Never put a decision or its rationale only in a message.

## Locks (shared resources only)

Coordination needs no lock (single writer per file). Locks exist only for resources that cannot be shared:

| Lock | Resource |
| --- | --- |
| `locks/integration.lock` | integration-test run (shared `devsqlz1`/`sce` and test login) |
| `locks/review.lock` | `/simplify` and `/code-review` (they re-run builds and tests) |
| `locks/heavy.lock` | heavy local runs that exhaust memory when several overlap: a full frontend suite, a production build, a full backend unit suite. Run a big check in pieces, one piece per hold. One integration run and one heavy run may overlap; two heavy runs may not |

Acquire atomically. Creating a directory fails if it already exists, so check-and-create is one step:

```powershell
New-Item -ItemType Directory "{{ROOT}}\batch-{{SLUG}}\locks\integration.lock" -ErrorAction Stop
```

- Success: write `owner.md` inside it (agent id, what is running, start time) and note the lock under **State** in your
  file.
- Failure: someone holds it. Read its `owner.md`, keep working on tasks that need no database, retry later.
- Release: `Remove-Item -Recurse` the lock directory as soon as the run ends (pass or fail); update your file.
- **Only the owner releases a lock, and only after reading `owner.md` and seeing its own id there.** Never chain
  acquire, run and release in one script with an unconditional cleanup: if the acquire fails, the cleanup deletes
  someone else's lock. Acquire, run and release as separate steps.
- **After a run is killed** (memory, timeout): release your lock, then check whether the run left shared state behind
  (tenant-wide settings a test changed, seeded rows its sweep would have removed); record it in your file and tell the
  orchestrator before anyone runs again.
- **Reset window**: do not start an integration run between **{{RESET_WINDOW}}** local time (`sce` is reset daily
  around 17:00; default window 16:30–17:30). Wait until 17:30; seeded rows (e.g. template catalogs)
  may need re-seeding after the reset. The lock is local to this machine: teammates can still write to `sce`, so
  tests must not depend on whole-table counts or max ids.
- **Stale** (older than 2 hours, or the owner says it is not running): do not break it yourself; nudge the
  orchestrator, who asks the user.

## Agent file template

Agent files follow `agents/_template.md`.

## Standing rules (all sessions)

- No `git fetch` / `push` / `pull` in either repo. Local commits, branches, rebases onto the already-fetched `develop`
  are fine.
- Commits: `[AI] CCE03-<package> - <summary>` (package key, e.g. `CCE03-797`), one commit per task.
- Execution: `/superpowers:executing-plans`, one task at a time, test first (`superpowers:test-driven-development`).
  Not subagent-driven. Each package ends with build/lint/test gates, then `/simplify`, then `/code-review` (under
  `locks/review.lock`).
- Rulings made without a question go under Notes → "Rulings for the user" in your file; they are reviewed before
  the MR.
- Pause: when the orchestrator says pause, stop at a safe point (commit, or record uncommitted work and the exact
  resume point in your file), release your locks, set State to `paused`.
- Root `CLAUDE.md` rules apply (schema freeze, credentials, legacy wording, restart API before browser re-test, verify
  CSS in a browser, MR threads discussed before fixing).
