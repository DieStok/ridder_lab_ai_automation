---
date: 2026-04-26
status: ready-for-plan
author: dstoker
type: scaffolding
---

# Brainstorm: Scaffold the De Ridder Lab AI Automation Monorepo

## What We're Building

A monorepo at `/hpc/compgen/projects/lab_ai_automation/` that aggregates the lab's AI-automation tools as git submodules. The monorepo itself is a git repo (`https://github.com/DieStok/ridder_lab_ai_automation.git`). Six submodules:

- `slack_paperbot_ridder_lab/` — the lab's existing live paper-FYI bot, moved in as-is. Source code stays untouched. Cron stays disabled until the user re-enables it from the new path.
- `deep_research_bot/` — stub. Slack-slash-command-driven research reports.
- `summarize_bot/` — stub. On-demand paper summary + adversarial review.
- `coding_agent_installer/` — already exists as a real repo; will be updated for sandboxing later.
- `automation_building_blocks/` — read-and-copy reference patterns + deployment recipes.
- `hackathon/` — the Spring 2026 retreat playground; teams ship into `submissions/team_<name>/`.

Each submodule has its own GitHub remote, listed in `.gitmodules`.

The first immediate use is the **Spring 2026 lab retreat hackathon**: distribute the monorepo (with `--recurse-submodules`) to participants who hack on automation ideas using laptops + coding agents.

## Why This Approach

**Submodules over a flat monorepo.** The existing paperbot has a shared GitHub remote and active commits; rewriting its history into a flat monorepo is unacceptable. Submodules preserve each tool's independent identity, history, and remote, while still letting `git clone --recurse-submodules` produce a single working tree where coding agents see everything.

**Tools are platform-agnostic; deployment is decoupled.** The HPC maintainer is uncomfortable with long-running internet-facing bots on submit nodes, so HPC cannot be the assumed runtime. Tool source reads `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL` from env with sensible defaults; platform-specific deployment notes live as recipes under `automation_building_blocks/deployment_recipes/`.

**`AGENTS.md` source-of-truth + `.claude/CLAUDE.md` symlink.** Following the pattern in `/hpc/compgen/projects/llm_GEO_project/harmonia_metadata_agent/analysis/dstoker/geo_harmonizer/`. Cross-tool agent compatibility (Claude, Codex, Cursor, Aider) without duplicating content. Curated and concrete — the ETH Zurich finding that naïve LLM-generated context files hurt agent performance in 5 of 8 settings is the motivation for keeping these files specific.

**Hackathon ships its own shared `uv` venv.** The retreat is hours-to-days; teams cannot afford to debug dependency resolution. A single `uv sync` in `hackathon/` populates `.venv/` with the common toolkit (Slack, LLM clients, LangGraph, web/PDF parsing, data tools). Teams can still create their own per-submission venv if they need isolation; default is the shared one.

