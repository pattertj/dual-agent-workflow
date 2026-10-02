# dual-agent-workflow

A reusable **cross-vendor coding-agent policy**: two AI coding tools from two different vendors —
**Claude Code** and **Codex** — take turns implementing and reviewing each other's work. Whichever
agent writes the code, the *other one* reviews it. Never itself.

That single rule (cross-vendor review) is the point. An AI reviewing its own work is anchored to its
own choices and shares its own blind spots; a different vendor is independent by construction and fails
differently, so it catches what a self-review never would. Everything else in these templates — the
work-type routing, the staged pipeline, the merge gates — exists to serve that rule and keep the
process honest.

This repo is a **template**, not a library. You copy one file (`AGENTS.md`) into your own repo, fill in
a handful of project-specific placeholders, and both agents pick up the policy automatically — Codex and
Claude Code (v2.1.277+) both read `AGENTS.md` natively.

## What's in here

| File | What it is |
|---|---|
| **`AGENTS.template.md`** | The canonical shared policy. Copy to `AGENTS.md` in your repo. Both agents read it natively. This is the file you edit. |
| **`CLAUDE.template.md`** | *Optional.* A thin pointer that imports `AGENTS.md` (`@AGENTS.md`). Only needed if your repo already has a `CLAUDE.md` (Claude Code then reads it *instead of* `AGENTS.md`) or someone runs Claude Code older than v2.1.277. |
| **`SETUP_PROMPT.md`** | Hand this to a Claude/Codex session running *inside your target repo* and it will do the setup for you — inspect the repo, fill placeholders, wire the files, and report back. |
| **`GUIDE.md`** | A from-scratch explainer for people new to the pattern: why it's shaped this way, the roles, the routing table, and two day-in-the-life walkthroughs. Start here if it's your first time. |

## Quick start

**Option A — let an agent do it.** Open your target repo in Claude Code, attach `AGENTS.template.md`,
`CLAUDE.template.md`, and `SETUP_PROMPT.md`, and paste the body of `SETUP_PROMPT.md`. It handles the
rest and shows you a diff to approve.

**Option B — by hand.**
1. Copy `AGENTS.template.md` → `AGENTS.md` at your repo root. If the repo already has a `CLAUDE.md`,
   add an `@AGENTS.md` line to it (or copy `CLAUDE.template.md` → `CLAUDE.md`) — otherwise Claude Code
   reads `CLAUDE.md` and skips `AGENTS.md`.
2. Fill the placeholders:
   - `{{HIGH_STAKES_DOMAIN}}` — your money-path / high-blast-radius area (auth, payments, migrations…).
   - `{{TASK_TRACKER}}` — where issues live.
   - `{{TEST_CMD}}` / `{{INTEGRATION_CMD}}` / `{{LINT_CMD}}` — the commands an engineer must pass.
3. Install the Codex plugin in Claude Code and log in:
   `/plugin marketplace add openai/codex-plugin-cc` → `/plugin install codex@openai-codex` →
   `/reload-plugins` → `/codex:setup`. Then confirm the dispatch command in the "Model routing" section
   resolves on your machine.
4. Delete the `> Placeholder key` block from `AGENTS.md`.

New to the whole idea? Read **[`GUIDE.md`](GUIDE.md)** first.

## The invariants (don't water these down)

1. **Reviewer is always the opposite vendor of the author, fresh every round.** Never self-review.
2. **TDD binds both vendors** — red test first, prove it discriminates by mutating the fix.
3. **Every writing agent works in its own git worktree**; the shared main checkout stays read-only.
4. **Reviewers report findings only; they never edit.** The engineer applies fixes; re-review runs on
   the final diff.
5. **Override the Claude reviewer's confidence floor at dispatch** — report everything, sort severity
   after. (Codex's adversarial reviewer needs no override.)
6. **The review loop exits when every finding is fixed-or-filed**, not when the verdict says "approve."
7. **Human approval on the final SHA is required only for the high-stakes path.**
8. **Work isn't done until `git push` succeeds.**

## Status

Living template — expect it to change as the workflow evolves. Pin to a commit if you need stability.

## License

[MIT](LICENSE).
