---
name: batch-orchestrator
description: Use when several related Jira tickets/features should be worked at the same time by parallel Claude sessions (one agent session per ticket, each in its own worktrees) and this session should coordinate them — from a bare list of Jira keys (intake, specs via legacy-migration when missing, setup of the shared blackboard folder, worktrees and start prompts) through running the batch (relaying agents' questions to the user, recording answers, unblocking agents, locks) to closing it (final branch, one MR per repo).
---

# Batch Orchestrator

## Overview

Runs a batch of related tickets with one Claude session per ticket ("agents") and one coordinating session (the
**orchestrator**, the session running this skill). Sessions communicate through a **blackboard**: a folder of markdown
files where each file has exactly one writer. Cross-session messages (`SendMessage`) are only nudges that point at the
files; the files are the source of truth.

```
<ROOT>/batch-<slug>/
  PROTOCOL.md       rules every session follows (from templates/PROTOCOL.md)
  BOARD.md          orchestrator-only: state, sessions, status, answers, scope, order, branches, MR plan, decision log
  agents/<id>.md    one per agent, written only by that agent: state, questions, done, notes for other agents
  locks/            folder locks for shared resources (integration tests, review, heavy local runs)
  prompts/          start prompts the user pastes into each agent session
```

The user stays the decision-maker. The orchestrator does not implement tickets: it plans the batch with the user, then
reads, relays, records and unblocks. It may write specs itself in the spec phase (see Phase 2).

**Standing rule for every session:** whenever a decision is needed (deviation from legacy, scope, naming, design choice,
anything a spec marks as open) or something genuinely blocks, stop and ask. Never pick a default and continue. The
orchestrator never answers a question for the user.

## When to use

- Two or more tickets of one domain built in parallel, with shared files or dependencies between them.
- The user wants to stay the decision-maker while several sessions work.

Not for a single ticket (use `legacy-migration` / the normal flow), and not for fan-out inside one task (use subagents).

## How a teammate starts it

1. Open a Claude Code session **at the project root** (`<ROOT>`, the folder that holds the repos and the root
   `CLAUDE.md`), not inside a repo or worktree.
2. Paste the prompt from `templates/orchestrator-prompt.md`, with the ticket keys filled in.
3. `/rename orchestrator` (only the user can rename a session).

Everything else is driven from that session. Nothing needs to exist beforehand except the repos and the tools below.

## Project profile (CampusCE defaults)

The templates use these values. On another project, change this block and the matching lines in
`templates/PROTOCOL.md`; nothing else is project-specific.

| Key | CampusCE value |
| --- | --- |
| `ROOT` | the folder holding `cce-vnext-backend`, `cce-vnext-frontend`, `baselinesnapshot` and the root `CLAUDE.md` |
| Repos | backend `cce-vnext-backend` (.NET), frontend `cce-vnext-frontend` (Angular, `pnpm`, `nx`) |
| Jira | project `CCE03` (scope every JQL with `project = CCE03`) |
| Specs | `docs/specs/` in the backend repo; a ticket is ready to build when its packages have a `tasks.md` |
| Spec workflow | `legacy-migration` skill (backend repo `.claude/skills/`), then the speckit skills it calls |
| Execution | `superpowers:executing-plans` inline + `superpowers:test-driven-development`; never subagent-driven |
| Commit prefix | `[AI] CCE03-<package> - <summary>` |
| Git network | the user does all `fetch`/`push`/`pull`; sessions only work locally |
| Shared test DB | `devsqlz1` / `sce` (integration tests); reset daily ~17:00 → `RESET_WINDOW` 16:30–17:30 |
| Seed scripts | run by the user with `sqlcmd … -U <login> -I -b -i <script>` (password prompted, never on the command line) |
| Branch names | `feature/CCE03-<n>_<slug>`; final branch `feature/CCE03-<a>-<b>-…_<domain-slug>` |
| MR plan | one MR per repo for the whole batch, unless a ticket ships on its own (then its own MR) |

The root `CLAUDE.md` rules (schema freeze, credentials, legacy wording, CSS verified in a browser, restart the API before
browser re-tests, MR threads discussed before fixing) apply to every session and win over this skill.

## Prerequisites (check in Phase 0; stop and tell the user what is missing)

- All sessions on the **same machine** (`SendMessage` / `ListAgents` reach local sessions only).
- Tools answer: the Jira MCP (read one ticket), CodeGraph on `baselinesnapshot` (`codegraph_explore`), dbgraph (one proc
  lookup), the `superpowers` plugin, the `legacy-migration` skill.
- The local `develop` (or `origin/develop`) is recent: ask the user to fetch if unsure. Record its hash in `BOARD.md`.
- Enough memory for several sessions (each agent runs builds and tests): note free memory; with < 16 GB free plan
  fewer parallel agents.

## Phase 0: Start

1. Check the prerequisites above.
2. Pick the batch `SLUG` and title with the user; create `<ROOT>/batch-<slug>/` with `PROTOCOL.md` (from
   `templates/PROTOCOL.md`, placeholders filled), `BOARD.md` (from `templates/BOARD.md`), `agents/_template.md` (from
   `templates/agent.md`), empty `locks/` and `prompts/`.
3. Add an "Active batch" section to the root `CLAUDE.md` pointing to `PROTOCOL.md` and `BOARD.md`, and a memory entry,
   so every new session and every compaction finds the batch.

## Phase 1: Intake (one question round with the user)

For each Jira key given:

1. Read the Feature with its subtasks (the BE/FE packages) and linked tickets. List packages, status, assignee.
2. Look for its specs: `docs/specs/*<key or package key>*/tasks.md` in the backend repo, and a `*-legacy-inventory.md`
   for the domain. Classify each ticket:
   - **ready**: every in-scope package has a `tasks.md`;
   - **partial**: inventory or spec exists, no `tasks.md`;
   - **none**: nothing yet.
3. Map prerequisites (delegate to an Explore subagent for many packages). For each package: what its `T000` (or
   equivalent) expects, owner ticket, present on `develop` or not, verdict (start now / after package X / blocked
   outside the batch), and the shared "whichever lands first" files it would create. Also check:
   - `develop` features that already implement or redesign what the spec builds (merged after the spec was written:
     `develop`'s version wins);
   - architecture tests on `develop` the spec would trip (e.g. single-implementation rules);
   - a spec's "defaults applied without clarification" / OPEN items: they are questions, collect them.
4. Present one summary to the user and wait:
   - per ticket: readiness, packages in / deferred (and why), dependencies;
   - proposed work order, branch per ticket and its base, final branch, MR plan;
   - spec phase mode for tickets that are not ready (see Phase 2; default: orchestrator);
   - every open question found, each with options and a recommendation.
5. Record the answers in `BOARD.md` (Scope, Work order, Branches, MR plan, Decision log with the date).

## Phase 2: Spec phase (only for tickets that are not ready)

Specs are produced with the `legacy-migration` skill, **one inventory/spec/plan per domain** (that skill's rule), not one
per package. Two modes; the user picks per batch or per domain in Phase 1:

| Mode | Who runs `legacy-migration` | Good for |
| --- | --- | --- |
| **orchestrator** (default) | this session, domain by domain, before starting the build agents | one consistent view across tickets; the user reviews every spec in one place |
| **agent** | the ticket's/domain's agent session, first thing after it starts, then it moves on to building | many or large domains; specs written in parallel; the agent keeps the context it built |

Either way:

- The spec work happens in a worktree on the ticket's branch (create it as in Phase 3), and is committed there.
- Deviation questions (rows marked **deviation** or ambiguous) go to the user in one grouped round per domain, with
  options and a recommendation. Record answers in the inventory and in `BOARD.md` → Answers.
- The spec is ready when `tasks.md` exists and `speckit-analyze` has no CRITICAL/HIGH findings left (fixed or ruled by
  the user). Mark the ticket **ready** in `BOARD.md`.
- In orchestrator mode, watch context: write each domain's decisions to `BOARD.md` as you go so a compaction loses
  nothing; delegate legacy tracing to Explore subagents when it gets large.
- In agent mode, use the spec variant of the agent prompt (`templates/agent-prompt.md`, "Spec first" block).

## Phase 3: Setup per agent

For each ticket whose first package is unblocked (others wait; record what they wait on):

1. **Worktrees**: `git -C <main checkout> worktree add -b <branch> <ROOT>/worktrees/<repo>-<id> <base>` for each repo it
   touches. Base = `develop`, or the branch that creates the shared files it needs (record that in `BOARD.md`).
2. **`agents/<id>.md`** from `agents/_template.md`: worktrees, branch, packages in order, deferred items, what it waits
   on, resume point.
3. **`prompts/agent-<id>.md`** from `templates/agent-prompt.md`: fill the placeholders; put known fixes in (wrong `T000`
   paths, "reuse X from develop, don't create", answers already given that affect it). Drop lines that don't apply.
4. Update `BOARD.md` → Sessions and Status summary.
5. Tell the user: open a new session at `<ROOT>`, paste `prompts/agent-<id>.md`, then `/rename agent-<id>`.
   Agents nudge the orchestrator when they start.

## Phase 4: Running the batch

- **Messages in**: an agent nudges when it starts, is blocked, finishes a package, or creates something others depend
  on. On each nudge read its `agents/<id>.md`; update Sessions / Status summary in `BOARD.md`.
- **Questions**: present them to the user grouped (`AskUserQuestion` works well: up to 4 per call), each with options
  and a recommendation. Verify cheap claims first (grep `develop`, read the legacy method or proc it quotes) and say
  what you did not verify. Several agents asking the same thing → one question, one answer for all.
- **Answers**: write the answer in `BOARD.md` → Answers (same id) and, if lasting, the Decision log. Then nudge:
  `Answer to Q-<id>-<n> in BOARD.md: <one line>. Resume <task>.`
- **Agent rulings made without a question** (an agent's "rulings for the user to see"): present them before the
  ticket's MR; deviations from legacy among them need explicit approval and a dated note in the spec.
- **User actions** (seed scripts, fetch, push, API restart): give the exact command, say what it touches (read the
  script first: data only? idempotent?), record it as done in `BOARD.md` when the user confirms.
- **Cross-agent dependencies**: when an agent records a commit others need (a contract commit, a shared file, a
  finished prerequisite ticket), record the next step in `BOARD.md` and nudge the waiting agent, or create the next
  agent (Phase 3).
- **After every fetch by the user**: check what landed on `develop` against every open ticket (shared readers,
  catalogs, engines, endpoints). Ask once for all affected agents; prefer moving onto `develop`'s piece after the batch
  rebase over merging `develop` mid-package.
- **Rules that change mid-batch** go in `PROTOCOL.md` and the Decision log; nudge every agent once.
- **Locks**: never break a lock yourself; a stale one (> 2 h, or the owner says it is not running) is the user's call.
  After a killed run, check whether the run left shared state behind (tenant settings, seeded rows) before the next run.
- **Watch the machine**: when a run is killed for memory, check free memory and the process list; propose fewer
  parallel heavy runs rather than retrying blindly.
- **Pause / resume**: on "pause", nudge every agent to stop at a safe point (commit or record uncommitted work and its
  resume point in its file, release locks), then set `BOARD.md` → Batch state to `PAUSED`. Sessions can be closed; reopen
  them with `/resume` (or `claude --resume` from `<ROOT>`), or paste the prompt again. On "resume": check `locks/` and
  each agent file, set Batch state to `RUNNING`, nudge each agent with its resume point.

## Phase 5: Closing the batch

1. A ticket that ships on its own (its own MR): when done, the user fetches; the agent rebases onto `develop`, re-runs
   build, unit and its integration tests; the user pushes and opens the MR. Other agents merge its branch locally
   before the packages that need it.
2. When every ticket is done: merge the per-ticket branches locally into the final branch, rebase onto the fetched
   `develop`, review `git diff develop` on shared files for duplicates (a clean merge can still duplicate code), apply
   moves recorded as "after the batch rebase", run every affected suite (one integration run at a time).
3. Hand the user one MR per repo (backend first when the frontend depends on its endpoints). The user pushes.
4. Remove the root `CLAUDE.md` "Active batch" section and the memory pointer; archive the batch folder. Add the
   batch's new lessons to the table below.

## Lessons

| Lesson | Why |
| --- | --- |
| Open agent sessions at the project root; work in worktrees via absolute paths and `git -C` | Memory and `.claude/settings.local.json` are keyed to the folder a session opens in; a session opened in a worktree loads neither |
| One writer per file; Edit, never Write, on shared files | No write locks needed; Edit fails on stale text instead of overwriting |
| Locks are folders (`New-Item -ItemType Directory` fails if it exists) | Check-and-create in one step; a file-exists check then create races |
| Only the owner releases a lock, after reading `owner.md`; acquire, run and release as separate steps | A one-line acquire-run-cleanup script whose acquire failed deleted another agent's lock |
| `heavy.lock` for full suites and prod builds, besides the integration lock | Four agents running suites at once exhausted 32 GB; Claude Code killed an FE full check and an integration run |
| Locks cover this machine only | Teammates still write to the shared test DB: tests must not depend on table counts or max ids |
| Integration-test reset window (no run starts 16:30–17:30 for `sce`) | The tenant is reset daily around 17:00; a run overlapping it fails for reasons unrelated to the code |
| A killed integration run skips its sweep | Tenant-wide settings a test changed (e.g. `TimeOffSet`) stay changed for every other suite: check and restore before the next run |
| Before "restoring" shared test-DB state, find who changed it and what the value was after the last reset | The orchestrator ordered a restore to yesterday's value; the post-reset value was different. Test helpers must update settings in place and restore the same row, never delete + reinsert |
| Spec "defaults applied without clarification" are open questions | Agents stopped on every unconfirmed OPEN item; confirm them when starting a package, in one batch |
| Check `develop`'s architecture tests when mapping prerequisites | A spec written before CCE03-1130 ported tag substitution as `.Replace` chains; `SingleImplementationTests` failed only at the package's final full check |
| A shared piece merged mid-batch (CCE03-1264 settings reader) affects every agent | Ask once for all agents; move after the batch rebase rather than merging `develop` mid-package |
| Agents report rulings they made without asking | Collect them before the MR; several were deviations from legacy that needed approval |
| Performance criteria as round-trip counts, not wall-clock | Network latency to the shared DB alone exceeded a 200 ms budget |
| A stored proc on the live test DB can be newer than the snapshot a spec traced | The fresh read wins; correct the inventory/spec in place with a dated note |
| A progress file's name is not evidence of how an agent works | An "SDD ledger" file raised a false subagent alarm; ask the agent before escalating |
| `/rename` is user-only | Record the auto-generated session name until the user renames it |
