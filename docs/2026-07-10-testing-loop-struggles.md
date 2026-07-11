# Testing Loop Struggles — 2026-07-10

This note records what went wrong while testing Loop with the `bhalilov/testing`
repo and Codex tasks. It is intentionally detailed because the failures were
not one bug. They were role-boundary bugs, state-source bugs, and tool-boundary
bugs that compounded.

## Starting Goal

The goal was to test a reusable small-project AI coding loop:

1. Owner approves work in GitHub.
2. A lightweight watcher notices work.
3. A coordinator routes one issue to one worker lane.
4. A worker claims one issue, creates a branch, commits, and opens a draft PR.
5. A reviewer gates the PR before owner review.

The test bed was:

- local repo: `/Users/burhanhalilov/code/testing`
- GitHub repo: `bhalilov/testing`
- GitHub Project: Project #3
- Codex tasks: watcher, coordinator, reviewer, worker 1, worker 2, architect

The design goal was KISS: GitHub is the durable state; chats are only nudges and
reasoning rooms.

## Clean Concepts That Survived

These still feel right:

- GitHub issue is the work order.
- GitHub Project status is the visible workflow.
- `Agent` field chooses the platform: `Codex`, `Claude`, or `Cursor`.
- Chat messages are doorbells, not authority.
- Each GitHub comment made by a role should identify the role and task id.
- Workers should derive work from GitHub, not from the nudge text.
- Workers build one issue in one branch and open one draft PR.
- Reviewer reviews PR artifacts; owner still decides merge.
- Architect is upstream only and speaks about spec clarity, not approval.

## Where We Got Tangled

### 1. Watcher and coordinator both became decision makers

At first, watcher was supposed to be a cheap scheduled doorbell. But as the test
failed, we kept adding logic to it:

- detect `Todo + Agent=Codex`
- infer free worker lanes
- detect assigned-but-unclaimed work
- decide whether to nudge coordinator, worker, or reviewer
- decide whether one claimed issue should block other Todo issues

That made watcher a second coordinator. Then the coordinator also had lane logic.
Two roles were trying to decide "what next?" and their prompts drifted.

Lesson: watcher should either be extremely dumb, or the Python/script layer
should do deterministic state detection. Do not create two AI decision makers.

### 2. Coordinator role became too vague

Coordinator was asked to:

- read all issues
- infer status
- know worker availability
- select issues
- select lanes
- write GitHub comments
- sometimes move status
- sometimes nudge workers
- sometimes route PRs to reviewer

That is too much for one fuzzy AI role. It behaved like a project manager, but
without a precise state machine. It reported "dispatchable" work instead of
dispatching, then later moved status but failed to assign a worker, then later
assigned a worker after correction.

Lesson: if coordinator remains AI, its job must be one very small decision at a
time. Better yet, deterministic logic should decide routing, and AI should only
do bounded tasks.

### 3. We changed the meaning of `In Progress` mid-test

Initial protocol said:

- coordinator moves `Todo -> In Progress`
- worker claims the `In Progress` issue

Then we realized a better meaning:

- coordinator assigns a worker lane while issue remains `Todo`
- worker accepts by moving `Todo -> In Progress`

This is cleaner because `In Progress` should mean "a worker actually accepted",
not "coordinator tried to dispatch".

But after changing the rule, worker prompts still said "claim only In Progress".
So worker 2 would not claim issue #6, because #6 was correctly still `Todo`.

Lesson: when a state meaning changes, every role prompt and automation rule must
change together. Partial migration creates deadlocks.

### 4. Worker prompts were too old for the new protocol

Workers originally knew:

- claim only `In Progress + Agent=Codex`

The corrected rule is:

- claim `Todo` or `In Progress` if coordinator dispatch comment selects your lane
- if claiming from `Todo`, worker moves it to `In Progress`

This correction was sent later, but the messy middle showed why role reset is
needed after protocol changes.

Lesson: workers need a compact, current claim contract. Old prompt history is
dangerous when protocols evolve.

### 5. PR creation failed because the branch had no diff

Worker 1 claimed issue #5 and pushed `worker-1-issue-5`, but could not open a
draft PR because the branch had no commits beyond `main`.

That was a real GitHub constraint, not a Codex bug.

Correct worker flow:

1. Claim issue.
2. Create/switch branch.
3. Make the smallest useful commit for the issue.
4. Push branch.
5. Open draft PR.

PR preflight should include:

- branch has at least one real commit/diff versus `origin/main`
- branch is pushed
- there is no existing open PR for the branch
- PR body links the issue with `Closes #<issue-number>`
- verification ran, or the worker clearly reports why it could not

Lesson: "open draft PR early" still means "after a real first commit exists".

### 6. Codex task-status and nudge plumbing is app-only

The Python idea looked attractive: let a local script scan GitHub and drive the
loop cheaply. But Python cannot reliably:

- read whether a Codex task is busy
- send app-native messages to an existing Codex task
- confirm the task received the nudge

Only Codex app tools can do that.

Lesson: local Python can own GitHub state scanning, but a Codex watcher task is
still needed as the app bridge unless Codex exposes a stable local task API.

