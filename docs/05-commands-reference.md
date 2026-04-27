# Commands Reference

Terse one-section-per-command reference. For the lifecycle and examples, see [02-ticket-workflow.md](02-ticket-workflow.md). For delegation flow, see [03-delegation.md](03-delegation.md). For the Plane MCP wiring, see [13-plane-integration.md](13-plane-integration.md).

All commands live in `commands/` in the repo and are symlinked into `~/.claude/commands/` by `install.sh`. They're Claude Code user-level slash commands, available in every project on every machine. Copilot mirrors live in `copilot-prompts/` and are regenerated from the Claude sources by `bin/sync-copilot-prompts` (also wired into `install.sh`).

## Dual-world dispatch

Every command except `/ticket-install` begins with a `Pre-flight: detect backend` block:

- `.claude/plane-config.md` present → **Plane backend** (drive Plane via MCP; identifiers like `SMOKE-12`)
- Else `.claude/ticket-config.md` + `tickets/` → **Markdown backend** (file-based; identifiers like `TKT-012`)
- Else → error, tell the user to run `/ticket-install`

`install.sh` smoke-tests that every ported command carries this block. The descriptions below use Plane vocabulary by default; the markdown backend's equivalent state name is in parentheses where relevant.

### State equivalence

| Plane state | Markdown status | Meaning |
|---|---|---|
| Backlog | open | created; not yet investigated |
| Ready | proposed | investigated; awaiting approval |
| In Progress | in-progress | approved; being implemented |
| In Review | review | implementation complete; awaiting human |
| Done | shipped | merged (and deployed if applicable) |
| Cancelled + `deferred:` comment | deferred (→ `tickets/deferred/`) | parked; may revisit |
| Cancelled + `wontfix:` comment | wontfix (→ `tickets/wontfix/`) | closed without shipping |

## `/ticket-install`

**Purpose**: Bootstrap a project (new or existing) to use the ticket workflow.

**Usage**: `/ticket-install`

**Pre-flight**: Detect current state. If `.claude/plane-config.md` exists → Plane update mode. If `.claude/ticket-config.md` exists (markdown) → ask: migrate to Plane, or update markdown config. Fresh install → ask which backend (defaults to Plane if the `plane` MCP server is reachable).

**Plane path**: Picks or creates a Plane project via `mcp__plane__list_projects` / `create_project`. Seeds the 6 standard states (renames `Todo` → `Ready`, creates `In Review`, sequence-pins after create). Seeds workspace-standard labels (`plan-ticket`, `stub`, `delegated`, `risk:*`). Creates per-project `app:<profile>` labels from detected preview profiles. Writes `.claude/plane-config.md` (UUIDs) + `.claude/ticket-config.md` (stack, commands, preview profiles; no `tickets/` directory).

**Markdown path**: Detects stack and preview profiles, writes `tickets/TEMPLATE.md` + `.claude/ticket-config.md`, appends a `## Tickets` section to `CLAUDE.md`.

**Preconditions**: None. May prompt to rename `master` → `main`.

**Side effects**: Creates config files. Plane path: creates/modifies states and labels in the target Plane project. Markdown path: creates `tickets/` and `tickets/TEMPLATE.md`.

## `/ticket-new "title"`

**Purpose**: Create a new ticket.

**Usage**: `/ticket-new [--no-parent | --parent SMOKE-N] "short description"`

**Plane**: Calls `mcp__plane__create_work_item` with state `Backlog`. Auto-links to the parent plan ticket if the current branch carries an identifier like `feat/SMOKE-12-slug` (looks up the parent via `retrieve_work_item_by_identifier`, uses its `parent` field as the new item's parent). `--no-parent` forces standalone; `--parent SMOKE-N` links to a specific parent (fails loud on 404).

**Markdown**: Copies `tickets/TEMPLATE.md` to `tickets/TKT-NNN.md`, fills frontmatter + Description + Acceptance Criteria, status `open`.

**Pasted images**: If you paste an image into the `/tn` prompt, the model uses it to generate the ticket and writes a *Visual context* block of factual prose into the description. The binary image itself is **not** uploaded — Plane's MCP exposes no attachment tool, and the markdown ticket file is text-only. The prose is the only thing downstream agents see, so paste any image you want represented before submitting (the model can't go back and look at it later).

