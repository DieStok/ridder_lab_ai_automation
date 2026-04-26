---
title: Initialize lab_ai_automation monorepo and migrate live Paper-FYI bot
type: feat
status: completed
date: 2026-04-26
origin: docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md
execution-report: docs/plans/2026-04-26-feat-lab-ai-monorepo-init-plan.execution-report.md
---

# Initialize lab_ai_automation monorepo and migrate live Paper-FYI bot

## Overview

Convert the static scaffolding at `/hpc/compgen/projects/lab_ai_automation/` (created during the 2026-04-26 brainstorm session) into a six-submodule git monorepo with active GitHub remotes for each. Move the live Paper-FYI Slack bot from its standalone HPC location into the monorepo as a submodule, preserving its remote and history. Cron stays disabled; the user re-enables it manually after the smoke-test passes.

This plan is the execution arm of the brainstorm at `docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md`. All naming, layout, distribution, and policy decisions are inherited from there. This plan adds precise command sequences, pre-flight checks, verification steps, and rollback procedures (see brainstorm: `docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md`).

## Problem Statement

The lab needs a single entry point for AI automation tooling so that:

1. Coding agents at the spring 2026 retreat can be pointed at one repo and see the lab's full surface.
2. New tools can be developed independently (own git history, own remote) but discovered together.
3. Reusable patterns and deployment recipes get extracted as the constellation grows.
4. The existing live Paper-FYI bot is preserved exactly as-is (no source-code edits) and continues to work after re-enabling cron at the new location.

The static structure exists. What's missing: git initialization of seven repos, push of initial commits to their remotes, submodule wiring in the parent, a verified clone-and-adopt of the live bot into the monorepo, and removal of its original on-HPC location once the bot is confirmed runnable from the new path.

## Proposed Solution

Four phases. Single explicit STOP POINT between Phase B and Phase C — the bot move is the only irreversible action.

- **Phase A — pre-flight (reversible).** Verify auth, remotes, identity, uv, scaffolding, bot state. ~10 min (estimate).
- **Phase B — wire 5 submodules + push monorepo (reversible).** Initialize the monorepo, push four scaffolded submodules to their fresh remotes, register them, register `coding_agent_installer` (existing remote, fresh clone), commit + push monorepo. ~45 min (estimate).
- **STOP POINT.** Surface state for user review.
- **Phase C — clone the bot into the monorepo, verify, then remove the original (irreversible at the final `rm`).** `git submodule add` clones the bot fresh from its existing remote, runtime state is rsync'd from the still-intact original location, venv is recreated, dry-run smoke-test runs, parent commit + push, **then** the original location is `rm -rf`'d as the last step. If anything fails before C.7, the original location is untouched and rollback is `git submodule deinit + rm` of the new path. ~30 min (estimate).
- **Phase D — execution report and handoff.** Phase-5-style report, list of source-code paths the user will refactor separately, suggested next steps. ~15 min (estimate).

## Technical Approach

### Architecture (final state)

```
/hpc/compgen/projects/lab_ai_automation/
├── .git/                                       # parent project, remote: ridder_lab_ai_automation.git
├── .git/modules/                               # cached .git dirs for all 6 submodules
├── .gitmodules                                 # 6 entries
├── README.md, AGENTS.md, .claude/CLAUDE.md, .gitignore
├── docs/{brainstorms, plans, prompt_for_claude_code_lab_ai_monorepo_updated.md}
├── slack_paperbot_ridder_lab/      → slack_paperbot_ridder_lab.git
├── deep_research_bot/              → ridder_lab_deep_research.git
├── summarize_bot/                  → ridder_lab_summarization_bot.git
├── coding_agent_installer/         → coding_agent_installer_ridder_lab.git
├── automation_building_blocks/     → ridder_lab_LLM_automation_building_blocks.git
└── hackathon/                      → ridder_lab_retreat_ai_hackathon_2026.git
```

### Implementation Phases

#### Phase A — Pre-flight (~10 min)

Goal: confirm we have everything we need so we don't roll back mid-execution.

**A.1 git identity**

```bash
git config user.name && git config user.email
```

If either is empty, abort and ask the user to set them globally.

**A.2 uv available**

```bash
command -v uv && uv --version
```

Expected: uv ≥ 0.4 on PATH.

**A.3 Pre-flight every remote** (all 7)

