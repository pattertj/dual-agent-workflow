# Setup prompt — port the Claude/Codex cross-vendor agent policy into this repo

Paste everything below the line into a fresh Claude Code (or Codex) session **running inside the target
repository**. Attach `AGENTS.template.md` (and `CLAUDE.template.md`, used only if the repo needs a
`CLAUDE.md`) alongside it.

---

You are setting up a **two-vendor coding-agent policy** for this repository: **Claude Code** and
**Codex** implement and review each other's work. The whole point is that **the reviewer is always the
opposite vendor of the author**, so review is independent by construction. Claude handles frontend/UI +
orchestration + review of Codex's work; Codex (`gpt-6-astra` for high-stakes backend, `gpt-6.1-sol`
for routine) handles backend implementation + review of Claude's frontend work.

I've attached `AGENTS.template.md` (the canonical shared policy) and `CLAUDE.template.md` (an optional
thin pointer that imports it). Adapt the policy to THIS repo and write it in place as `AGENTS.md` at the
repo root. Codex and Claude Code (v2.1.277+) both read `AGENTS.md` natively, so a `CLAUDE.md` is only
needed in the cases in Step 4. The vendors, model IDs, and Codex dispatch command are
already filled in — **do not genericize them**. You only need to supply the repo-specific bits.

## Step 1 — Learn this repo before writing anything

Do NOT fill placeholders from assumptions. First inspect the repo and answer:

- **Languages / stack** and where the code lives (frontend dirs, backend/service dirs, packages). This
  tells you whether the Opus-vs-Astra/Sol split maps cleanly onto real directories.
- **What is the highest-stakes / highest-blast-radius area?** (money-path, auth, payments, data
  migrations — anything where a silent bug is expensive.) This becomes `{{HIGH_STAKES_DOMAIN}}` and
  drives both the Astra routing and the human-SHA merge gate.
- **The exact commands** for the full test suite, the integration-test tier (if separate — the pipeline
  requires running it), and lint. These become `{{TEST_CMD}}`, `{{INTEGRATION_CMD}}`, `{{LINT_CMD}}`.
- **Where task tracking lives** (`{{TASK_TRACKER}}`).

If any are ambiguous, **ask me** before writing the files.

## Step 2 — Fill in the placeholders

Replace every `{{...}}` token:

| Placeholder | Meaning |
|---|---|
| `{{HIGH_STAKES_DOMAIN}}` | This repo's money-path / high-blast-radius area. |
| `{{TASK_TRACKER}}` | Where issues live + how agents touch it (e.g. GitHub Issues via `gh`). |
| `{{TEST_CMD}}` | Full unit-test-suite command. |
| `{{INTEGRATION_CMD}}` | Integration-tier command (if none exists, say so and remove the reference). |
| `{{LINT_CMD}}` | Lint command(s) the Engineer must pass before opening a PR. |

Leave the model IDs (`claude-opus-5-5[1m]`, `gpt-6-astra`, `gpt-6.1-sol`) as-is unless you know they've
changed. Keep full Codex IDs — the `codex` plugin doesn't resolve `astra`/`sol` aliases. If this repo
has **no frontend**, the Codex-reviews-Opus rows can never fire — note that and
collapse those rows rather than leaving dead routing.

## Step 3 — Verify the Codex dispatch mechanics actually work here

The template's "Model routing & cross-vendor review" section hardcodes how Claude (the orchestrator)
shells out to Codex:

```
CODEX_ROOT=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
node "$CODEX_ROOT/scripts/codex-companion.mjs" task \
  --model {gpt-6-astra|gpt-6.1-sol} --effort {high|low} --write --prompt-file <prompt.txt>
```

Confirm the `codex` plugin is installed (that cache path resolves) and Codex is logged in. If it isn't
installed on the machine that will run this, leave the section intact but add
`TODO(human): install the codex plugin (/plugin marketplace add openai/codex-plugin-cc,
/plugin install codex@openai-codex, /codex:setup) before this pipeline can run`. Do **not** invent a
different command.

## Step 4 — Wire up the files correctly

- **No existing `CLAUDE.md`:** don't create one. Claude Code reads `AGENTS.md` natively.
- **An existing `CLAUDE.md`** (or the team runs Claude Code older than v2.1.277): Claude Code reads
  `CLAUDE.md` *instead of* `AGENTS.md` when both exist, so `CLAUDE.md` must import it with an
  `@AGENTS.md` line. Use `CLAUDE.template.md` as the shape. `CLAUDE.md` must contain **no policy of its
  own** — move any policy it already has into `AGENTS.md` so the two never drift.
- Keep every **Claude Code:** / **Codex:** marker so each agent follows only its own mechanics.
- Delete the `> Placeholder key` block from `AGENTS.md` once you're done.
- If this repo already has a `CLAUDE.md`/`AGENTS.md` with real content, don't blow it away — show me a
  merge plan first.

## Step 5 — Preserve these invariants (they are the point, don't water them down)

1. **Reviewer is always the opposite vendor of the author, fresh every round.** Claude reviews Codex;
   Codex reviews Claude. Never let an author review its own work.
2. **TDD binds both vendors** — red test first, prove it discriminates by mutating the fix.
3. **Every writing agent works in its own git worktree**; the shared main checkout stays read-only.
4. **Reviewers report findings only; they never edit.** The engineer applies fixes and re-review runs
   on the final diff.
5. **Override the Claude reviewer's confidence floor at dispatch** — report everything, classify
   severity in a separate pass. (Codex's `/codex:adversarial-review` needs no override.)
6. **The review loop exits when every finding is fixed-or-filed**, not when the verdict says
   "approve"; deferral issues must exist before merge.
7. **Human approval on the final SHA is required only for the high-stakes path.**
8. **Work isn't done until `git push` succeeds.**

## Step 6 — Report back

Show me a diff of every file you wrote, plus a short list of every `TODO(human)` you left and every
assumption you made, so I can confirm before this policy governs real work.
