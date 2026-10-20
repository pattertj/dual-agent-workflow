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
| claude-opus-5-5[1m] (frontend) | **Codex `gpt-6-astra` @ `high`** via companion `adversarial-review` |
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

| Change | Axes (round 1) |
|---|---|
| Staged pipeline (high-stakes) | correctness + spec + standards |
| Everything else with logic | correctness + spec (spec is skipped when there is no issue or spec) |
| Boilerplate (light pass) | correctness |

**Round 1 reviews the full change; every later round reviews only the delta.** Round 1 covers the
whole PR diff against base. From round 2 on, a review covers only what changed since the previous
round's `Reviewed-SHA`. The reviewer first checks that each finding marked fixed really is fixed,
then looks for defects the delta introduces, including breakage it causes in code it didn't touch.
Delta rounds run **correctness**, plus every other axis that had a finding fixed in the previous
round.

The trade-off is deliberate. Delta rounds are faster, cheaper and stay focused on the fixes. But a
defect that round 1 missed in unchanged code won't be caught by a later round. Round 1 has to be the
thorough one.

If the branch was rebased or rewritten, the previous SHA is no longer an ancestor of HEAD. The delta
is then the `git range-diff` of the PR's own commits (see `agent-roles/reviewer.md`). A
`range-diff` doesn't show how the rebased code interacts with the new upstream commits. CI on the
new HEAD is the check for that.

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
   anchoring). Round 1 covers the FULL PR diff against base on all three **review axes** in
   parallel. Later rounds cover only the delta since the previous `Reviewed-SHA` (see **Review
   axes**). For the
   usual high-stakes case (Astra-implemented) that is three `reviewer` subagents (Opus, session at
   `xhigh`). For Opus-implemented work it is **Codex `gpt-6-astra`** (mechanics below). The reviewer
   **reports findings only** — it never edits code; the
   Engineer applies fixes and the loop repeats until every finding is **fixed or filed** as a scoped
   issue — not merely until the verdict reads `approve`. Classify findings BLOCKER/MAJOR/MINOR when
   recording. Fix rounds routinely introduce real blockers of their own, so verify every delta.
   Round 1 is the only full pass, so dispatch it at full strength.
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
node "$CODEX_ROOT/scripts/codex-companion.mjs" task --cwd <worktree> \
  --model {gpt-6-astra|gpt-6.1-sol} --effort {high|low} --write --prompt-file <prompt.txt>
```

**Always pass `--cwd <worktree>`** to every companion call (`task`, `adversarial-review`, `status`,
`result`, `cancel`). The plugin keys its jobs and its app-server broker by git workspace root. A call
without `--cwd` runs against the orchestrator's checkout, not the Engineer's worktree.

The orchestrator calls the companion script directly. The plugin's `/codex:*` slash commands are
marked `disable-model-invocation`, so only a human can run them.

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

**Reviewers** are always the opposite vendor, fresh every round, one fresh reviewer per **review
axis**, dispatched in parallel. Round 1 is `Mode: full` over the whole PR diff. Later rounds are
`Mode: delta` from the previous round's `Reviewed-SHA`, and the prompt carries that round's
adjudicated findings for the axis:
- **claude-opus-5-5[1m] reviews Codex (backend/routine/boilerplate) work:** one `reviewer` subagent
  per axis, all in the same message so they run concurrently. Each dispatch prompt names
  `Worktree: <path>`, `Axis: <correctness|spec|standards>`, `Mode: <full|delta>`,
  `Base: origin/main`, `HEAD: <SHA>`, the spec location (design doc / issue) and, in delta mode,
  `Previous: <Reviewed-SHA>` plus the prior findings. The subagent starts in the orchestrator's
  checkout, so the `Worktree:` line is what points it at the code under review.
- **Codex astra reviews Opus (frontend) work:**
  - correctness: `node "$CODEX_ROOT/scripts/codex-companion.mjs" adversarial-review --cwd <worktree>
    --model gpt-6-astra --base <ref> --scope branch [focus]`, run with the Bash tool's
    `run_in_background`. `<ref>` is `origin/main` in round 1 and the previous `Reviewed-SHA` in delta
    rounds. In delta rounds, put the prior findings to verify in the
    focus text. If the branch was rebased (the previous SHA isn't an ancestor of HEAD),
    `--base <prev>` would sweep in upstream commits. Run correctness as a read-only `task` under
    `agent-roles/reviewer.md` instead, which handles the `range-diff`. The reviewer is already
    adversarial and reports every finding with a confidence, so it needs no confidence-floor
    override.
  - spec and standards: a **read-only** companion `task` per axis (the Engineer command above
    **without** `--write`, with `--cwd <worktree> --model gpt-6-astra --effort high`), with a prompt file of `Role: Reviewer. Read
    agent-roles/reviewer.md and follow it exactly.` plus the same inputs as the Claude dispatch
    above.
  - Codex review is strongest on code correctness, not visual/UX taste. Pair it with a Claude visual
    pass or a screenshot check for UI diffs.

Effort mechanics: a Codex `task` takes `--effort` per call (`none`…`xhigh`), but
`adversarial-review` takes no `--effort` and inherits `model_reasoning_effort` from `~/.codex/config.toml` (not
per-call). Claude subagents accept an `effort` frontmatter field, which overrides the session level.
`architect` and `engineer` pin `xhigh`. `reviewer` deliberately has no `effort` field and inherits
the session's, so set the orchestrating session (`/effort`) to `high`, or `xhigh` on the high-stakes
path, before dispatching reviewers. Both vendors' reviewers **report findings only** — they never
edit.
Post each round's review verbatim as a PR `### Code review` comment (`### Code review — round N`),
with one sub-section per axis, its mode and range, and the adjudication of each finding. Put the
reviewed SHA in the body (a `Reviewed-SHA: <HEAD_SHA>` / `Reviewer: <model>` trailer per axis) so the
merge checklist's "comment cites HEAD_SHA" check passes.