**Build-block: `slack_app_skeleton/` over an extracted block from the live bot.** The live paperbot is HPC-coupled and will be refactored later; extracting from it would freeze HPC assumptions into the building blocks library. A fresh, portable Socket-Mode skeleton is cleaner.

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Monorepo location | `/hpc/compgen/projects/lab_ai_automation/` | Matches the prompt's name; HPC convention is honored at the *root* level only. Internal layout is flat. |
| Monorepo remote | `https://github.com/DieStok/ridder_lab_ai_automation.git` | Provided by user. |
| Git strategy | Submodules for all six tools | Preserves remotes and histories; supports independent contributor work. |
| Hackathon repo | `https://github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git` | Provided by user. Submodule of the monorepo. |
| Hackathon contribution model | Branch-per-team; submissions/team_\<name\>/ | Trusted lab members; forks add friction; branches keep work visible. |
| Hackathon distribution | Clone the parent monorepo `--recurse-submodules` (preferred); standalone clone of just the hackathon repo also works | Coding agents benefit from monorepo context; standalone is the lower-friction path for participants who don't need it. |
| Existing bot dir name | `slack_paperbot_ridder_lab/` | Matches its GitHub remote name. The other tools use shorter underscore-only local dir names while their remote names are longer (handled by `git submodule add <url> <short_dir>`). |
| Bot move | `mv` preserving `.git/`, then `git submodule add <remote> slack_paperbot_ridder_lab` | Smallest reversible operation; `.git` and remote come along atomically. |
| Cron after move | **DISABLED** (already removed in this session); user re-enables manually after smoke-test | Safety. `set_cron_job.sh --minutes 10` from the new path will reinstall the tagged line. |
| Bot source code edits | **NONE.** | User refactors the bot for portability separately. Phase 5 report enumerates absolute paths the user will want to address. |
| Stub READMEs | Full versions per the prompt template (40–120 lines each) | Hackathon participants and future contributors benefit from grounded design briefs. |
| Building blocks first content | `slack_app_skeleton/` (fresh, portable) + four deployment recipes (`local_dev`, `hpc_cron_sbatch`, `systemd_timer` placeholder, `cloud_vm` placeholder) | Two recipes fleshed out (`local_dev`, `hpc_cron_sbatch`); two are placeholders awaiting decisions. |
| Hackathon shared venv | Generous "kitchen-sink" `pyproject.toml` with Slack, Ollama, LiteLLM, LangChain, LangGraph, PDF tools, search, data tools, Jupyter, pytest. Optional extras (`crawl`, `viz`, `bio`) for heavier deps. | Time-boxed retreat; minimize first-hour friction. |
| `automation_ideas.md` content | The 7 user-supplied seed ideas as bullets, plus an "Add your own" section | Per user direction. |
| Old empty dir cleanup | Move prompt md into `lab_ai_automation/docs/`; delete `/hpc/compgen/projects/slack_bots_ridderlab/` at the END of all work (so the working shell doesn't lose its CWD mid-execution again) | Safety lesson learned during this session. |

## Repo Layout (final)

```
lab_ai_automation/
├── README.md                          # human-facing
├── AGENTS.md                          # source-of-truth agent guide
├── .claude/CLAUDE.md → ../AGENTS.md   # symlink
├── .gitignore
├── .gitmodules                        # populated by /ce:plan
├── docs/
│   ├── brainstorms/2026-04-26-lab-ai-monorepo-scaffold-brainstorm.md   # this file
│   └── prompt_for_claude_code_lab_ai_monorepo_updated.md               # historical
│
├── slack_paperbot_ridder_lab/   →  github.com/DieStok/slack_paperbot_ridder_lab.git
├── deep_research_bot/           →  github.com/DieStok/ridder_lab_deep_research.git
│   ├── README.md (stub design brief)
│   ├── .env.example
│   └── .gitignore
├── summarize_bot/               →  github.com/DieStok/ridder_lab_summarization_bot.git
│   ├── README.md (stub design brief)
│   ├── .env.example
│   └── .gitignore
├── coding_agent_installer/      →  github.com/DieStok/coding_agent_installer_ridder_lab.git
│
├── automation_building_blocks/  →  github.com/DieStok/ridder_lab_LLM_automation_building_blocks.git
│   ├── README.md
│   ├── INDEX.md
│   ├── .gitignore
│   ├── slack_app_skeleton/
│   │   ├── README.md, app.py, pyproject.toml, .env.example
│   └── deployment_recipes/
│       ├── README.md
│       ├── local_dev.md
│       ├── hpc_cron_sbatch.md
│       ├── systemd_timer.md (placeholder)
│       └── cloud_vm.md (placeholder)
│
└── hackathon/                   →  github.com/DieStok/ridder_lab_retreat_ai_hackathon_2026.git
    ├── README.md (10-min start recipe + ground rules)
    ├── AGENTS.md (retreat-specific agent rules)
    ├── .claude/CLAUDE.md → ../AGENTS.md
    ├── automation_ideas.md (7 seeds)
    ├── pyproject.toml (shared kitchen-sink kit)
    ├── .env.example
    ├── .gitignore
    └── submissions/
        └── README.md
```

## Current Scaffolding Status (as of this brainstorm)

**Already on disk** (this session created the static structure; no git ops yet):

- All directories above.
- All `README.md`, `AGENTS.md`, `.env.example`, `.gitignore`, `pyproject.toml`, `.md` recipe files, `app.py`, `.gitmodules` (commented stub), `INDEX.md`, `automation_ideas.md`, `submissions/README.md`.
- Both `.claude/CLAUDE.md` symlinks (root and hackathon).
- The prompt md moved into `docs/`.

**Not yet done** (this is the work for `/ce:plan`):

1. `git init` the monorepo, add the remote, push the scaffolded content as the initial commit.
2. For each of `deep_research_bot`, `summarize_bot`, `automation_building_blocks`, `hackathon`:
   - Move the existing scaffolded content out of the way, `git submodule add <remote_url> <short_dir>` to clone the empty (or existing) GitHub repo, then re-apply the scaffolded content into the new submodule, commit, and push.
   - For `coding_agent_installer/`: just `git submodule add` (it has existing content; nothing to write).
   - For `slack_paperbot_ridder_lab/`: see "Move the live bot" below.
3. **Move the live bot** (Phase 3 of the original prompt; STOP POINT 2 — get user approval first):
   - The bot is currently at `/hpc/compgen/projects/slackbot_fyi_papers_info/slackbot_fyi_papers_info/slack-paper-bot/`.
   - Cron has already been removed in a prior session turn; SLURM queue confirmed empty.
   - `mv <old_path> /hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab` — preserves `.git/` and remote.
   - `cd /hpc/compgen/projects/lab_ai_automation && git submodule add <remote> slack_paperbot_ridder_lab` (or the equivalent that adopts an existing on-disk repo).
   - Smoke-test: `cd slack_paperbot_ridder_lab && uv run run.py --dry-run` (or whatever the bot supports). Do NOT post to Slack.
   - **Cron stays disabled.** Re-enable is the user's call: `./set_cron_job.sh --minutes 10` from the new path.
4. Delete the now-empty `/hpc/compgen/projects/slack_bots_ridderlab/` (still exists as a placeholder so the shell can keep working — this is the LAST step).
5. Phase 5 report: tree, git status across all submodules, list of absolute paths in the bot's source the user will want to refactor for portability, suggested next-step list.

## Open Questions

(Resolved during the brainstorm session; kept here for record.)

- ~~Two example tools or stub-only?~~ → Full stubs per the prompt for all four planned tools; the existing bot is the second "example."
- ~~Submodules for the stubs?~~ → Yes, all six are submodules. URLs provided.
- ~~Hackathon distribution model?~~ → Submodule + branch-per-team + `submissions/team_<name>/`.
- ~~Hackathon AGENTS pattern?~~ → Replicate the geo_harmonizer pattern: `AGENTS.md` source-of-truth, `.claude/CLAUDE.md` symlink.
- ~~Bot move scope?~~ → `mv` preserving `.git/`; cron stays off; no source edits.
- ~~Building block content for v1?~~ → `slack_app_skeleton/` + four recipe files (two fleshed, two placeholders).
- ~~Existing dir cleanup?~~ → Delete `slack_bots_ridderlab/` at the END (after CWD-hostage lesson).

## Outstanding Decisions for Later (NOT blocking /ce:plan)

These are noted in the relevant stub READMEs as "open questions" and don't block scaffolding:

- Knowledge-store backend for `deep_research_bot` (Obsidian-only vs. Obsidian + Neo4j).
- Cloud VM target for the long-running bot listeners.
- Per-user vs. shared install model for `coding_agent_installer`.
- Paywalled-PDF handling for `summarize_bot`.
- Refactoring the live paperbot to read `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL` from env (the user has explicitly committed to doing this themselves, separately).

## Handoff to /ce:plan

`/ce:plan` should produce an executable plan that performs items 1–5 in **Current Scaffolding Status → Not yet done** above. Stop at the bot-move step for explicit user approval (it is the only irreversible operation; the rest is git-init / git-submodule-add which is reversible by deleting `.git/` and `.gitmodules`).

The plan should include:

- Exact `git` commands per submodule, in dependency order (init monorepo first, then add submodules one at a time).
- A pre-flight check that the user has push rights to all seven GitHub repos (or a graceful degradation: init local + remote add but don't push).
- A smoke-test recipe for the moved bot before any cron consideration.
- A rollback plan for the bot move (it boils down to `mv` back, since `.git/` came with it).
- The final Phase 5 report content as a deliverable.

The plan should NOT include:

- Any modification to the bot's Python source (out of scope; user does this separately).
- Re-enabling cron (out of scope; user does this manually after they're satisfied).
- Any push to GitHub of `coding_agent_installer/` (it has existing content; we just clone it via submodule-add).
