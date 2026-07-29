# AGENTS.md

Canonical policy for every coding agent in this repository — read it before making any change. It loads
automatically for both agents: **Codex** reads this file natively; **Claude Code** imports it via
`@AGENTS.md` in `CLAUDE.md`. This is the single source of truth — `CLAUDE.md` holds no policy of its own.

Most rules are universal. A few are tool-specific and marked **Claude Code:** or **Codex:** — follow
the one that applies to you, ignore the other.

> **Placeholder key** (delete this block once filled in):
> - `{{HIGH_STAKES_DOMAIN}}` — your money-path / high-blast-radius area (e.g. "order execution, auth,
>   payments"). Drives the Sol-vs-Terra routing split and the human-SHA gate.
> - `{{TASK_TRACKER}}` — where issues live (e.g. "GitHub Issues on org/repo", via the `gh` CLI).
> - `{{TEST_CMD}}` / `{{INTEGRATION_CMD}}` / `{{LINT_CMD}}` — the full-suite, integration-tier, and lint
>   commands the Engineer must run before opening a PR.
> - Model IDs below (`claude-opus-4-8[1m]`, `gpt-5.6-sol`, `gpt-5.6-terra`) are Claude/Codex defaults —
>   bump them to the current ids if they've moved on.

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

**Claude Code:** drive development through your workflow skills — test-driven-development (red first),
worktree isolation, subagent parallelization. Codex has its own equivalents; the TDD requirement above
is what binds both.

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
here means {{HIGH_STAKES_DOMAIN}} **backend logic** — inherently backend, so it routes to **Sol**.
Frontend/UI is **never** high-stakes for routing: a UI change is Opus even for a high-stakes-adjacent
feature; a diff spanning UI + high-stakes logic is **split** (UI → Opus, logic → Sol), or if
inseparable, the high-stakes logic governs → Sol.

| Work type | Implementer | Effort |
|---|---|---|
| Frontend / UI / UX, visual/design | **claude-opus-4-8[1m]** | `xhigh` |
| Backend, high-stakes — {{HIGH_STAKES_DOMAIN}}, logic-bearing migrations | **Codex `gpt-5.6-sol`** | `high` |
| Routine backend / coding — ordinary services, non-UI, CRUD, analytics/admin reads, glue | **Codex `gpt-5.6-terra`** | `high` |
| Boilerplate / mechanical — codegen, mass rename, config, deps, test scaffolding, 0-logic edits | **Codex `gpt-5.6-terra`** | `low` |

**Reviewer = the opposite vendor @ `high`** (mechanics in "Model routing & cross-vendor review"):

| Implemented by | Reviewer |
|---|---|
| Codex sol / terra (backend, routine, boilerplate) | **claude-opus-4-8[1m] @ `high`** — `xhigh` on the high-stakes path |
| claude-opus-4-8[1m] (frontend) | **Codex `gpt-5.6-sol` @ `high`** via `/codex:adversarial-review` |
| Boilerplate (terra @ `low`) | light Opus pass; **skip** when 0-logic |
| Docs / comments | none |

Planning / **Architect stays claude-opus-4-8[1m] at `xhigh`** — it is not the author, so a fresh
opposite-vendor reviewer of the implementation still applies. This routing governs **all**
implementation work; the staged pipeline below is its high-stakes instance, and lighter changes use the
same implementer/reviewer routing without the full Architect design doc.

#### Reviewer confidence filters — override them

This applies to **Claude** reviewers — the reviewer for all Codex-implemented work (backend, routine,
boilerplate), plus the everyday `/code-review` path. The stock `feature-dev` and `pr-review-toolkit`
reviewer agents ship with a hard reporting floor:

- `feature-dev/agents/code-reviewer.md` — *"Only report issues with confidence ≥ 80."*
- `pr-review-toolkit/agents/code-reviewer.md` — *"Only report issues with confidence ≥ 80"*

Modern models take that literally and **under-report**: they find real bugs at high precision and
recall even at low effort, then suppress them for not clearing an arbitrary bar. On the high-stakes
path a suppressed finding is a shipped defect.

**When dispatching either reviewer, override the filter in the prompt** — instruct it to report
everything it finds and to classify severity (BLOCKER / MAJOR / MINOR) in a **separate pass after
discovery**, not to gate discovery on a confidence score. Filtering is the adjudicating agent's job,
not the finder's.

These agent definitions live in installed plugins under `~/.claude/plugins/`, **outside this
repository** — they are third-party files, so do not edit them (the change is lost on plugin update and
is not version-controlled). Override at dispatch time instead.

The Codex `/codex:adversarial-review` reviewer carries **no** confidence floor — its prompt is already
adversarial and reports every finding with its own 0–1 confidence — so no override is needed for it.

### Multi-agent delivery pipeline (high-stakes work)

For high-risk issues ({{HIGH_STAKES_DOMAIN}}), run each issue through a staged agent pipeline; stages
hand off via files/PRs, never shared state:

1. **Architect** (claude-opus-4-8[1m] @ `xhigh`, read-only) — traces code end-to-end, confirms/refutes
   the issue's hypothesized root cause, picks an approach vs alternatives, writes a self-sufficient
   design doc (files/functions, failure-mode analysis, TDD test plan, out-of-scope).
2. **Engineer** (**Codex `gpt-5.6-sol`** for high-stakes backend — this pipeline's default; a high-risk
   frontend change uses the same cross-vendor routing, Opus-built and Sol-reviewed, but without this
   pipeline's human-SHA gate; own worktree) — implements the design via strict TDD (red first, watch it
   fail; prove discrimination by mutating the fix and watching tests go red), runs `{{TEST_CMD}}` +
   `{{INTEGRATION_CMD}}` + `{{LINT_CMD}}`, opens a PR with `Closes #N`. Deviations from the design must
   be justified in the PR body.
3. **Principal reviewer** — the **opposite vendor** of the Engineer, a fresh review every round (no
   anchoring), over the FULL PR diff against base. For the usual high-stakes case (Sol-implemented)
   that is **claude-opus-4-8[1m] @ `xhigh`** via a fresh Claude reviewer subagent (with the
   confidence-floor override); for Opus-implemented work it is **Codex `gpt-5.6-sol`** via
   `/codex:adversarial-review`. The reviewer **reports findings only** — it never edits code; the
   Engineer applies fixes and the loop repeats until every finding is **fixed or filed** as a scoped
   issue — not merely until the verdict reads `approve`. Classify findings BLOCKER/MAJOR/MINOR when
   recording. Expect this loop to earn its keep: successive fresh reviews routinely find distinct real
   blockers.
4. **Merge** (`/merge` skill) — post the review record as a `### Code review` PR comment citing the
   approved SHA, defer MINOR and out-of-scope findings to issues (never lose them), CI green, SHA-drift
   check, squash-merge (auto-closes the issue), remove the worktree, fast-forward main.

On the high-stakes path the Engineer is **Codex `gpt-5.6-sol`** and the reviewer is a fresh
**claude-opus-4-8[1m] @ `xhigh`** (cross-vendor by construction); these changes additionally need
**human approval on the final SHA** before merge. That is the only category requiring it — everything
else merges on the review above.

Everything outside this pipeline uses the same implementer/reviewer routing, scaled to the change: one
cross-vendor review at the `high` floor (docs, comments, and pure boilerplate need `low` or none).

#### Model routing & cross-vendor review (dispatch mechanics)

These are **Claude Code (orchestrator) dispatch mechanics** — how the agent driving the pipeline
invokes each implementer/reviewer. A dispatched Codex or Claude engineer/reviewer does not run these;
it just implements or reviews per its prompt.

**Codex models (implementation)** run through the installed `codex` plugin's app-server runtime (no
`codex exec` stdin gotchas). Implement in an isolated worktree via (resolve the current plugin version —
do not hardcode it; the cache dir changes on plugin update):