**Process hygiene.** The `codex` plugin starts a long-lived **app-server broker** (a
`app-server-broker.mjs` process plus its `codex app-server` child) **per git workspace root**.
Every Engineer worktree therefore gets its own broker. Brokers and `--background` task workers are
spawned *detached* (their own session), so killing the shell or process group that launched them
doesn't reach them, and brokers have no idle timeout. The plugin's `SessionEnd` hook tears down only
the broker for the session's own directory. Worktree brokers outlive the session unless you tear them
down, and so does every broker if the session ends abnormally.
- Start companion runs with the Bash tool's `run_in_background`, never in the foreground with a
  timeout. Waiting is the harness's job; killing a run by accident is not.
- Collect each job's output (`codex-companion.mjs result --cwd <worktree> [job-id]`). To abandon a
  run, cancel it (`codex-companion.mjs cancel --cwd <worktree> <job-id>`; `status --cwd <worktree>`
  lists the jobs). `cancel` interrupts the Codex turn and kills the worker's process tree.
- **Before removing an Engineer worktree, tear down its broker.** Run the plugin's own `SessionEnd`
  cleanup pointed at that worktree. It shuts down the broker and its app-server, kills this session's
  still-running jobs there, and clears the plugin's state for the worktree:
  ```
  printf '{"cwd":"%s","session_id":"%s"}' "<worktree>" "$CODEX_COMPANION_SESSION_ID" |
    node "$CODEX_ROOT/scripts/session-lifecycle-hook.mjs" SessionEnd
  ```
  Run it while the worktree still exists: the plugin resolves the worktree's git root to find its
  state. This calls an internal plugin script, not a documented command, so re-check it after a plugin
  update.
- Never `pkill -f codex`. It also kills other sessions' runs.
- Processes that tests start are the Engineer's to tear down (`agent-roles/engineer.md`). Any it
  reports under "Left running" are the orchestrator's to stop.

**Once per session**, before relying on either vendor:
- Confirm both vendors are usable (Codex logged in, astra/sol respond).
- Check for leftover Codex processes from earlier sessions:
  `ps -eo pid,etime,command | grep -E '[a]pp-server-broker|[c]odex app-server|[c]odex-companion'`.
  Each broker's command line shows its `--cwd`. A broker whose worktree no longer exists is a leak.
  Report any to the human instead of killing them: another live session may own them.

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

- **In-scope regression the diff introduces** → fix, then run a delta review round over the fix.
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

1. **Review record posted.** The PR has a `### Code review — round N` comment for every round, each
   with every axis's report, the adjudication of each finding (fixed / deferred-to-`#N` /
   false-positive / rejected judgement call) and a `Reviewed-SHA:` trailer.
   - The rounds must chain without gaps: round 1 is `full` from base, and each later round's range
     starts at the previous round's `Reviewed-SHA`.
   - In the last round, every prior finding marked fixed was `verified`.
2. **No lost findings.** Every deferred finding (MINOR, pre-existing, out-of-scope) has an issue that
   already exists and is linked from the review comment.
3. **No SHA drift.** The PR's current head SHA equals the last round's `Reviewed-SHA`. Any commit after
   that review, including a rebase, needs another delta round. That round is not optional.
4. **CI green** on that head SHA.
5. **Human approval on that SHA**, on the high-stakes path only.
6. **Squash-merge.** `Closes #N` closes the issue. If the PR ships only part of the issue, the
   follow-up issue must already exist (see the operational rules below).
7. **Clean up.** Tear down the worktree's Codex broker (see **Process hygiene**), then remove the
   worktree, delete the branch, and fast-forward the local `main`.

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
5. **Clean up** — collect or cancel any Codex jobs this session started, tear down each worktree's
   Codex broker, then remove the worktree once merged and prune remote branches, so no Codex process
   outlives the session (see **Process hygiene**).
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