**Side effects**: Creates one work item / file. Does not investigate.

## `/ticket-list`

**Purpose**: Show active tickets (optionally all).

**Usage**: `/ticket-list [--all]`

**Plane**: `mcp__plane__list_work_items` with client-side filter to active state groups (backlog / unstarted / started). `--all` includes terminal (completed / cancelled). Grouped output.

**Markdown**: Reads `{tickets-dir}/TKT-*.md`, groups by status; `--all` recurses into `shipped/`, `deferred/`, `wontfix/`.

**Side effects**: None — read-only.

## `/ticket-investigate SMOKE-N [SMOKE-M ...] [--in-given-order]`

**Purpose**: Explore the codebase and write a plan. Multi-ticket also produces a recommended implementation order.

**Single-ticket**:
- Verifies status is Backlog (open). If later, notes "already investigated" and stops.
- Reads `CLAUDE.md`, `.claude/ticket-config.md`, context docs, and (for Plane) the parent plan ticket's description if the `parent` field is non-null.
- Deep-dives relevant code; writes Investigation + Proposed Solution + Implementation Plan.
  - Plane: appends to `description_html` via `update_work_item`.
  - Markdown: fills the corresponding sections in the ticket file.
- Applies `risk:{low|medium|high}` label.
- Plane: posts `[investigated_at: <sha>]` comment; transitions state → Ready.
- Markdown: sets `regression_risk:` frontmatter; status → proposed.

**Multi-ticket**: investigates each sequentially; prints a one-line summary per ticket. After all complete, scores by declared-dep → risk → shared-file → quick-win → ID and outputs recommended order with copy-pasteable `/ta` / `/tch` commands. `--in-given-order` skips the ordering pass.

**Side effects**: Modifies work item / ticket file. Read-only on source. No branch creation.

## `/ticket-approve SMOKE-N`

**Purpose**: Create a feature branch and implement the plan.

- Verifies state is Ready (proposed) and an Implementation Plan is present.
- Verifies clean working tree.
- Creates branch `ticket/{lowercased-id}-{slug}` from main.
- Transitions state → In Progress (Plane) / status → in-progress (Markdown).
- Works through the Implementation Plan item by item.
- Runs test + build from `.claude/ticket-config.md`.
- Commits with message `{ID}: {title}`.
- Transitions state → In Review (Plane) / status → review (Markdown). Writes Files Changed, Test Report.

**Side effects**: Creates git branch, edits source, creates commits. Does not push.

## `/ticket-review SMOKE-N`

**Purpose**: Generate the human verification checklist.

- Verifies state is In Review (review).
- Verifies the feature branch is checked out.
- Reads the diff against main.
- Runs automated checks (tests / build / lint / typecheck / rebase status) and records pass/fail into an Automated Checks section.
- Writes a Verification Checklist (for human) with Setup / Core Functionality / Edge Cases / Regression Checks.

**Side effects**: Updates work item / ticket. Runs test/build/lint. No source changes.

## `/ticket-ship SMOKE-N [--target BRANCH]`

**Purpose**: Rebase, test, merge, (optionally) deploy.

- Verifies state is In Review and feature branch has commits.
- Fetches, rebases onto main (or `--target`); stops on conflicts.
- Runs tests + build post-rebase.
- Checks out main, pulls, merges `--no-ff`.
- Runs tests + build once more on main (reset on failure).
- Pushes main.
- If Deploy is configured, runs it.
- Deletes the feature branch.
- Transitions state → Done (Plane) / moves file to `tickets/shipped/` (Markdown). Plane: posts a PR link comment.

**Side effects**: The one command that modifies remote state. Never force-pushes.

## `/ticket-defer SMOKE-N {reason}` / `/ticket-close SMOKE-N {reason}`

**Purpose**: Cancel a ticket. `defer` = parked, may revisit; `close` = wontfix (dup / invalid / obsolete).

**Plane**: Transitions state → Cancelled. Posts comment `deferred: {reason}` or `wontfix: {reason}`.

**Markdown**: Status → deferred / wontfix. `git mv` the file to the corresponding subfolder.

