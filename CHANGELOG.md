# Changelog

All notable changes to this template are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Because this is a policy
*template* rather than software, "breaking" means a change that alters an existing rule or placeholder
in a way that would change how already-adopted repos behave if they re-copied the templates.

## [Unreleased]

<!-- Add new entries here as you revise the policy. -->

### Added

- **Role definitions** in `templates/`, copied into the target repo as-is:
  - `agent-roles/engineer.md` and `agent-roles/reviewer.md`: role contracts both vendors follow,
    covering inputs, rules, stop conditions and a fixed report format.
  - `.claude/agents/architect.md`, `engineer.md` and `reviewer.md`: Claude Code subagents. The
    architect and reviewer are read-only, with no edit tools. The engineer uses
    `isolation: worktree` and preloads the `tdd` skill. All three use the `opus` alias, so a Claude
    model bump needs no edits here.
- **TDD via the `tdd` skill** (mattpocock/skills) for both vendors' engineers: pre-agreed seams
  (supplied by the Architect's design doc), vertical slices, red for the right reason, no tautological
  or implementation-coupled tests, and refactoring left to review. The existing
  mutate-the-fix discrimination proof is kept.
- **Review axes**, adapted from mattpocock/skills `code-review`: correctness, spec and standards, each
  run by a separate fresh reviewer in parallel, with no cross-axis reranking. The standards axis
  carries a Fowler code-smell baseline as MINOR judgement calls that the repo's documented standards
  override. Unlike the source skill, reviewers have no word cap.
- **Merge checklist** inline in `AGENTS.md` (review record, no lost findings, no SHA drift, CI, human
  SHA approval on the high-stakes path, squash-merge, clean-up). It replaces the dependency on an
  unshipped `/merge` skill.
- **Process hygiene** for Codex dispatch: background runs, a process group per run via a portable
  `perl` wrapper, killing the whole group on abort, collecting or cancelling plugin jobs, a check for
  leftover Codex processes once per session, and engineers tearing down what their tests start.
- A recorded decision on why Codex is dispatched through the plugin companion rather than an MCP
  server, and what would reopen the question.

### Changed

- **Breaking:** the confidence-floor override is now built into the repo's own `reviewer` subagent
  instead of being restated in each dispatch prompt. Re-copy `templates/` along with `AGENTS.md`.
- Engineer reports are evidence-backed (red output, discrimination proof, gate results, deviations,
  processes left running), so the "probationary" check on Codex runs compares the report against the
  diff.
- Effort mechanics corrected: Claude Code subagents *do* accept an `effort` frontmatter field.
  `architect` and `engineer` pin `xhigh`. `reviewer` inherits the session's effort, so the
  orchestrator still sets `high` or `xhigh` per path.

## [0.2.0] - 2026-10-02

### Changed

- **Breaking:** model routing bumped to the current tiers. Claude: `claude-opus-4-8[1m]` →
  `claude-opus-5-5[1m]`. Codex: the high-stakes / frontend-review tier `gpt-5.6-sol` → `gpt-6-astra`;
  the routine / boilerplate tier `gpt-5.6-terra` → `gpt-6.1-sol`. The Codex dispatch command and
  `/codex:adversarial-review` examples now pass these full IDs (the `codex` plugin doesn't resolve
  `astra`/`sol` aliases).
- Claude Code (v2.1.277+) reads `AGENTS.md` natively, including nested `AGENTS.md` files in
  subdirectories. `CLAUDE.md` is now **optional**: `CLAUDE.template.md` is only for repos that already
  have a `CLAUDE.md` (which Claude Code reads *instead of* `AGENTS.md` by default) or that run older
  Claude Code. README, GUIDE and SETUP_PROMPT updated to match.
- Effort mechanics clarified: Codex `task` takes `--effort` per call (up to `xhigh`) while Codex review
  reads `model_reasoning_effort` from `~/.codex/config.toml`. Claude effort is a session setting, not a
  per-subagent one.
- Claude Engineer subagents use `isolation: worktree` for the per-agent worktree rule.
- Setup docs include the Codex plugin install steps (`openai/codex-plugin-cc`, `codex@openai-codex`,
  `/codex:setup`).

## [0.1.0] - 2026-07-29

### Added

- `AGENTS.template.md` — canonical cross-vendor policy (Claude Code + Codex). Work-type routing table,
  cross-vendor review rule, reviewer confidence-floor override, staged high-stakes delivery pipeline,
  Codex dispatch mechanics, review-finding adjudication, and "landing the plane" session-completion
  rules. Placeholders: `{{HIGH_STAKES_DOMAIN}}`, `{{TASK_TRACKER}}`, `{{TEST_CMD}}`,
  `{{INTEGRATION_CMD}}`, `{{LINT_CMD}}`.
- `CLAUDE.template.md` — thin pointer that imports `AGENTS.md` via `@AGENTS.md`; holds no policy of its
  own so the two files can't drift.
- `SETUP_PROMPT.md` — a prompt to hand an agent running inside a target repo so it performs the setup
  (inspect repo, fill placeholders, verify Codex dispatch, wire the files, report back).
- `GUIDE.md` — from-scratch explainer for newcomers: the one-sentence idea, why cross-vendor review
  beats self-review, the roles, the routing table, first-time setup, and two day-in-the-life
  walkthroughs.
- `README.md` — repo landing page and quick start.
- `.github/ISSUE_TEMPLATE/policy-change.md` — issue template for proposing policy changes.
- `LICENSE` — MIT.

[Unreleased]: https://github.com/pattertj/dual-agent-workflow/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/pattertj/dual-agent-workflow/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/pattertj/dual-agent-workflow/releases/tag/v0.1.0
