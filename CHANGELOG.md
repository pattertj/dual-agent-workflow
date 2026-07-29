# Changelog

All notable changes to this template are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Because this is a policy
*template* rather than software, "breaking" means a change that alters an existing rule or placeholder
in a way that would change how already-adopted repos behave if they re-copied the templates.

## [Unreleased]

<!-- Add new entries here as you revise the policy. -->

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

[Unreleased]: https://github.com/pattertj/dual-agent-workflow/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/pattertj/dual-agent-workflow/releases/tag/v0.1.0