**Side effects**: Terminal transition. `/ticket-reopen` brings it back.

## `/ticket-reopen SMOKE-N [reason]`

**Purpose**: Bring a Cancelled or Done ticket back to active.

- Cancelled + `deferred:` + has Implementation Plan → Ready (resume).
- Cancelled + `deferred:` + no plan → Backlog.
- Cancelled + `wontfix:` → Backlog (investigation stale).
- Done → Backlog (regression — old investigation stale).
- Posts `reopened: {reason}` comment (Plane) / appends to the ticket file (Markdown).

**Side effects**: State/status transition. Does not create branches.

## `/ticket-status [SMOKE-N]`

**Purpose**: Show lifecycle timeline of a ticket, or the active set with no arg.

**Plane**: `retrieve_work_item_by_identifier` + `list_work_item_activities` + `list_work_item_comments`. Reconstructs phases from comment markers.

**Markdown**: Reads frontmatter + Delegation Log; reconstructs from ticket file.

**Side effects**: None — read-only.

## `/ticket-preview SMOKE-N`

**Purpose**: Launch a preview of a ticket's feature branch without shipping.

- State must be In Progress or In Review.
- Reads the ticket's `app:<profile>` label (Plane) / `app:` field (Markdown) and looks up the preview profile in `.claude/ticket-config.md`.
- Computes port = `Preview port base + numeric-id + offset`.
- Launches the profile's command in the feature branch's worktree. Writes `.worktrees/ticket-{id}/preview.pid` + `preview.meta`.
- Plane: posts `[preview] <url>` comment.

**Side effects**: Runs a local process. `/ticket-cleanup` tears it down.

## `/ticket-batch [SMOKE-N ...] [--mode=auto|rollup|individual] [--no-preview]`

**Purpose**: Run investigate → implement → preview on multiple tickets in parallel worktrees.

- Resolve set: explicit IDs or Backlog-state items with no arg.
- Pre-implement conflict check: parse Implementation Plan file paths; warn on overlap.
- Phase 1: investigate all Backlog items (parallel subagents in Claude Code; sequential loop in Copilot).
- Phase 2: auto-approve; cut per-ticket worktrees under `.worktrees/ticket-{id}/`.
- Phase 3: implement in parallel.
- Phase 4: preview per mode (`rollup` combines all branches; `individual` per ticket; `auto` tries rollup then falls back).
- Prowl once when ready.

**Side effects**: Worktrees, feature branches, per-ticket state transitions, preview processes.

## `/ticket-chain [SMOKE-N ...] [--dry-run|--sequential|--ship]`

**Purpose**: Queue up work and walk away. Parallel investigation → dependency graph → wave execution → preview + consolidated review checklist.

Extends `/ticket-batch` with a topological dependency graph built from (a) declared deps in Implementation Plan prose and (b) Plane's native `work_item_relations` (`blocked_by` as hard edge, `relates_to` as soft overlap). Groups into waves; parallel within a wave, sequential across. `--ship` merges each ticket after its implementation instead of stopping at preview.

> **Copilot note:** the Copilot mirror is single-ticket only. For multi-ticket, use `/ticket-batch` in Copilot or `/ticket-chain` in Claude Code.

## `/ticket-delegate SMOKE-N [SMOKE-M ...] [phase]`

**Purpose**: Hand off a ticket to another agent (full lifecycle or a specific phase).

- `/ticket-delegate 5` — full lifecycle, single ticket
- `/ticket-delegate 10 11 12 13` — batch (asks parallel vs. sequential)
- `/ticket-delegate SMOKE-5 investigate` / `implement` / `verify`

Reads the appropriate brief template from `~/.claude/brief-templates/{phase}.md`, fills in placeholders, writes brief files (Markdown only — Plane stores the brief as a comment on the work item). Applies `delegated` label and posts `[delegated_to: <agent>]` comment. For full/implement, creates the feature branch (and worktree for parallel batch).

**Side effects**: Brief files (Markdown) / work-item comment (Plane), label, state transition.

## `/ticket-collect SMOKE-N [SMOKE-M ...]`

**Purpose**: Collect and review work returned from delegated tickets.