```bash
for url in \
  https://github.com/DieStok/ridder_lab_ai_automation.git \
  https://github.com/DieStok/ridder_lab_deep_research.git \
  https://github.com/DieStok/ridder_lab_summarization_bot.git \
  https://github.com/DieStok/coding_agent_installer_ridder_lab.git \
  https://github.com/DieStok/ridder_lab_LLM_automation_building_blocks.git \
  https://github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git \
  https://github.com/DieStok/slack_paperbot_ridder_lab.git ; do
    echo "=== $url ==="
    git ls-remote "$url" | head -5
    echo
done
```

Expected per URL:

| URL | Expected `ls-remote` |
|---|---|
| `ridder_lab_ai_automation.git` | empty (we will populate) |
| `ridder_lab_deep_research.git` | empty |
| `ridder_lab_summarization_bot.git` | empty |
| `ridder_lab_LLM_automation_building_blocks.git` | empty |
| `ridder_lab_retreat_ai_hackathon_2026.git` | empty |
| `coding_agent_installer_ridder_lab.git` | non-empty (existing repo; we clone it) |
| `slack_paperbot_ridder_lab.git` | non-empty (live bot's remote; we clone it) |

If a "should-be-empty" URL has content (e.g., GitHub auto-init README): halt, ask user to either delete + recreate empty or approve a one-time `fetch + merge` of the auto-init.

If any URL is unreachable / 401 / 404: halt, ask user to verify push rights and refresh credentials.

**A.4 Bot's git state**

```bash
cd /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot
git status
git log @{u}..HEAD     # local commits not on remote — must be empty
git log HEAD..@{u}     # remote commits not local — should be empty
```

Expected: working tree clean modulo untracked logs noted in Phase 0; HEAD = origin/main.

**A.5 Cron stays disabled and no jobs running**

```bash
crontab -l 2>/dev/null | grep -i 'slack-bot\|paper'
squeue -u dstoker
```

Both must be empty.

**A.6 Bot venv smoke-check**

```bash
cd /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot
.venv/bin/python -c "import slack_sdk, requests, bs4, ollama"
```

If imports fail, the venv is already broken — note for the user; not blocking (Phase C.4 recreates it anyway).

**A.7 Scaffolded subdirs present and intact**

```bash
cd /hpc/compgen/projects/lab_ai_automation
ls deep_research_bot summarize_bot automation_building_blocks hackathon
readlink .claude/CLAUDE.md hackathon/.claude/CLAUDE.md
```

Both symlinks should resolve to `../AGENTS.md`.

---

#### Phase B — Wire 5 submodules and push monorepo (~45 min)

**B.1 Initialize the monorepo**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git -c init.defaultBranch=main init
git remote add origin https://github.com/DieStok/ridder_lab_ai_automation.git
```

**B.2 For each scaffolded stub: init → commit → push → adopt as submodule**

The recipe (substitute `<NAME>` and `<URL>`):

```bash
cd /hpc/compgen/projects/lab_ai_automation
mv <NAME> _migrate_<NAME>_tmp
( cd _migrate_<NAME>_tmp && \
  git -c init.defaultBranch=main init && \
  git add . && \
  git commit -m "Initial scaffolding of <NAME>" && \
  git remote add origin <URL> && \
  git push -u origin main )
rm -rf _migrate_<NAME>_tmp                    # safe: content is on the remote
git submodule add <URL> <NAME>                # clones from the freshly-populated remote
```

Run this for all four stubs:

```bash
# --- deep_research_bot ---
cd /hpc/compgen/projects/lab_ai_automation
mv deep_research_bot _migrate_deep_research_bot_tmp
( cd _migrate_deep_research_bot_tmp && \
  git -c init.defaultBranch=main init && \
  git add . && \
  git commit -m "Initial scaffolding of deep_research_bot" && \
  git remote add origin https://github.com/DieStok/ridder_lab_deep_research.git && \
  git push -u origin main )
rm -rf _migrate_deep_research_bot_tmp
git submodule add https://github.com/DieStok/ridder_lab_deep_research.git deep_research_bot

# --- summarize_bot ---
mv summarize_bot _migrate_summarize_bot_tmp
( cd _migrate_summarize_bot_tmp && \
  git -c init.defaultBranch=main init && \
  git add . && \
  git commit -m "Initial scaffolding of summarize_bot" && \
  git remote add origin https://github.com/DieStok/ridder_lab_summarization_bot.git && \
  git push -u origin main )
rm -rf _migrate_summarize_bot_tmp
git submodule add https://github.com/DieStok/ridder_lab_summarization_bot.git summarize_bot

# --- automation_building_blocks ---
mv automation_building_blocks _migrate_abb_tmp
( cd _migrate_abb_tmp && \
  git -c init.defaultBranch=main init && \
  git add . && \
  git commit -m "Initial scaffolding of automation_building_blocks (slack_app_skeleton + deployment_recipes)" && \
  git remote add origin https://github.com/DieStok/ridder_lab_LLM_automation_building_blocks.git && \
  git push -u origin main )
rm -rf _migrate_abb_tmp
git submodule add https://github.com/DieStok/ridder_lab_LLM_automation_building_blocks.git automation_building_blocks

# --- hackathon ---
mv hackathon _migrate_hackathon_tmp
( cd _migrate_hackathon_tmp && \
  git -c init.defaultBranch=main init && \
  git add . && \
  git commit -m "Initial scaffolding of hackathon repo (Spring 2026 retreat)" && \
  git remote add origin https://github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git && \
  git push -u origin main )
rm -rf _migrate_hackathon_tmp
git submodule add https://github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git hackathon
```

**B.3 Add `coding_agent_installer` (existing remote, no scaffolding)**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git submodule add https://github.com/DieStok/coding_agent_installer_ridder_lab.git coding_agent_installer
```

**B.4 Verify**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git submodule status         # 5 entries
cat .gitmodules              # 5 entries (overwritten from the comment-only stub)
git status                   # untracked top-level files: README, AGENTS, etc.
```

**B.5 Commit and push the monorepo's initial state**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git add README.md AGENTS.md .claude .gitignore docs/ .gitmodules
git commit -m "Initial scaffolding of lab_ai_automation monorepo

Adds 5 of 6 submodules (deep_research_bot, summarize_bot,
automation_building_blocks, hackathon, coding_agent_installer).

The slack_paperbot_ridder_lab submodule is wired up by Phase C of the
plan, after the user explicitly approves moving the live cron-driven
bot from /hpc/compgen/projects/slackbot_fyi_papers_info/.

Origin: docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md"
git push -u origin main
```

**B.6 Verify on GitHub**: monorepo's `main` branch shows README, AGENTS, the 5 submodule pointers, and the brainstorm + plan in `docs/`.

---

### STOP POINT — between Phase B and Phase C

Surface to the user:

- `tree -L 2` of the monorepo
- `git -C lab_ai_automation submodule status` — should list 5 entries
- `cat .gitmodules` — should list 5 entries
- `git -C lab_ai_automation log --oneline` — should show 1 commit
- The Phase C command sequence (below), ready for explicit approval.

**Wait for "go" before any operation that touches the live bot.**

---

#### Phase C — Clone the bot into the monorepo, verify, then remove the original (~30 min)

This is the irreversible-at-the-end phase. The key property: **the original bot location stays intact and runnable until the very last step (C.7).** If anything fails between C.1 and C.6, rollback is a single `git submodule deinit + rm -rf` of the new path; the original location is untouched and the user can re-enable cron there if they want to abort the move entirely.

**C.1 Final pre-flight on the bot**

```bash
cd /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot
git status
git log @{u}..HEAD                   # must be empty (no local commits not on remote)
crontab -l 2>/dev/null | grep -i 'slack-bot\|paper'   # must be empty
squeue -u dstoker                    # must be empty
```

**C.2 Submodule-add the bot from its remote** (fresh clone into the monorepo path)

```bash
cd /hpc/compgen/projects/lab_ai_automation
git submodule add https://github.com/DieStok/slack_paperbot_ridder_lab.git slack_paperbot_ridder_lab
```

Same content as the original `slack-paper-bot/` directory for tracked files (verified by C.1). Different path, fresh `.git` inside `.git/modules/slack_paperbot_ridder_lab/`. The original location is **not yet touched**.

**C.3 Sync gitignored runtime state from the original (still-intact) location**

```bash
BOT=/hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot
NEW=/hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab
cp "$BOT/config/.env" "$NEW/config/.env"
rsync -a "$BOT/data/" "$NEW/data/"
rsync -a "$BOT/logs/" "$NEW/logs/"

# Spot-check: processed_messages.json must be non-trivial. Phase 0 saw ~2696 bytes.
ls -la "$NEW/config/.env"
wc -c "$NEW/data/processed_messages.json"
PMSIZE=$(stat -c%s "$NEW/data/processed_messages.json" 2>/dev/null || echo 0)
test "$PMSIZE" -gt 100 && echo "OK: processed_messages.json has $PMSIZE bytes" \
                       || echo "FAIL: processed_messages.json is too small ($PMSIZE bytes) — abort and investigate."
```

If `processed_messages.json` did not transfer correctly, the bot would re-post every recent paper on the next cron fire — user-visible Slack noise. Verify before proceeding.

We deliberately do **not** copy `browsers/`, `browser-profile/`, or `.venv/` from the original — they are large, regenerable, and may contain stale absolute paths. Browsers and the profile are only needed if the user wants the crawl4ai fallback path; see D.3 for re-installing them when activating the `[fallback]` extra.

**C.4 Recreate the venv at the new location**

```bash
cd /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab
uv sync
.venv/bin/python -c "import slack_sdk, requests, bs4, diskcache, tenacity" 2>&1 | head
```

(For the smoke-test below, the core dependencies are sufficient. The `[fallback]` extras — `crawl4ai`, `ollama`, `pydantic` — are needed for the production fallback chain but are not exercised by `--dry-run`. See D.3 for the production setup.)

**C.5 Smoke-test** (dry-run; no Slack post; no sbatch)

```bash
cd /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab
.venv/bin/python run.py --dry-run --lookback-hours 1
```

Expected: completes without exception. Writes a `dry_runs/<timestamp>.html` artifact. Does **not** post to Slack. Does **not** submit `sbatch`. The fallback chain is gracefully skipped if the optional `[fallback]` extras are absent.

If smoke-test fails: **stop**, do not commit. See **Rollback** below. The original `$BOT` is intact.

**C.6 Commit the submodule registration to the monorepo and push**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git add .gitmodules slack_paperbot_ridder_lab
git commit -m "Add slack_paperbot_ridder_lab as submodule

Moved from /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot/.
Cron is left disabled. The user will re-enable from the new location with:
    cd slack_paperbot_ridder_lab && ./set_cron_job.sh --minutes 10

Origin: docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md"
git push origin main
```

**C.7 Remove the original location** (final irreversible step; only after C.6 succeeded)

```bash
rm -rf /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot
ls /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/
# Expected: only `analysis/` (with empty `dstoker/` inside) remains.
```

The old parent dir `/hpc/compgen/projects/slackbot_fyi_papers_info/` is left in place per the brainstorm decision ("stays as-is unless you say otherwise"). If the user wants it gone too, they'll remove it manually after they're satisfied with the new location.

**C.8 Verify final state**

```bash
cd /hpc/compgen/projects/lab_ai_automation
git submodule status                # 6 entries
git log --oneline                   # 2 commits
ls -la                              # 6 submodule dirs + scaffolding files
```

---

#### Phase D — Execution report & handoff (~15 min)

**D.1 Write the execution report** to `docs/plans/2026-04-26-feat-lab-ai-monorepo-init-plan.execution-report.md`. Contents:

- `git submodule status` output (final).
- `git log --oneline -n 3` for the monorepo and each submodule.
- The exact cron line the user will install when ready: `cd slack_paperbot_ridder_lab && ./set_cron_job.sh --minutes 10` (the script reads `$SCRIPT_DIR` at install time and rewrites the cron line correctly).
- Reminder: at C.7, the original bot location at `/hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot/` was removed. The bot's tracked content remains on its GitHub remote and inside the monorepo submodule. Gitignored runtime state (`.env`, `data/`, `logs/`) was rsync'd at C.3 and now lives only in the monorepo location.

**D.2 Bot-source paths the user will want to refactor for portability** (NOT done by this plan; flagged here per brainstorm direction):

| File | Line | Issue | Suggested fix |
|---|---|---|---|
| `src/upgrades/version_check.py` | 36 | Hardcoded `cd /hpc/compgen/projects/ollama/ollama_run/analysis/dstoker` in an Ollama-server upgrade hint string. | Read `OLLAMA_SCRIPTS_DIR` env var; fallback to a generic `<your install dir>` placeholder string. One-line change. |

All other absolute paths in the bot are inside `docs/` (informational only) or in `config/config.yaml` (already env-var driven via `OLLAMA_SCRIPTS_DIR`, with a placeholder value). `set_cron_job.sh` already uses `$SCRIPT_DIR`; `src/utils/config_loader.py` already uses `Path(__file__).parent.parent`. Net: only one functional path edit is needed for portability, and it's a one-line change the user will make separately.

**D.3 Suggested next steps for the user**:

1. Visit each of the 6 GitHub remotes and verify the initial commit + content look right.
2. Test cloning `--recurse-submodules` from a different machine to confirm participant-onboarding works.
3. **Activate the `[fallback]` extra for the bot before re-enabling cron.** The smoke-test in C.5 only exercises core deps. Production cron uses crawl4ai, ollama, pydantic via the bot's `[fallback]` extra:
   ```bash
   cd /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab
   uv sync --extra fallback
   uv run playwright install chromium --only-shell   # populates browsers/ for crawl4ai
   ```
4. Re-enable cron when ready:
   ```bash
   cd /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab
   ./set_cron_job.sh --minutes 10
   crontab -l   # confirm the line points at the new path
   ```
5. Watch the first 1–2 cron fires under `slack_paperbot_ridder_lab/logs/cron.log` and `logs/slurm-worker-*.{out,err}` in the same dir.
6. Refactor `src/upgrades/version_check.py:36` separately (see D.2).
7. Optionally remove the now-empty parent dir at the old bot location: `rmdir /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/analysis/dstoker /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/analysis /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info /hpc/compgen/projects/slackbot_fyi_papers_info`.
8. Before the retreat: populate `hackathon/automation_ideas.md` with retreat-day-specific items if desired; populate `hackathon/README.md` with date and logistics; consider adding `.agents/skills/` symlinks if useful for participants' coding agents.

**D.4 Hand back control.** No further phases unless the user requests one.

## Alternative Approaches Considered

**Option A — in-place adoption with `git submodule absorbgitdirs`** (rejected). `mv` the bot in, then manually wire it as a submodule via `git config -f .gitmodules submodule.<name>.{path,url}` + `git add <path>` + `git submodule absorbgitdirs <path>` + commit. Avoids the network re-clone. Rejected because:

- Step ordering is fragile; one missed command leaves the parent in an inconsistent state.
- `absorbgitdirs` has version-dependent edge cases.
- The network re-clone is fast (~3 MB tracked content).
- Rollback semantics are murkier — the bot's `.git/` has already been relocated.

**Option B — mv-aside + clone** (rejected). `mv` into monorepo path, then `mv` aside, then `git submodule add` clones into the freed path, then restore runtime state. Considered earlier in this plan. Rejected in favor of Option C because rollback requires moving `_bot_temp_<ts>` back to the original path, which adds a step and a brief window where the bot exists in two places (original gone, temp present, new path present).

**Option C — clone-then-rm-old (CURRENT PLAN).** `git submodule add` clones the bot fresh into the monorepo from its existing remote without moving anything. Runtime state is rsync'd from the still-intact original location. Smoke-test runs from the new location. **Only after the parent commit + push succeeds** does the original location get `rm -rf`'d. Rollback before C.7 is: `git submodule deinit + rm -rf` of the new path; the original is untouched and the user can re-enable cron there if they abort. Cost: ~350 MB exists in two places for ~10 min during the smoke-test window — trivial on HPC.

The brainstorm's preference was "mv preserving .git, then add as submodule" — Option C preserves the brainstorm's intent (old location ends up empty, no source edits, registered as submodule) while being strictly safer because the rollback target is intact until the very last step.

**Option D — subtree merges** (rejected at brainstorm). Locks history into the monorepo and complicates the bot's continued use of its own remote.

**Option E — plain monorepo, no submodules** (rejected at brainstorm). Loses the bot's existing remote and history.

**Option F — sequence Phase B and Phase C in parallel via background jobs** (rejected). The pushes are independent in principle, but interleaving them increases the cognitive load of debugging a partial failure. Sequential is easier to reason about and only adds a few minutes of wall-clock.

## System-Wide Impact

### Interaction graph

- **`git submodule add` triggers** an HTTPS clone from each remote into `.git/modules/<name>/`, plus a working-tree checkout into the submodule path. Side effects: creates/updates `.gitmodules`, stages a gitlink at the submodule path, stages `.gitmodules`.
- **Cloning the bot via `git submodule add`** triggers an HTTPS clone of the bot's remote into `lab_ai_automation/slack_paperbot_ridder_lab/`, plus a working-tree checkout of `origin/main`. Side effects: appends a `[submodule "slack_paperbot_ridder_lab"]` entry to `.gitmodules`, stages a gitlink, creates `.git/modules/slack_paperbot_ridder_lab/`.
- **`rsync` of runtime state** at C.3 copies `config/.env`, `data/`, `logs/` from the original location to the new one. The dispatcher's `data/dispatcher.lock` is just an empty flock target — copying it is harmless because nothing has it open (cron is off; no SLURM jobs).
- **`uv sync` at C.4** rebuilds the venv with absolute paths correct for the new location. The original venv at the old location is untouched until C.7's `rm -rf`.
- **`rm -rf` of the original at C.7** deletes the bot's old working tree including its `.git/`, the old `.venv/`, `browsers/`, `browser-profile/`. This is the only step that destroys data — and at this point all tracked content lives on the GitHub remote and inside the monorepo, and runtime state lives inside the monorepo.
- **Submodule registration commit** propagates to the parent's HEAD; cloning with `--recurse-submodules` triggers all six checkouts.

### Error & failure propagation

| Layer | Error class | How it surfaces | How we handle |
|---|---|---|---|
| Auth | 401 / `Permission denied` on push | `git push` exits non-zero | A.3 catches up front; if it slips through, halt the per-stub recipe at the `push` step before anything is `submodule add`'d |
| Remote state | `! [rejected] main -> main (non-fast-forward)` | `git push` rejected | A.3 catches; never use `--force` |
| Network | clone interrupted | `git submodule add` partial state | `git submodule deinit -f <path> && git rm -f <path> && rm -rf .git/modules/<name>` then retry |
| Filesystem | `mv` cross-device fallback | unexpected slowness; no error | not expected; both source and dest are on `/hpc/compgen/`; if it happens, use `cp -a + rm -rf` instead |
| Bot smoke-test | `ImportError`, `ConnectionRefusedError`, etc. | `run.py --dry-run` exits non-zero | Phase C does not commit; rollback per the Rollback section |

### State lifecycle risks

- The bot's runtime state (`config/.env`, `data/processed_messages.json`, `logs/`) is gitignored. **It is NOT carried by git.** The plan rsync's it from the original location to the new one in C.3 — this works precisely because the original location remains intact through C.6. If the rsync is skipped or `processed_messages.json` does not transfer correctly, the bot at the new location would re-post every recent paper on the next cron fire — user-visible Slack noise. **Mitigation:** C.3 includes an explicit size-check on `processed_messages.json` (must be > 100 bytes; Phase 0 saw ~2696). The smoke-test in C.5 does not catch this (dry-run doesn't post), so the size-check is the safeguard.
- `.venv/` is regenerated by `uv sync` at C.4, so any in-flight installs from the old venv are lost. No risk: `pyproject.toml` + `uv.lock` capture the exact dep set.
- The bot's GitHub remote is not modified.
- Until C.7 runs, the bot exists at TWO locations on disk simultaneously. This is intentional: the original is the rollback target. After C.7 succeeds, only the new location has the bot.

### API surface parity

- The bot's CLI (`run.py --dry-run`, `run.py --only-parent-ts`, `set_cron_job.sh`, `scripts/delete_bot_messages.py`) is unchanged.
- The submodule registration pattern matches what `git clone --recurse-submodules` does for any future user — verifiable via Integration test #1 below.

### Integration test scenarios

1. **Fresh clone of the monorepo from a different machine.** `git clone --recurse-submodules https://github.com/DieStok/ridder_lab_ai_automation.git` in a scratch dir. Expected: all 6 tools checked out; each has `.git` linked to `.git/modules/<name>/` in the parent; `cd hackathon && uv sync` succeeds. Verifies the participant-onboarding story.
2. **Cron re-enable + first fire from the new location.** User runs `./set_cron_job.sh --minutes 10`, waits 10 min, checks `logs/cron.log`. Expected: dispatcher claims `dispatcher.lock`, fetches Slack, processes new messages or exits cleanly. Verifies path correctness end-to-end.
3. **Independent push from the bot's submodule location.** User makes a small commit in `slack_paperbot_ridder_lab/` (e.g., a comment), `git push` from inside that submodule. Expected: pushes to `github.com/DieStok/slack_paperbot_ridder_lab`. Verifies the move did not corrupt the remote linkage.
4. **Re-clone an individual submodule.** `git clone https://github.com/DieStok/ridder_lab_deep_research.git` somewhere. Expected: works as a standalone repo with the stub README, env example, .gitignore. Verifies submodules are self-contained.
5. **Submodule-pointer bump from the parent.** After the user makes a commit inside `slack_paperbot_ridder_lab/` and pushes, then from the parent: `git submodule update --remote slack_paperbot_ridder_lab && git add slack_paperbot_ridder_lab && git commit -m "Bump slack_paperbot_ridder_lab to <sha>"`. Verifies the submodule update workflow.

## Acceptance Criteria

### Functional

- [x] `/hpc/compgen/projects/lab_ai_automation/.git/` exists.
- [x] `git -C /hpc/compgen/projects/lab_ai_automation submodule status` lists exactly 6 entries.
- [x] Each of 7 GitHub remotes has a `main` branch with the expected initial content (visible on github.com).
- [x] `/hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot/` no longer exists (removed at C.7); its working tree is now at `/hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab/`.
- [x] `cd lab_ai_automation/slack_paperbot_ridder_lab && uv run python run.py --dry-run --lookback-hours 1` exits 0 and writes a dry-run artifact.
- [x] `cd lab_ai_automation/slack_paperbot_ridder_lab && stat -c%s data/processed_messages.json` returns a size > 100 bytes matching pre-move state (within rounding). (2696 bytes, matches pre-move.)
- [x] `crontab -l | grep slack-bot` returns nothing (cron remains disabled per user direction).
- [ ] `cd lab_ai_automation/hackathon && uv sync` succeeds (validates the kitchen-sink venv kit resolves). Deferred to user verification before the retreat.

### Non-functional

- [x] No bot Python source file has been edited. `git -C slack_paperbot_ridder_lab diff origin/main..HEAD -- src/` is empty.
- [x] No `--force` push to any remote.
- [x] No history rewrite on any remote.
- [x] The execution report at `docs/plans/2026-04-26-feat-lab-ai-monorepo-init-plan.execution-report.md` exists.

### Quality gates

- [x] STOP POINT respected: user explicitly approved before Phase C began.
- [x] Smoke-test passed before C.6 (the parent commit) ran.
- [x] Original bot location was removed only at C.7, after C.6 succeeded.

## Dependencies & Prerequisites

- `git` ≥ 2.30 (for `init.defaultBranch=main`).
- `uv` ≥ 0.4 (for `uv sync` portability).
- `gh` CLI optionally helpful for confirming GitHub state; not required.
- The user has push rights to all 7 GitHub repos (verified A.3).
- The 4 fresh stub remotes are empty (verified A.3).
- HPC submit node has internet access to GitHub (already used by the bot today).

## Risk Analysis & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Push auth fails mid-Phase-B | Low | Medium | A.3 pre-flight; halt before any irreversible step |
| Stub repo has auto-init README | Low | Medium | A.3 detects; user deletes + recreates, or approves `fetch + merge` |
| Smoke-test fails at C.5 | Low | High | Original bot location intact through C.6; rollback per the **Rollback** section is `git submodule deinit + rm -rf` of the new path |
| `processed_messages.json` not transferred → bot re-posts | Low (rsync is reliable) | High (Slack noise) | C.3 includes explicit size-check (must be > 100 bytes) before proceeding |
| `.venv` regen fails (uv error / network) | Low | Medium | Pre-flight A.6; fix `uv sync` separately; rollback as above |
| Cron accidentally re-enabled at old path before C.7 | Low | Medium | Cron stays off through the whole plan per user direction. After C.7 the old path doesn't exist, so cron pointing there silently fails. Belt-and-suspenders: only install the cron line from the new path's `set_cron_job.sh` |
| Submodule order causes parent-ref divergence | Very low | Low | `submodule add` and commit are serial, no concurrent edits |
| Network failure mid-clone | Low | Low | `submodule deinit -f` recovery in error-propagation table |
| Disk briefly holds bot in two places between C.2 and C.7 | Certain | Negligible | ~700 MB peak on `/hpc/compgen/`; trivial |

## Rollback

### If Phase B fails partway

State: monorepo has some submodules wired, some not; some remotes pushed, some not.

1. The remotes that received pushes stay pushed (no rollback needed; the user can delete them on GitHub if redoing).
2. To redo the local monorepo: `rm -rf /hpc/compgen/projects/lab_ai_automation/.git /hpc/compgen/projects/lab_ai_automation/.git/modules`, restore `.gitmodules` to its comment-only stub from `git show <hash>:.gitmodules` (if there is no commit yet, just delete it), and start Phase B over. The scaffolded files survive (they are NOT inside the `_migrate_*_tmp` dirs by this point — those got `rm -rf`'d after a successful push, or never moved if the move-step itself failed).
3. Or, more surgically: identify the partial state and finish from where it stopped.

### If Phase C smoke-test fails (C.5)

State: bot has been cloned into the monorepo and runtime state rsync'd; `--dry-run` failed; C.6 (commit) NOT yet made; original `$BOT` location is intact and runnable.

1. **Don't commit.** The monorepo's working tree has staged but uncommitted changes (`.gitmodules` + `slack_paperbot_ridder_lab`).
2. Diagnose: read stderr; check `slack_paperbot_ridder_lab/logs/`; verify Ollama endpoint is reachable from the smoke-test host; verify `.env` was restored correctly; verify `data/processed_messages.json` size matches pre-move.
3. If the bot's code or config is to blame and the user opts in: fix it inside the submodule. Re-run C.5. If it now passes, commit per C.6.
4. If unrecoverable, full rollback:
    ```bash
    cd /hpc/compgen/projects/lab_ai_automation
    git restore --staged .gitmodules slack_paperbot_ridder_lab 2>/dev/null
    git submodule deinit -f slack_paperbot_ridder_lab
    git rm -f slack_paperbot_ridder_lab 2>/dev/null
    rm -rf slack_paperbot_ridder_lab
    rm -rf .git/modules/slack_paperbot_ridder_lab
    git config -f .gitmodules --remove-section submodule.slack_paperbot_ridder_lab 2>/dev/null
    # The original bot location at /hpc/compgen/projects/slackbot_fyi_papers_info/.../slack-paper-bot
    # is untouched. The user can re-enable cron there if they want to abort the move entirely.
    ```
5. The original `$BOT` location is the source of truth for the rollback. Nothing was moved or deleted from it.
6. Cron remained disabled throughout. No "bot ran on bad state" concern during rollback.

### If Phase C committed successfully (C.6) but the bot misbehaves under cron later

The monorepo commit is preserved on GitHub. Likely the issue is environmental (Ollama endpoint, `.env` tokens, `[fallback]` extras not installed per D.3 step 3) rather than the move itself. Diagnose first; do not assume rollback is necessary. If a real rollback is needed and the original location was already removed at C.7:
- The bot's full git history is on `github.com/DieStok/slack_paperbot_ridder_lab`. Re-clone anywhere: `git clone https://github.com/DieStok/slack_paperbot_ridder_lab.git /hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot`.
- Restore runtime state from the monorepo copy: `cp /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab/config/.env <restored>/config/.env`, etc.
- `git revert` the parent commit if removing the submodule from the monorepo is desired.

## Resource Requirements

- Compute: minimal.
- Network: ~10 MB total (5 small repos + the bot's tracked content).
- Time: 1.5–2 hours including deliberate verification at each step. The STOP POINT for user approval is the gating wait.
- People: just `dstoker`. No coordinator needed.

## Future Considerations

- When `slack_paperbot_ridder_lab` is refactored for portability, the patterns extracted will become new building blocks (e.g. `cron_to_sbatch_dispatcher/`, `ollama_endpoint_discovery/`). Only `automation_building_blocks/INDEX.md` needs an entry per new block.
- Promoting per-team retreat submissions into their own repos: lift `submissions/team_<n>/` out, `git init`, push, `git submodule add` if the work is mature. Same pattern as Phase B.
- Adding new submodules later: one-liner `git submodule add <url> <path>` operation — no monorepo-wide changes.
- An `.agents/skills/` directory at the monorepo root and at `hackathon/` (with `.claude/skills/<name>` symlinks) would replicate the geo_harmonizer skill-distribution pattern. Out of scope for this plan; trivial to add later.

## Documentation Plan

This plan IS the documentation. After execution, the following files reflect the change:

- `lab_ai_automation/README.md` — already current; lists all 6 tools.
- `lab_ai_automation/.gitmodules` — populated by `git submodule add`.
- `lab_ai_automation/docs/plans/2026-04-26-feat-lab-ai-monorepo-init-plan.md` — this file.
- `lab_ai_automation/docs/plans/2026-04-26-feat-lab-ai-monorepo-init-plan.execution-report.md` — written at Phase D.

No other docs need updating.

## Sources & References

### Origin

- **Brainstorm document:** [`docs/brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md`](../brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md). Decisions carried forward verbatim:
  1. Submodule strategy for all 6 tools with named GitHub remotes.
  2. Hackathon repo: `https://github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git`, branch-per-team contribution model, AGENTS.md + symlink pattern from `geo_harmonizer`.
  3. Bot move: `mv` preserving `.git/`, then submodule-add (refined here to **clone-then-rm-old**: the bot's `.git` is rebuilt fresh from its remote rather than physically moved, but the brainstorm's intent — old location empty, no source edits, registered as submodule — is preserved. See Alternative Approaches Considered for why).
  4. Cron stays disabled; user re-enables manually.
  5. No edits to the bot's Python source.

### Internal references

- Existing live bot (pre-move): `/hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot/` — remote `https://github.com/DieStok/slack_paperbot_ridder_lab.git`.
- AGENTS pattern reference: `/hpc/compgen/projects/llm_GEO_project/harmonia_metadata_agent/analysis/dstoker/geo_harmonizer/{AGENTS.md, .claude/CLAUDE.md, .agents/}`.
- Bot's Phase 0 inspection notes (from the brainstorm-session conversation): cron details, sbatch architecture, Ollama config, secrets layout, hardcoded-path inventory.
- Original prompt: `docs/prompt_for_claude_code_lab_ai_monorepo_updated.md`.

### External references

- `git submodule` documentation: https://git-scm.com/docs/git-submodule
- `uv` documentation: https://docs.astral.sh/uv/
- AGENTS.md cross-tool standard: https://agents.md/

### Related work

- Brainstorm session: 2026-04-26 (this monorepo's first brainstorm).
- Phase 0 inspection of the live bot: prior session's first turn.
