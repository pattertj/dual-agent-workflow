<!--
  OPTIONAL. Claude Code v2.1.277+ reads AGENTS.md natively, so most repos need no CLAUDE.md at all.
  Copy this to CLAUDE.md only if:
    - someone runs Claude Code older than v2.1.277, or
    - the repo already has (or needs) a CLAUDE.md for other reasons.
  When a CLAUDE.md exists, Claude Code by default reads it *instead of* AGENTS.md, so the
  `@AGENTS.md` import below is what keeps the shared policy loaded. Delete this comment once copied.
-->
# CLAUDE.md

Claude Code's policy for this repository is the shared, canonical **[`AGENTS.md`](AGENTS.md)**, imported
below so it loads into context. There is **no separate Claude policy in this file** — edit `AGENTS.md`,
not this one.

The Claude-Code-specific mechanics in `AGENTS.md` are yours (Codex ignores them): anything marked
**Claude Code:**, the Claude-run orchestration and dispatch mechanics in "Model routing &
cross-vendor review" (including the `architect` / `engineer` / `reviewer` subagents in
`.claude/agents/`), and the slash commands (e.g. `/codex:adversarial-review`). Sections marked
**Codex:** are Codex's. Everything else, including the role contracts in `agent-roles/`, is shared
policy.

@AGENTS.md