- Verifies `delegated` label (Plane) or status `delegated` (Markdown).
- Reads the `[delegated_to: ...]` comment (or Delegation Log) to determine phase.
- For `full` delegation: Claude acts as code reviewer — reads investigation, reviews diff, checks tests, verifies AC. Writes a Delegation Review verdict.
- For `implement`: reads the feature-branch diff, fills Files Changed + Test Report.
- Removes `delegated` label and `[delegated_to: ...]` comment.
- Transitions state to the appropriate next phase (usually In Review).

**Side effects**: Work item / ticket updates. No source changes. For batch: generates consolidated review checklist.

## `/ticket-promote SMOKE-N [SMOKE-M ...] | --all`

**Purpose**: Promote stub tickets (label `stub`) to the active set (remove the label, send to Backlog).

Used after `/brainstorm` or `/plan-new` to graduate specific stubs into real work. `--all` promotes every stub.

**Side effects**: Removes `stub` label / status change.

## `/ticket-cleanup [SMOKE-N | --all]`

**Purpose**: Reap stale worktrees and preview processes.

- Lists all `.worktrees/ticket-*/` directories.
- For each, resolves the ticket. If state is terminal (Done / Cancelled) or the ticket no longer exists → reap the worktree and kill any running preview PID.
- `--all` reaps every stale worktree.

**Side effects**: Deletes worktree directories, kills preview processes. Idempotent.

## `/plan-new "brain dump"`

**Purpose**: Extract principles + candidate tickets from a brain dump; create a plan ticket with children.

- Creates a work item with label `plan-ticket` whose `description_html` contains the principles.
- Creates child work items for each candidate, linked via `parent`.
- Posts open-questions-for-user as a comment on the plan ticket.

Plane only. Markdown projects use `/brainstorm` instead.

## `/plan-verify SMOKE-N`

**Purpose**: Audit a plan ticket after its children have shipped — compare the principles in the description against the actual diffs of the Done children, write a judgment comment.

- Gate: target must carry `plan-ticket` label and have a principles heading in its description.
- Fetches Done children + their PR links.
- Diffs against main; judges principles per-diff.
- Posts one Principles Judgment comment per run (idempotent within the same SHA).

Plane only.

## `/op-scaffold <path-to-plan.md>`

**Purpose**: Scaffold a multi-plan operation from a master plan markdown file.

**Usage**: `/op-scaffold docs/proposals/<operation-slug>-plan.md`