### 7. GitHub-only flags each had tradeoffs

We considered ways for watcher to know whether coordinator is needed:

- Project field such as `Needs coordinator`
- label such as `needs-coordinator`
- comment marker such as `COORDINATOR NEEDED`
- local snapshot compare

Project fields felt finicky because every role must set and clear them.
Labels are visible and simple, but still require set/clear discipline.
Comment markers are easy to write but awkward to clear.
Snapshot compare is ceremony-free but invisible and can miss stale work.

Lesson: the simplest no-label option is a local snapshot watcher: nudge
coordinator whenever GitHub state changes. Add a stale timer later if needed.

### 8. "No nudge needed" was sometimes false

Watcher said "No nudge needed" while:

- issue #6 was `Todo`
- worker 2 was free
- or issue #6 was assigned to worker 2 but not claimed

This happened because watcher had too much half-correct logic and stale role
rules. It saw one lane occupied and treated the whole loop as occupied, then
later saw assignment and failed to nudge the selected worker.

Lesson: watcher either needs no routing logic, or routing must be a deterministic
script with explicit tests.

### 9. Scheduled watcher produced UI artifacts

Watcher sometimes emitted `::inbox-item{...}`. That is app/UI metadata, not part
of the Loop protocol and not useful as GitHub state.

Lesson: watcher output should be plain text only. Durable state belongs in
GitHub or the local watcher memory file, not UI directives.

### 10. Architect blurred "spec clear" with "good to go"

Architect correctly understood it did not dispatch, but said issue #5 was "good
to go". That phrase sounded like approval/build dispatch.

Correct architect vocabulary:

- `SPEC CLEAR`
- `SPEC NEEDS SHAPING`
- `OWNER QUESTION`

Architect should not say:

- `good to go`
- `approved`
- `ready to build`

Lesson: architect only speaks about spec clarity. Owner approves. Coordinator or
controller routes. Worker builds.

## Current Direction After the Struggle

The emerging better architecture is:

| Layer | Job |
|---|---|
| local watcher/controller script | deterministic GitHub state scan and next-step suggestions |
| Codex watcher task | app bridge that can read task status and send nudges |
| coordinator | either removed, or reduced to a tiny bounded lane-assignment task |
| worker | given one issue and goal; builds it |
| reviewer | given one PR and goal; reviews it |
| architect | given one idea; shapes a spec |

The key insight: AI should not be a fuzzy global scheduler. AI does well with
one bounded task. Deterministic logic should handle mechanical state transitions.

## Candidate Simplest Future Protocol

### Option A: no labels, local snapshot watcher

1. Python reads Project items, issues, comments, and PRs.
2. Python compares to `.loop/state.json`.
3. If GitHub changed, Python emits a nudge plan.
4. Codex watcher reads the plan and sends messages to idle Codex tasks.
5. AI workers/reviewer perform bounded tasks.

Pros:

- no labels
- no custom attention field
- no manual flag ceremony
- cheap and deterministic

Cons:

- a stuck unchanged state may not trigger without a stale timer
- local state file is invisible in GitHub
- still needs Codex watcher as app bridge

### Option B: one `needs-coordinator` label

1. Any role can add `needs-coordinator`.
2. Watcher sees the label and nudges coordinator.
3. Coordinator clears the label when handled.

Pros:

- visible in GitHub
- easy to search
- explicit manual override

Cons:

- agents must remember to set and clear it
- stale labels cause repeated nudges
- missing labels cause missed work

Owner preference after discussion leaned away from labels because they feel
finicky.

## Clean Reset Applied

The messy test issues were closed:

- #5
- #6
- #7
- #8

Fresh issues were created:

- #9 Add generic README context for Loop watcher tests
- #10 Add fake example app config
- #11 Add tiny smoke-check script
- #12 Add watcher test notes template

All fresh issues were added to Project #3 as:

- `Status = New`
- `Agent = Codex`

The old half-started branch `worker-1-issue-5` was deleted locally and remotely.

Watcher automation was paused before retry.

## Clean Role Reset Sent

All roles were told to forget the messy middle state for #5-#8 except as
obsolete history:

- watcher
- coordinator
- reviewer
- architect
- worker 1
- worker 2

Workers were also given PR preflight rules so the empty-branch PR failure does
not repeat.

## Open Design Question

The remaining design question is whether the AI coordinator should exist.

Current suspicion:

- a human-readable "coordinator role" is useful in docs,
- but the actual coordinator behavior may be better as deterministic local
  logic plus direct worker/reviewer nudges,
- because AI coordinator became too vague and kept needing more instructions.

The next design pass should decide:

1. Keep coordinator as a tiny AI lane assigner.
2. Replace coordinator with a deterministic local controller.
3. Keep a coordinator chat only for unusual blockers, not normal dispatch.

## Most Important Lesson

Do not use AI for global orchestration if the state machine can be written down.

Use AI for:

- shaping one spec,
- building one issue,
- reviewing one PR,
- explaining one blocker.

Use deterministic GitHub state for:

- what changed,
- what is assigned,
- who is claimed,
- whether a PR exists,
- what should be nudged next.

