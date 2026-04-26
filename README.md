# lab_ai_automation

A monorepo for the De Ridder Lab's AI-based automation and research tooling. Each tool is its own git repo (with its own GitHub remote) but lives here as a submodule so contributors can see everything in one place, share patterns, and bring a coding agent up to speed on the lab's full automation surface in one clone.

The monorepo is portable. Tool code does not assume any particular platform: every tool runs on a developer laptop with a `.env` file, on the UMC Utrecht HPC cluster behind SLURM, or on a small cloud VM, with platform-specific concerns living in deployment recipes rather than baked into source.

## Why portable

The HPC is excellent for batch compute but is **not a good home for long-running internet-facing processes** — the cluster maintainer is uncomfortable with bots that hold open connections to Slack or poll external APIs from the submit nodes. Laptops are fine for hacking but aren't always available. A small cloud VM (Google Cloud, Sprite, etc.) is the most likely long-term home for the bot listeners. Treating the platform as a deploy-time choice rather than a build-time assumption keeps every tool runnable in all three places.

## Directory layout

```
lab_ai_automation/
├── README.md                            # this file
├── AGENTS.md                            # source-of-truth agent guidance
├── .claude/CLAUDE.md → ../AGENTS.md     # symlink for Claude Code
├── docs/                                # repo-level docs, brainstorms, plans
│
├── slack_paperbot_ridder_lab/           # submodule; live paper-FYI bot (HPC-coupled, will be refactored)
├── deep_research_bot/                   # submodule; stub
├── summarize_bot/                       # submodule; stub
├── coding_agent_installer/              # submodule; existing repo, sandboxing rework planned
├── automation_building_blocks/          # submodule; reference patterns + deployment recipes
└── hackathon/                           # submodule; lab retreat playground (Spring 2026)
```

## Conventions all tools follow

- **No hardcoded absolute paths in source code.** Use `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL` env vars with sensible defaults (`./data`, `./logs`, `http://localhost:11434`).
- **Per-tool, self-contained `uv` venv.** Each tool has its own `pyproject.toml` and `uv.lock`; `uv sync` in the tool dir creates `.venv/`. No shared global venv.
- **Secrets in `.env`** (gitignored). `.env.example` is committed and lists the required keys with no values.
- **Logs default to stdout/stderr.** Optional file logging via `LAB_AI_LOG_DIR`. Same code works under cron, systemd, and Docker.
- **Lowercase, underscore-only directory names.** No dashes, dots, or spaces.
- **Deployment is decoupled from the tool.** The same entry point (e.g. `python -m mybot.run_once`) is invoked by HPC cron+sbatch, by a systemd timer on a VM, by Docker, or by a developer at the terminal. Recipes for each live in `automation_building_blocks/deployment_recipes/`.

## Running a tool

### Locally (laptop or interactive HPC session)

```bash
cd <tool>
cp .env.example .env       # then fill in tokens
uv sync                    # creates .venv inside the tool dir
uv run python -m <tool>.run_once
```

Most tools also expose a CLI script via `pyproject.toml` (`uv run <tool>` or `uv run paper-bot` etc.). Read the tool's own README.

### On the HPC cluster

Heavy compute belongs in `sbatch` jobs. Long-running listeners do not belong on submit nodes — see the deployment notes. The reference cron+sbatch pattern is in `automation_building_blocks/deployment_recipes/hpc_cron_sbatch.md`.

### On a cloud VM

Planned. See `automation_building_blocks/deployment_recipes/cloud_vm.md` (currently a placeholder; will be filled in once a target VM is chosen).

## How to clone

The whole constellation, with every tool checked out:

```bash
git clone --recurse-submodules https://github.com/DieStok/ridder_lab_ai_automation.git
```

Just one tool, standalone:

```bash
git clone https://github.com/DieStok/<tool_remote>.git
```

If you forgot `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## How to add a new tool

1. Create a GitHub repo for it under your account or `UMCUGenetics`.
2. From the monorepo root: `git submodule add <remote_url> <local_dir>` (use a short underscore-only local name; the remote name can be longer).
3. Inside the new dir, copy `automation_building_blocks/slack_app_skeleton/` if it's a Slack bot, otherwise `uv init` fresh. Add a `.env.example` and a `README.md` matching the conventions above.
4. Commit at the monorepo root: `Add <tool> as submodule`.

## Building blocks

`automation_building_blocks/` is a library of reference patterns and deployment recipes. **Read-and-copy, not import.** Each block stands alone with its own mini-README explaining what to change when copying. See `automation_building_blocks/INDEX.md` for the catalogue.

## Current tools

| Name | Status | One-line description |
|---|---|---|
| `slack_paperbot_ridder_lab` | live | Cron-driven Slack bot that enriches paper links posted in `#fyi-papers` with metadata, summaries, and code/data links. |
| `deep_research_bot` | stub | On-demand multi-source research reports triggered by `/deep_research`. |
| `summarize_bot` | stub | On-demand paper summaries plus adversarial review triggered by `/summarize`. |
| `coding_agent_installer` | in development | Provisions Claude Code (and others) on a host with lab defaults pre-wired. |
| `automation_building_blocks` | seeded | Reference patterns and deployment recipes for the rest of the constellation. |
| `hackathon` | active (Spring 2026 retreat) | Playground for the lab retreat; teams ship into `submissions/team_<name>/`. |

## Hackathon

For the lab retreat: `hackathon/README.md`. New here? Open that file first; it has a 10-minute start recipe.

## Maintainer

`dstoker` (Dieter Stoker). Primary channel: lab Slack. Issues on the relevant submodule's GitHub repo are also welcome.
