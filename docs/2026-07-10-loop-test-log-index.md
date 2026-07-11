# Loop test log index

This repo was used as a disposable test bed for the small-project Loop protocol.

The detailed chat logs are stored locally, not in git:

`/Users/burhanhalilov/code/testing/local-chat-logs/2026-07-10-loop-test/`

That folder is intentionally ignored by git because it is local debugging evidence, not repo product.

## What was tested

- GitHub Project status flow: `New -> Todo -> In Progress -> Done`.
- `Agent` Project field values: `Codex`, `Claude`, `Cursor`.
- Codex tasks acting as:
  - watcher,
  - coordinator,
  - worker 1,
  - worker 2,
  - reviewer,
  - architect.
- Scheduled Codex watcher as a lightweight doorbell.
- Cross-task nudges from watcher/coordinator to worker chats.
- GitHub issues/comments/PRs as durable coordination state.

## What happened

The test proved that Codex task-to-task nudges work, but the role boundaries were too fuzzy.

Main problems:

1. The watcher became more than a watcher.
   It started needing lane occupancy, PR state, issue comments, project fields, and task idleness.

2. The coordinator contract changed mid-test.
   At first coordinator moved issues to `In Progress`; later we decided workers should do that when accepting work.

3. Worker prompts became stale.
   Worker 1 and worker 2 were not updated at the same time as watcher/coordinator, so they followed older assumptions.

4. PR creation failed for a good reason.
   Worker 1 tried to open a draft PR from a branch with no real commit/diff, so GitHub would not create the PR.

5. AI was being asked to make too many state-machine decisions.
   The more the watcher/coordinator had to infer, the less KISS the loop became.

## Current reset state

Fresh active issues in Project #3:

| issue | title | status | agent |
|---|---|---|---|
| `#9` | Add generic README context for Loop watcher tests | New | Codex |
| `#10` | Add fake example app config | New | Codex |
| `#11` | Add tiny smoke-check script | New | Codex |
| `#12` | Add watcher test notes template | New | Codex |

Old issues `#5`-`#8` are obsolete history.

No open PRs.

Watcher automation is paused.

## Local log files

Local-only folder:

`local-chat-logs/2026-07-10-loop-test/`

Files:

- `README.md` — local log map and task ids.
- `role-chat-excerpts.md` — important excerpts from the Testing project role chats.
- `scheduled-watcher-runs.md` — scheduled watcher behavior and lesson.
- `github-state.md` — human-readable GitHub state snapshot.
- `project-items.json` — raw Project item snapshot.
- `issues.json` — raw issue snapshot.
- `prs.json` — raw PR snapshot.

## Practical lesson

The simplest stable direction is:

- GitHub remains the durable source of truth.
- Chat remains the doorbell.
- AI workers should get one concrete issue and do one bounded job.
- Broad board scanning and lane decisions should either be extremely small or become deterministic code.

