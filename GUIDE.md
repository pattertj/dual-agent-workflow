# Cross-vendor agent workflow — a guide for newcomers

This explains the "Claude + Codex review each other" pattern from the ground up: what it is, why it's
shaped this way, how to stand it up in a repo, and how a normal day of work flows through it. If you've
never run a two-agent workflow before, start here.

---

## 1. The one-sentence idea

**Two AI coding tools from two different vendors take turns: whichever one writes the code, the *other
one* reviews it — never itself.** That single rule (cross-vendor review) is what makes the whole thing
worth the setup.

Everything else in `AGENTS.md` — the routing table, the pipeline, the merge gates — exists to serve
that rule and to keep the process honest.

## 2. Why bother (the problem it solves)

An AI that reviews its own work is a bad reviewer. It's anchored to the choices it just made, it shares
the same blind spots, and it tends to rubber-stamp. You get the *feeling* of review without the
substance.

Using a **different vendor** for review breaks that:

- **Independence by construction.** The reviewer literally did not write the code and doesn't share the
  author's model biases, so it catches things a self-review never would.
- **Two model families, two sets of blind spots.** Claude and Codex fail differently. Where they
  disagree is exactly where the interesting bugs live.
- **A paper trail.** Findings get written down, adjudicated, and either fixed or filed as issues — so
  nothing quietly disappears.

The cost is coordination overhead, which is why the process scales *down* for cheap changes (one light
review) and *up* for risky ones (a full staged pipeline + human sign-off).

## 3. The cast

| Role | Who | What they do |
|---|---|---|
| **Orchestrator** | Claude Code | The agent *you* talk to. It plans, dispatches the others, and adjudicates review findings. |
| **Architect** | Claude (Opus, read-only) | For risky work: traces the code, confirms the root cause, writes a design doc. Doesn't write code. |
| **Engineer** | Claude *or* Codex | Writes the code, test-first, in an isolated worktree. Frontend → Claude; backend/high-stakes → Codex. |
| **Reviewer** | The *opposite* vendor of the Engineer | Reads the full diff, reports findings. **Never edits.** |
| **Human (you)** | You | Approve the final commit on high-stakes changes; otherwise just steer. |

The key constraints: the reviewer is **always the opposite vendor**, is **fresh every round** (no
memory of prior rounds, so no anchoring), and **only reports** — the Engineer applies every fix.

## 4. Who writes what (the routing table)

Work is routed by **the riskiest thing the diff touches**:

- **Frontend / UI / visual** → **Claude (Opus)**. Reviewed by **Codex**.
- **High-stakes backend** (money-path, auth, migrations, anything expensive to get wrong) →
  **Codex `sol`**. Reviewed by **Claude**.
- **Routine backend / glue** → **Codex `terra`**. Reviewed by **Claude**.
- **Pure boilerplate** (renames, config, codegen, zero logic) → **Codex `terra`** at low effort. Light
  Claude pass, or skip review entirely.
- **Docs / comments** → whoever; no review needed.

A diff that spans UI *and* high-stakes logic gets **split** into two pieces so each half is authored by
the right vendor. If it can't be split, the high-stakes half wins and Codex takes it.

## 5. The two files, and how they fit together

- **`AGENTS.md`** — the single source of truth. Both tools read it. Codex reads it natively; Claude
  reads it via an `@AGENTS.md` import.
- **`CLAUDE.md`** — a *thin pointer* that just imports `AGENTS.md`. It holds **no policy of its own**,
  so the two files can never drift apart.

Some rules are tool-specific and tagged **Claude Code:** or **Codex:** in `AGENTS.md` — each agent
follows only its own tag and ignores the other's.

## 6. Setting it up (first time)

1. **Prereqs.** Install Claude Code and the `codex` plugin; log into both. Confirm Codex responds.
2. **Drop in the templates.** Copy `AGENTS.template.md` → `AGENTS.md` and `CLAUDE.template.md` →
   `CLAUDE.md` at your repo root.
3. **Fill the placeholders** (or better — hand the templates plus `SETUP_PROMPT.md` to a Claude session
   running inside the repo and let it do steps 2–4 for you). You supply:
   - `{{HIGH_STAKES_DOMAIN}}` — your money-path / high-blast-radius area.
   - `{{TASK_TRACKER}}` — where issues live.
   - `{{TEST_CMD}}` / `{{INTEGRATION_CMD}}` / `{{LINT_CMD}}` — the commands an Engineer must pass.
4. **Verify the Codex dispatch command resolves** on the machine that'll run it (the plugin cache path
   in the "Model routing" section).
5. **Delete the `> Placeholder key` block** from `AGENTS.md` once filled in.
6. **Sanity check:** ask Claude to summarize the routing rules back to you. If it can, the files loaded
   correctly.

## 7. A normal day (two walkthroughs)

**Small change** (fix a typo'd label, tweak a query):
1. You describe the change to Claude.
2. Claude routes it (UI → it builds; backend → it dispatches Codex), test-first, in a worktree.
3. **One** cross-vendor review pass. Fix findings or file them.
4. CI green → squash-merge → issue closed. Done.

**High-stakes change** (touches `{{HIGH_STAKES_DOMAIN}}`):
1. **Architect** (Claude, read-only) confirms the root cause and writes a design doc.
2. **Engineer** (Codex `sol`) implements it via strict TDD — red test first, watch it fail, then make
   it pass — runs the full suite + integration + lint, opens a PR.
3. **Reviewer** (fresh Claude) reviews the *entire* diff and reports findings. Codex fixes them. Repeat
   with a **new** reviewer each round until every finding is **fixed or filed**.
4. **You approve the final commit SHA.** (This human gate applies *only* to high-stakes work.)
5. Merge, close the issue, clean up the worktree.

## 8. The rules people trip over

- **"Approve" is not the finish line.** The loop ends when every finding is *fixed or filed as an
  issue* — a green verdict with unaddressed findings is not done.
- **Reviewers never edit.** If a reviewer starts "helpfully" fixing things, it's now an author and can
  no longer review that code. Findings only; the Engineer fixes.
- **Override the Claude reviewer's confidence floor.** Stock reviewer agents suppress anything below
  ~80% confidence and quietly drop real bugs. Tell the reviewer to report *everything* and sort
  severity afterward. (Codex's adversarial reviewer already does this.)
- **Fresh reviewer every round.** Re-using the same reviewer anchors it to its earlier take. New one
  each time.
- **Everyone works in their own worktree.** Two agents in one checkout will clobber each other and leak
  edits into `main`.
- **Out-of-scope ≠ ignore.** A real bug the diff didn't introduce still gets a filed issue before
  merge — you just don't block *this* PR on it. "Out of scope" is not a way to wave away a regression.
- **Not done until it's pushed.** A finished-but-unpushed branch is stranded work.

## 9. When *not* to use the full pattern

- Solo throwaway scripts, spikes, or scratch work — the ceremony isn't worth it.
- Repos with only one AI tool available — cross-vendor review can't happen, so simplify instead of
  faking it.
- Pure docs/comment changes — no review needed.

Scale the process to the risk. The staged pipeline is for the scary stuff; a one-line CSS fix gets one
quick review and ships.

## 10. Files in this repo

- `README.md` — the repo landing page: what this is and how to get started fast.
- `AGENTS.template.md` → copy to `AGENTS.md` in your repo — the canonical policy. Edit this one.
- `CLAUDE.template.md` → copy to `CLAUDE.md` — the pointer. Don't put policy here.
- `SETUP_PROMPT.md` — hand this to an LLM inside the target repo to do the setup for you.
- `GUIDE.md` — this guide.
- `LICENSE` — MIT.