```
CODEX_ROOT=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
node "$CODEX_ROOT/scripts/codex-companion.mjs" task \
  --model gpt-5.6-{sol|terra} --effort {high|low} --write --prompt-file <prompt.txt>
```

`--write` is **required** to edit files — without it the sandbox is read-only and the task can only
inspect ("workspace is mounted read-only"). Full-suite runs and self-commit also need network in the
sandbox: set `[sandbox_workspace_write] network_access = true` in `~/.codex/config.toml` (otherwise
loopback-binding tests fail and the orchestrator must run the full gate + commit outside the sandbox).
Pass long prompts via `--prompt-file`. Structure the prompt: hand it the Architect design doc, require
strict red-first TDD, running `{{TEST_CMD}}` + `{{INTEGRATION_CMD}}` + `{{LINT_CMD}}`, and a commit with
the standard trailers. Treat early Sol/Terra runs as **probationary** — verify the red-first discipline
and the integration run actually happened, not just that tests are green at the end.

**claude-opus-4-8[1m] (implementation)** — frontend/UI work runs as a Claude Engineer subagent at
`xhigh` in its own worktree, same TDD discipline.

**Reviewers** are always the opposite vendor, fresh every round, over the FULL PR diff:
- **Codex sol reviews Opus (frontend) work:**
  `/codex:adversarial-review --model gpt-5.6-sol --base origin/main --scope branch [focus]`
  (`--background` for large diffs, `--wait` otherwise). Sol is strong on code correctness but weak on
  visual/UX taste — pair it with a Claude visual pass or a screenshot check for UI diffs.
