# Board: {{BATCH_TITLE}}

Orchestrator-owned (only the orchestrator edits this file). How to use it: `PROTOCOL.md`. Started {{DATE}}. Owner:
{{USER}}.

## Sessions

| Role | Session name (`ListAgents`) | Worktree | State |
| --- | --- | --- | --- |
| Orchestrator | `orchestrator` | `{{ROOT}}` | running |
| Agent {{ID}} | — | `worktrees/<repo>-{{ID}}` | not started |

## Batch state

**RUNNING** since {{DATE}}. (`PAUSED` / `RUNNING` / `CLOSED`, with the date and the resume instructions when paused.)
`develop` at intake: `{{DEVELOP_HASH}}`.

## Status summary

Detail lives in `agents/<id>.md`; this is the orchestrator's roll-up.

| Ticket / package | Spec (ready / partial / none, mode) | State | Last commit | Next step |
| --- | --- | --- | --- |

Locks: see `locks/` (empty = free).

## Answers

Answers to agents' questions (and the user's intake answers), keyed by question id.

| Id | Answer | Date |
| --- | --- | --- |

## Scope

| Ticket | In scope | Out / deferred |
| --- | --- | --- |

## Work order

1. …

## Branches and worktrees

Worktrees live in `{{ROOT}}/worktrees/<repo>-<ticket>/`.

| Branch | Repo(s) | Starts from | Worktree |
| --- | --- | --- | --- |

## MR plan

- …

## Decision log

| Date | Decision |
| --- | --- |
