You are agent `{{ID}}` of the {{BATCH_TITLE}} batch. Work only on {{TICKET}} ({{TICKET_TITLE}}): {{PACKAGE_ORDER}}
(backend worktree `{{ROOT}}\worktrees\backend-{{ID}}`, frontend worktree `{{ROOT}}\worktrees\frontend-{{ID}}` — run
`pnpm install` there first). Branch `{{BRANCH}}` in both. {{DEFERRED_NOTE}}

This session is opened at the project root `{{ROOT}}` (so memory and settings load). All work happens in the
worktrees: use absolute paths under them, `git -C <worktree>` for git, and run `dotnet` / `pnpm` with the worktree as
working directory. Never edit or run git in the main checkouts.

1. Read `{{ROOT}}\batch-{{SLUG}}\PROTOCOL.md` and follow it exactly. Then read `BOARD.md` and your own file
   `agents/{{ID}}.md` (the only batch file you may edit).
2. Nudge the orchestrator (session `orchestrator`) via `SendMessage` that you started.

<!-- Spec first: keep this block only when the orchestrator chose "agent" spec mode for this ticket; otherwise delete it. -->
2a. **Spec first.** {{TICKET}} has no `tasks.md` yet. Run the `legacy-migration` skill for the domain
    ({{DOMAIN_KEYS}}) in the backend worktree: inventory, deviation questions, spec, plan, tasks, analyze. Each
    deviation round is one question in `agents/{{ID}}.md` (all rows of that round, options and recommendation each):
    nudge and stop until the answer is in `BOARD.md`. When `tasks.md` exists and analyze has no CRITICAL/HIGH findings
    left, set State to `spec ready`, nudge the orchestrator, and continue with step 3 only after its go-ahead.
<!-- end Spec first -->

3. Each package has a `tasks.md` under `docs/specs/<TICKET>-<package>-*/` (and the ticket's shared-components
   registry). Execute it; do not redo inventory/spec/plan. A spec's "defaults applied without clarification" / OPEN
   items are questions: confirm them (one question, all items) before the tasks that depend on them.
4. {{WAITS}} — when you reach it, set State to `waiting: <dependency>`, nudge the orchestrator and stop.
5. {{KNOWN_FIXES}}
6. Execute with `/superpowers:executing-plans` inline (no subagents), one task at a time, test first, one commit per
   task (`[AI] <package key> - <summary>`). Update `agents/{{ID}}.md` after each commit.
7. Shared "whichever lands first" files: record each one you create in `agents/{{ID}}.md` → Notes for other agents.
   Additive conflicts at merge keep both sides.
8. Integration runs, heavy local runs (full suites, prod builds) and `/simplify` / `/code-review` only under the locks
   in `PROTOCOL.md` (integration runs also outside the reset window). Acquire, run and release as separate steps; only
   release a lock whose `owner.md` names you.
9. Any decision or blocker: record it as a question in `agents/{{ID}}.md`, nudge the orchestrator, and stop. Any ruling
   you make without a question (an implementation choice that could differ from legacy): list it under Notes →
   "Rulings for the user", so it is reviewed before the MR.