- **claude-opus-4-8[1m] reviews Codex (backend/routine/boilerplate) work:** a fresh Claude reviewer
  subagent at `high` (`xhigh` on the high-stakes path), with the confidence-floor override. Boilerplate
  (terra @ `low`) gets a light Opus pass, or none when 0-logic.

Review effort: a Codex review inherits reasoning-effort from `~/.codex/config.toml` (not per-call); the
Claude reviewer's effort is set at dispatch. Both reviewers **report findings only** — they never edit.
Post the review verbatim as the PR `### Code review` comment with the reviewed SHA in the body (a
`Reviewed-SHA: <HEAD_SHA>` / `Reviewer: <model>` trailer) so `/merge`'s "comment body cites HEAD_SHA"
check passes. Once per session, confirm both vendors are usable (Codex logged in, sol/terra respond)
before relying on them.

#### Review-finding adjudication

A strong adversarial reviewer keeps surfacing real defects, including **pre-existing** and
**out-of-scope** ones — that is a feature, not noise. Deciding which findings gate *this* PR is the
**adjudicating agent's** job (the orchestrator), never the reviewer's. After each review round,
classify every finding:

- **In-scope regression the diff introduces** → fix, then re-review.
- **Out-of-scope / pre-existing / adjacent / scope-creep** → file an issue (area + priority label),
  link it in the `### Code review` comment; it does **not** block this PR.
- **Minor / nit on changed lines** → fix if trivial; otherwise file a low-priority issue.

The review loop exits when every finding is **fixed or filed** — not merely when the verdict reads
`approve`. Merging over a `needs-attention` verdict is allowed **only** when every residual finding has
a filed issue and the adjudication (each finding → fixed / deferred-to-`#N` / false-positive) is
recorded in the review comment. Every deferral issue must exist **before** merge — a `Closes #N` that
silently drops a finding is the exact failure this prevents.

**Anti-abuse:** "out of scope" means genuinely unrelated to the diff's purpose, not a real regression
rationalized away to clear the gate. When in doubt, treat it as in-scope, or ask the human. On the
high-stakes path, human approval on the final SHA is the backstop against mis-scoping a real blocker as
pre-existing.

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
5. **Clean up** — remove the worktree once merged, prune remote branches.
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