**Pre-flight**: Reads the plan file end-to-end and validates it against the inlined plan-format spec (see Step 1 of the command file). Required sections: `# Operation: <slug>`, `## Why this exists`, `## Dispatch order` (multi-plan path) OR top-level briefs (flat single-plan path), per-plan sections with Goal + Briefs table + Briefs-detail, and `## What done looks like` at the bottom (which MUST contain an executable `bash` block, not prose only — the scaffolder prepends those commands to every brief's verification block). Validates the dispatch graph (no cycles, all "Depends on" entries resolve).

**Validation rules** (hard rejects):
1. Required sections present.
2. `## What done looks like` contains an executable `bash` block.
3. No verification command contains placeholder markers (`<...>`, `# replace`, `# TODO`, `<your-`, `<path-to-`).
4. UI-touching briefs have either an automated UI verification in acceptance OR a tracker ticket reference + manual smoke step (escape hatch).

**Validation rules** (soft warnings — proceed only after human confirms):
- A file appears in two or more briefs' "Files touched" — confirm cross-brief invariants live on the last brief to touch the file.
- A brief whose title/goal signals deletion (remove/delete/rip out/cleanup/retire/drop/eliminate) has only grep-style acceptance with no behavior test.

**Action**: Expands to `docs/operations/<slug>/`:
- `00-master.md` — verbatim copy of the plan
- `README.md` — generated overview + dispatch graph + run instructions
- `operation-state.json` — initial pending state (each brief has a `residuals` array)
- `HANDOFF.md` — placeholder
- `VERIFY.md` — placeholder (manual-eyeball checklist filled at finalization)
- `plan-<ID>-<slug>/00-conductor.md` — generated per-plan task-lead brief; "Plan-level verification" block prepends the op-level command set
- `plan-<ID>-<slug>/NN-<brief-slug>.md` — one file per brief; the brief's Verification block has the op-level command set auto-prepended verbatim

**Side effects**: Creates the operation folder under the project's `docs/operations/`. Does not run anything; does not commit.

**Plan generation**: To convert a brain-dump into a plan-format master plan, copy the body of `~/.claude/operation-templates/META_PROMPT_FOR_PLAN_OPUS.md` into a fresh Opus session along with your dump.

## `/op-run <path-to-operation-folder>`

**Purpose**: Run a scaffolded operation autonomously in the current session.

**Usage**: `/op-run docs/operations/<slug>/`

**Pre-flight**:
1. Verifies the folder is a valid scaffold (00-master.md / README.md / operation-state.json / VERIFY.md / at least one 00-conductor.md).
2. Checks working tree (uncommitted files outside the operation's "Files touched" lists block the run).
3. Runs baseline using the op-level command set from `00-master.md`'s `## What done looks like` block (NOT a hardcoded `npm run typecheck && npm test`); known-baseline-failures listed in 00-master.md are tracked by name.
4. **Resume detection**: if the state JSON shows a previous attempt, prompts the user with `yes/no/abort` to clean up dirty partial-write files and reset state. Special case: if `blocker.reason == "residuals-undisposed"`, prints the undisposed residuals and exits without resetting state — the user dispositions them by editing operation-state.json, then re-runs.
5. Confirms with the user (operation summary + count of plans/briefs + branch + baseline result). For operations >40 briefs, fires an extra "long operation may exhaust context" warning requiring `proceed-anyway`.

**Action**: The main session plays conductor + task-lead inline (the harness strips `Agent` from spawned subagents — see operation-conductor.md / operation-task-lead.md frontmatter warnings for why). For each plan in dispatch order (parallel where the table permits), dispatches `operation-worker` (Sonnet) subagents per brief in parallel batches per the conductor brief, verifies each worker's output (diff + every command in the brief's Verification block, top to bottom — including the auto-prepended op-level set), dispositions any residuals the worker reported (fold-in / follow-up / accept-with-justification), commits approved work with prefix `<operation-slug> [plan-<ID>] brief NN: ...`, runs plan-level checks, advances. After every plan completes, runs operation-level "What done looks like" checks at HEAD.

**Rework loop**: 3 attempts per brief. After the third failure, marks the plan blocked, sets the top-level blocker, prowls priority=1, and exits.

**Residual-completion gate**: Before finalization, walks every brief's `residuals` array. If ANY residual lacks a disposition, sets `blocker.reason: residuals-undisposed`, prowls priority=1, and exits without writing HANDOFF.md or VERIFY.md.

**Finalize**: Writes `VERIFY.md` (one section per UI-touching brief: what landed in user-facing terms, where to look, what to confirm — or the empty-state line if no UI changed), writes `HANDOFF.md` (timestamps, commits, dispatches, rework summary, per-plan landed sections, verification output run at HEAD, deviations, commits-in-order, "Known limitations" from accept-with-justification residuals, "Follow-up tickets opened" from follow-up residuals, link to VERIFY.md, next steps), updates `operation-state.json` with `completed_at` + `final_verification`, sends a Prowl notification (key extracted from `~/.claude/CLAUDE.md` at runtime; never hardcoded).

**Side effects**: Commits to the current branch. Does NOT push, merge, or deploy. Session must stay open for the duration; closing halts the run. The human walks `VERIFY.md` offline before merging — failed eyeball checks become new tickets, not a re-open of the operation.

## Where to look if a command isn't doing what you expect

Each command's behavior is defined in `commands/{command-name}.md`. Because the file is symlinked into `~/.claude/commands/`:

```bash
cat ~/.claude/commands/ticket-investigate.md
# or
cat ~/src/claude-config/commands/ticket-investigate.md
```

Claude Code reads these files verbatim. If you want to tweak behavior for a specific command, edit the file in the repo, commit, push, pull on other machines. No re-install needed for Claude Code (commands are symlinked — edits are live). For Copilot mirrors, re-run `install.sh` so `bin/sync-copilot-prompts` regenerates the Copilot prompt.
