# AGENTS.md

Canonical policy for every coding agent in this repository — read it before making any change. It loads
automatically for both agents: **Codex** and **Claude Code** (v2.1.277+) both read `AGENTS.md` natively,
including nested `AGENTS.md` files in subdirectories. This is the single source of truth. If a
`CLAUDE.md` exists, it holds no policy of its own — it only imports this file (`@AGENTS.md`), because
Claude Code reads `CLAUDE.md` *instead of* `AGENTS.md` by default when both are present.

Most rules are universal. A few are tool-specific and marked **Claude Code:** or **Codex:** — follow
the one that applies to you, ignore the other.

> **Placeholder key** (delete this block once filled in):
> - `{{HIGH_STAKES_DOMAIN}}` — your money-path / high-blast-radius area (e.g. "order execution, auth,
>   payments"). Drives the Astra-vs-Sol routing split and the human-SHA gate.
> - `{{TASK_TRACKER}}` — where issues live (e.g. "GitHub Issues on org/repo", via the `gh` CLI).
> - `{{TEST_CMD}}` / `{{INTEGRATION_CMD}}` / `{{LINT_CMD}}` — the full-suite, integration-tier, and lint
>   commands the Engineer must run before opening a PR.
> - Model IDs below (`claude-opus-5-5[1m]`, `gpt-6-astra`, `gpt-6.1-sol`) are the current Claude/Codex
>   tiers — bump them if they've moved on. Pass full Codex IDs: the `codex` plugin doesn't resolve
>   `astra`/`sol` aliases.

Before modifying a subtree, read the nearest applicable nested `AGENTS.md`. A nested `AGENTS.md` may
impose **stricter** requirements than this file for its own subtree; where they conflict, the stricter
rule wins. Stop for human clarification before any destructive, irreversible, production, security, or
{{HIGH_STAKES_DOMAIN}} action.

## Task tracking

Task tracking is **{{TASK_TRACKER}}**. Classify each item by type (Bug/Feature/Task/Chore), priority,
risk, and an area label.

## Shell hygiene

Always use non-interactive flags for file/network operations so an aliased `-i` prompt can't hang an
agent (`cp -f`, `mv -f`, `rm -f`/`rm -rf`, `ssh -o BatchMode=yes`, `apt-get -y`, etc.).

Before any recursive deletion, resolve and inspect the target path. Never run `rm -rf` outside the
current worktree, against an empty or unverified variable-expanded path, or against a repository root
unless the task explicitly requires it and the target has been confirmed.

## Development Workflow

**Always use TDD for bug fixes and feature work: write a failing test first, then implement until it
passes.** Start from a red test that captures the bug/requirement, never from the implementation. Prove
discrimination by mutating the fix and watching the test go red. This is binding on **every**
implementer — Claude and Codex alike.

**Both vendors: use the `tdd` skill** (mattpocock/skills; install with
`npx skills add https://github.com/mattpocock/skills --skill tdd`, which installs it for Claude Code and
Codex alike). Test only at agreed **seams** (public interfaces, never internals), in **vertical
slices** (one test → minimal code → repeat, never all tests first), with expected values from an
independent source (no tautological tests), and mock only at system boundaries. Refactoring is not
part of the red → green loop. It comes back from review as findings. On dispatched work the seam
list comes from the Architect's design doc or the dispatch prompt. The full Engineer contract is
`agent-roles/engineer.md`.

## Git / Worktrees

Every agent that modifies repository files must work in its own isolated git worktree. Never edit from
the shared main checkout, and never allow two writing agents to share a worktree. Read-only
investigators and reviewers may use a clean checkout or a throwaway worktree. Treat the shared main
checkout as read-only.

Concurrent agents may rewrite shared branches and orphan commits, and sharing a worktree causes edits
to collide and leak into `main`'s working tree. Keep `main` clean except for deliberate controller
commits.

## PR Workflow

Standard delivery flow for fixes: **diagnose root cause → TDD fix → code review → CI green →
squash-merge → close the issue.** Don't skip steps — verify the root cause before designing the fix,
and close the associated issue once the PR ships.

Diagnose from tests, telemetry, logs, traces, and sanitized snapshots. A direct production-database
query **requires human approval**, and must be **read-only** with an explicit statement timeout, a max
row count, and explicitly selected columns (no `SELECT *`); redact results before they enter any
committed artifact.

### Models

The **implementer** model is routed by **work type**; the **reviewer** is always the **opposite
vendor** of the implementer (cross-vendor review), which makes the reviewer independent of the author
by construction. A reviewer must always be independent of the author and fresh every round.

**Implementer by work type** — route on the highest-risk component the diff touches. "High-stakes"
here means {{HIGH_STAKES_DOMAIN}} **backend logic** — inherently backend, so it routes to **Astra**.
Frontend/UI is **never** high-stakes for routing: a UI change is Opus even for a high-stakes-adjacent
feature; a diff spanning UI + high-stakes logic is **split** (UI → Opus, logic → Astra), or if
inseparable, the high-stakes logic governs → Astra.

| Work type | Implementer | Effort |
|---|---|---|
| Frontend / UI / UX, visual/design | **claude-opus-5-5[1m]** | `xhigh` |
| Backend, high-stakes — {{HIGH_STAKES_DOMAIN}}, logic-bearing migrations | **Codex `gpt-6-astra`** | `high` |
| Routine backend / coding — ordinary services, non-UI, CRUD, analytics/admin reads, glue | **Codex `gpt-6.1-sol`** | `high` |
| Boilerplate / mechanical — codegen, mass rename, config, deps, test scaffolding, 0-logic edits | **Codex `gpt-6.1-sol`** | `low` |

**Reviewer = the opposite vendor @ `high`** (mechanics in "Model routing & cross-vendor review"):

| Implemented by | Reviewer |
|---|---|
| Codex astra / sol (backend, routine, boilerplate) | **claude-opus-5-5[1m] @ `high`** — `xhigh` on the high-stakes path |
| claude-opus-5-5[1m] (frontend) | **Codex `gpt-6-astra` @ `high`** via `/codex:adversarial-review` |
| Boilerplate (sol @ `low`) | light Opus pass; **skip** when 0-logic |
| Docs / comments | none |

Planning / **Architect stays claude-opus-5-5[1m] at `xhigh`** — it is not the author, so a fresh
opposite-vendor reviewer of the implementation still applies. This routing governs **all**
implementation work; the staged pipeline below is its high-stakes instance, and lighter changes use the
same implementer/reviewer routing without the full Architect design doc.

#### Review axes

Each review round is split into separate **axes**, each run by its own fresh reviewer in parallel, so
that one axis can't mask another. Code that follows every standard can still implement the wrong
thing, and vice versa:

- **correctness**: bugs, regressions, security, resource handling, test quality.
- **spec**: missing or partial requirements, scope creep, wrong implementations, unjustified
  deviations from the design doc. Quote the spec line for each finding.
- **standards**: documented repo standards plus a fixed code-smell baseline, which are always
  MINOR judgement calls.

| Change | Axes |
|---|---|
| Staged pipeline (high-stakes) | correctness + spec + standards |
| Everything else with logic | correctness + spec (spec is skipped when there is no issue or spec) |
| Boilerplate (light pass) | correctness |

The full reviewer contract, shared by both vendors, is `agent-roles/reviewer.md`. Reviewers **report
everything they find and classify severity (BLOCKER / MAJOR / MINOR) in a separate pass after
discovery**. They have no confidence floor and no word cap. Models take a reporting floor literally
and under-report: they find real bugs, then suppress them for missing an arbitrary bar. On the
high-stakes path a suppressed finding is a shipped defect. Filtering is the adjudicating agent's job,
not the finder's. The repo's own `reviewer` subagent (`.claude/agents/reviewer.md`) has this built in.
If you ever review with a stock plugin agent instead (`feature-dev` or `pr-review-toolkit`
`code-reviewer`, which only report at confidence ≥ 80), override that floor in the dispatch prompt.
Don't edit those plugin files: they live outside this repo and are overwritten on update.

### Multi-agent delivery pipeline (high-stakes work)

For high-risk issues ({{HIGH_STAKES_DOMAIN}}), run each issue through a staged agent pipeline; stages
hand off via files/PRs, never shared state:

1. **Architect** (the `architect` subagent: Opus @ `xhigh`, read-only). It traces the code
   end-to-end, confirms or refutes the issue's hypothesized root cause, picks an approach over the
   alternatives, and returns a self-sufficient design doc: files and functions, **seams under test**,
   a TDD test plan, failure modes and out-of-scope. The orchestrator stores the doc with the issue in
   {{TASK_TRACKER}} and hands it to the Engineer verbatim.
2. **Engineer** (**Codex `gpt-6-astra`** for high-stakes backend — this pipeline's default; a high-risk
   frontend change uses the same cross-vendor routing, Opus-built and Astra-reviewed, but without this
   pipeline's human-SHA gate; own worktree) — follows `agent-roles/engineer.md`: implements the design
   with the `tdd` skill at the design's seams (red first, watch it fail; prove discrimination by
   mutating the fix and watching tests go red), runs `{{TEST_CMD}}` + `{{INTEGRATION_CMD}}` +
   `{{LINT_CMD}}`, opens a PR with `Closes #N`, and returns the evidence-backed Engineer report.
   Deviations from the design must be justified in the PR body.
3. **Principal reviewer** — the **opposite vendor** of the Engineer, fresh every round (no
   anchoring), over the FULL PR diff against base, on all three **review axes** in parallel. For the
   usual high-stakes case (Astra-implemented) that is three `reviewer` subagents (Opus, session at
   `xhigh`). For Opus-implemented work it is **Codex `gpt-6-astra`** (mechanics below). The reviewer
   **reports findings only** — it never edits code; the
   Engineer applies fixes and the loop repeats until every finding is **fixed or filed** as a scoped
   issue — not merely until the verdict reads `approve`. Classify findings BLOCKER/MAJOR/MINOR when
   recording. Expect this loop to earn its keep: successive fresh reviews routinely find distinct real
   blockers.
4. **Merge**: follow the **Merge checklist** below.

On the high-stakes path the Engineer is **Codex `gpt-6-astra`** and the reviewer is a fresh
**claude-opus-5-5[1m] @ `xhigh`** (cross-vendor by construction); these changes additionally need
**human approval on the final SHA** before merge. That is the only category requiring it — everything
else merges on the review above.

Everything outside this pipeline uses the same implementer/reviewer routing, scaled to the change: one
cross-vendor review at the `high` floor (docs, comments, and pure boilerplate need `low` or none).

#### Model routing & cross-vendor review (dispatch mechanics)

These are **Claude Code (orchestrator) dispatch mechanics** — how the agent driving the pipeline
invokes each implementer/reviewer. A dispatched Codex or Claude engineer/reviewer does not run these;
it just follows its role contract.

**Role definitions.** Each role's contract is a file in this repo, so dispatch prompts carry only the
inputs and never restate the rules:

| Role | Contract | Claude Code subagent |
|---|---|---|
| Architect | `.claude/agents/architect.md` (Claude only) | `architect`: read-only, Opus @ `xhigh` |
| Engineer | `agent-roles/engineer.md` (both vendors) | `engineer`: `isolation: worktree`, `tdd` skill preloaded, Opus @ `xhigh` |
| Reviewer | `agent-roles/reviewer.md` (both vendors) | `reviewer`: report-only, one axis per dispatch |

The subagents use the `opus` model alias, so a Claude model bump needs no edits there. Their
"read-only" and "report-only" limits remove the edit tools, but `Bash` can still write. The rest of
the guarantee is the instruction in the contract, so check reviewers' worktrees are clean.

**Codex models (implementation)** run through the installed `codex` plugin's app-server runtime (no
`codex exec` stdin gotchas). Write a prompt file that carries only the inputs:

```
Role: Engineer. Read agent-roles/engineer.md and follow it exactly; AGENTS.md also applies.
Issue: #<N> <title>
Branch / worktree: <branch> in <worktree path>
Seams under test: <from the design doc, or listed here for lighter changes>
Design doc:
<the Architect's design doc verbatim, or "none (non-pipeline change)">
```

Then dispatch it in the Engineer's isolated worktree (resolve the current plugin version — do not
hardcode it; the cache dir changes on plugin update):

```
CODEX_ROOT=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
perl -e 'print STDERR "codex pgid=$$\n"; setpgrp(0,0); exec @ARGV' -- \
  node "$CODEX_ROOT/scripts/codex-companion.mjs" task \
  --model {gpt-6-astra|gpt-6.1-sol} --effort {high|low} --write --prompt-file <prompt.txt>
```

The `perl` wrapper puts the run in its own process group and prints the group ID (see **Process
hygiene**). It works on Linux and macOS alike; `setsid` doesn't ship with macOS.

`--write` is **required** to edit files — without it the sandbox is read-only and the task can only
inspect ("workspace is mounted read-only"). Full-suite runs and self-commit also need network in the
sandbox: set `[sandbox_workspace_write] network_access = true` in `~/.codex/config.toml` (otherwise
loopback-binding tests fail and the orchestrator must run the full gate + commit outside the sandbox).
Treat early Astra/Sol runs as **probationary**. Check the Engineer report against the diff: the red
output must come from the assertion, the discrimination proof must name a real mutation, and the
gates must have actually run (a gate reported `not run` is the orchestrator's to run).

**claude-opus-5-5[1m] (implementation)** — frontend/UI work runs as the `engineer` subagent in its own
worktree. Its dispatch prompt carries the same inputs as the Codex prompt file above. Note that
`isolation: worktree` branches from the default branch, not from the orchestrator's HEAD.

**Reviewers** are always the opposite vendor, fresh every round, over the FULL PR diff, one fresh
reviewer per **review axis**, dispatched in parallel:
- **claude-opus-5-5[1m] reviews Codex (backend/routine/boilerplate) work:** one `reviewer` subagent
  per axis, all in the same message so they run concurrently. Each dispatch prompt names
  `Axis: <correctness|spec|standards>`, `Base: origin/main`, `HEAD: <SHA>` and the spec location
  (design doc / issue).
- **Codex astra reviews Opus (frontend) work:**
  - correctness: `/codex:adversarial-review --model gpt-6-astra --base origin/main --scope branch
    [focus]` (`--background` for large diffs, `--wait` otherwise). It is already adversarial and
    reports every finding with a confidence, so it needs no confidence-floor override.
  - spec and standards: a **read-only** companion `task` per axis (the command above **without**
    `--write`, `--model gpt-6-astra --effort high`), with a prompt file of `Role: Reviewer. Read
    agent-roles/reviewer.md and follow it exactly.` plus the axis, base, HEAD SHA and spec.
  - Codex review is strongest on code correctness, not visual/UX taste. Pair it with a Claude visual
    pass or a screenshot check for UI diffs.

Effort mechanics: a Codex `task` takes `--effort` per call (`none`…`xhigh`), but
`/codex:adversarial-review` inherits `model_reasoning_effort` from `~/.codex/config.toml` (not
per-call). Claude subagents accept an `effort` frontmatter field, which overrides the session level.
`architect` and `engineer` pin `xhigh`. `reviewer` deliberately has no `effort` field and inherits
the session's, so set the orchestrating session (`/effort`) to `high`, or `xhigh` on the high-stakes
path, before dispatching reviewers. Both vendors' reviewers **report findings only** — they never
edit.
Post the review verbatim as the PR `### Code review` comment, with one sub-section per axis and the
adjudication of each finding. Put the reviewed SHA in the body (a `Reviewed-SHA: <HEAD_SHA>` /
`Reviewer: <model>` trailer per axis) so the merge checklist's "comment cites HEAD_SHA" check passes.

**Process hygiene.** Long Codex runs can outlive the Bash call that started them: a Bash tool timeout
kills the shell, but not necessarily the companion and its `codex app-server` child.
- Start companion `task` runs with the Bash tool's `run_in_background`, never in the foreground with a
  timeout. Waiting is the harness's job; killing a run by accident is not.
- Note the `codex pgid=` line the wrapper prints. If a run is aborted or abandoned, kill the whole
  group (`kill -TERM -- -<pgid>`), not just the PID you started.
- End every Codex dispatch, reviews included, by collecting or cancelling its jobs with the plugin's
  job commands (`/codex:status`, `/codex:result`, `/codex:cancel`). Never leave a `--background`
  review running.
- Never `pkill -f codex`. It also kills other sessions' runs.
- Processes that tests start are the Engineer's to tear down (`agent-roles/engineer.md`). Any it
  reports under "Left running" are the orchestrator's to stop.

**Once per session**, before relying on either vendor:
- Confirm both vendors are usable (Codex logged in, astra/sol respond).
- Check for leftover Codex processes from earlier sessions:
  `ps -eo pid,pgid,etime,command | grep -E '[c]odex-companion|[c]odex app-server'`. Report any to the
  human instead of killing them: another live session may own them.

**Why not an MCP server for Codex.** Codex is dispatched through the plugin companion, not through an
MCP wrapper (a third-party one, or Codex's own `codex mcp-server`). An MCP tool call blocks until it
returns and is bounded by the client's tool timeout. That fits long TDD runs badly, and a timed-out
call is exactly what leaves a Codex child orphaned. The third-party wrappers reviewed so far also drop
per-call `--effort` and adversarial review. Revisit this if `codex mcp-server` gains asynchronous jobs
and per-call effort.

#### Review-finding adjudication

A strong adversarial reviewer keeps surfacing real defects, including **pre-existing** and
**out-of-scope** ones — that is a feature, not noise. Deciding which findings gate *this* PR is the
**adjudicating agent's** job (the orchestrator), never the reviewer's. After each review round,
classify every finding:

- **In-scope regression the diff introduces** → fix, then re-review.
- **Out-of-scope / pre-existing / adjacent / scope-creep** → file an issue (area + priority label),
  link it in the `### Code review` comment; it does **not** block this PR.
- **Minor / nit on changed lines** → fix if trivial; otherwise file a low-priority issue.
- **Standards-axis smells** are judgement calls → fix, file, or record a one-line reason for rejecting
  them in the review comment.
- **Spec-axis scope creep** → remove it, or justify it in the PR body and in the review comment.

The review loop exits when every finding is **fixed or filed** — not merely when the verdict reads
`approve`. Merging over a `needs-attention` verdict is allowed **only** when every residual finding has
a filed issue and the adjudication (each finding → fixed / deferred-to-`#N` / false-positive) is
recorded in the review comment. Every deferral issue must exist **before** merge — a `Closes #N` that
silently drops a finding is the exact failure this prevents.

**Anti-abuse:** "out of scope" means genuinely unrelated to the diff's purpose, not a real regression
rationalized away to clear the gate. When in doubt, treat it as in-scope, or ask the human. On the
high-stakes path, human approval on the final SHA is the backstop against mis-scoping a real blocker as
pre-existing.

#### Merge checklist

Run these in order. If any step fails, stop. Don't merge around it. A repo may wrap this checklist in
its own `/merge` skill, but the checklist is the policy.

1. **Review record posted.** The PR has a `### Code review` comment with every axis's report, the
   adjudication of each finding (fixed / deferred-to-`#N` / false-positive / rejected judgement call),
   and a `Reviewed-SHA:` trailer.
2. **No lost findings.** Every deferred finding (MINOR, pre-existing, out-of-scope) has an issue that
   already exists and is linked from the review comment.
3. **No SHA drift.** The PR's current head SHA equals the `Reviewed-SHA`. Any commit after the review,
   including a rebase that changed content, needs a fresh review round. That round is not optional.
4. **CI green** on that head SHA.
5. **Human approval on that SHA**, on the high-stakes path only.
6. **Squash-merge.** `Closes #N` closes the issue. If the PR ships only part of the issue, the
   follow-up issue must already exist (see the operational rules below).
7. **Clean up.** Remove the worktree, delete the branch, fast-forward the local `main`.

## Landing the Plane (Session Completion)

**When ending a work session, work is NOT complete until `git push` succeeds.**

1. **File issues for remaining work** — before you declare done.
2. **Run quality gates** (if code changed) — tests, linters, builds.
3. **Update issue status** — close finished work, update in-progress items.
4. **Sync and push** — rebase onto the base branch first; a branch behind `origin/main` has not been
   verified against what it will merge into:
   ```bash
   git pull --rebase
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** — remove the worktree once merged, prune remote branches, and collect or cancel any
   Codex jobs this session started, so no Codex process outlives it (see **Process hygiene**).
6. **Verify** — all changes committed AND pushed.
7. **Hand off** — provide context for the next session.

**Critical rules:** NEVER stop before pushing. NEVER say "ready to push when you are" — YOU push. If
push fails, resolve and retry until it succeeds. Never push to a protected or shared branch unless the
workflow explicitly authorizes it.

## Operational rules learned the hard way

- Merging PRs sequentially moves `origin/main` — every in-flight engineer must rebase before its next
  push.
- Concurrent agents sharing a test database must use unique fixture IDs and per-ID cleanup, never
  `TRUNCATE`.
- A PR that defers part of its issue's scope must file the follow-up issue BEFORE merge (a `Closes #N`
  on a partially-shipped issue silently orphans the rest).
