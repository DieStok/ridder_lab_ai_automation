# AGENTS.md — lab_ai_automation

> This file is the source of truth for agent guidance in this monorepo. `.claude/CLAUDE.md` is a symlink to it. Tools that read `AGENTS.md` (Codex, Cursor, Aider, etc.) and tools that read `CLAUDE.md` (Claude Code) all see the same content.
>
> Curated, lab-specific, opinionated. Not generic agent boilerplate. (Recent ETH Zurich research found that naïve LLM-generated context files hurt agent performance in 5 of 8 settings — keep this concrete.)

## Repo purpose

`lab_ai_automation` aggregates the De Ridder Lab's AI automation tools as submodules so contributors and coding agents can see the whole constellation at once. Tool code is platform-agnostic; deployment specifics (cron+sbatch, systemd, Docker, cloud) live in `automation_building_blocks/deployment_recipes/`, never inside a tool.

## Absolute rules

1. **Never commit `.env`, model weights, or bulk data.** `.env.example` is committed; `.env` is not. Bulk data goes under `LAB_AI_DATA_DIR` or external storage.
2. **Never force-push or rewrite history on any submodule.** `slack_paperbot_ridder_lab` is live in production with a shared remote; treat all submodules with the same caution.
3. **No hardcoded absolute paths in tool source code.** Use the documented env vars: `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL`. Defaults: `./data`, `./logs`, `http://localhost:11434`.
4. **Directory names: lowercase, underscores only.** No dashes, dots, or spaces.
5. **When you change a tool's behavior, update its README and `.env.example` in the same commit.** A drift-free onboarding story matters more than perfect implementation.
6. **Never modify `slack_paperbot_ridder_lab/` source.** It is live, HPC-coupled, and the maintainer (`dstoker`) will refactor it for portability separately. The only acceptable edits in that submodule are the user's own.

## Platform awareness

This repo runs in three places: laptops (developers), HPC cluster (batch compute), and (eventually) a small cloud VM. Don't assume any specific platform inside tool code.

If you need to do something platform-specific (submit an `sbatch` job, add a systemd unit, write a Dockerfile), put it in a deployment recipe under `automation_building_blocks/deployment_recipes/`. The tool's source code stays platform-agnostic.

**HPC policy worth knowing:** the cluster maintainer is uncomfortable with long-lived internet-facing processes on submit nodes. Slack listeners and external-API pollers should not be assumed to run on HPC. HPC is one possible deployment target, not the default.

## Environment expectations

- `uv` is installed and is the only Python package manager used here. Per-tool `pyproject.toml` + `uv.lock` + `.venv/`.
- Python `>=3.11`.
- Ollama may run on `localhost` (laptop), on a GPU compute node (HPC, started via `sbatch`), or on a remote endpoint (cloud). Always read `OLLAMA_BASE_URL`. Never hardcode `http://localhost:11434`.
- The lab's HPC SLURM account is `compgen`. Submit nodes are `hpcs05` / `hpcs06`. Compute nodes only via `sbatch`. SLURM logs use `%x_%j` naming.

## Per-tool overrides

Each submodule may have its own `AGENTS.md` (and `.claude/CLAUDE.md` symlink). Inside a submodule's tree, that file's instructions take precedence over this one. The hackathon submodule has its own AGENTS.md aimed at retreat participants — agents working there should follow it.

## Where reusable patterns live

`automation_building_blocks/INDEX.md` is the catalogue. Before writing a new Slack handler, deployment script, or LLM client, **check the index and prefer copying an existing block**. If you copy and adapt, leave a one-line attribution comment at the top of the file noting which block it came from. If a pattern is needed in two or more tools, lift it into a new building block.

Building blocks are **read-and-copy, not import**. They have no `pyproject.toml` at the root and are not installed as packages.

## Secrets

Each tool maintains its own `.env` (gitignored) and `.env.example` (committed, no values). Common keys you'll see across tools:

- `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `SLACK_CHANNEL_ID` — Slack apps.
- `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` — remote LLM access (optional; default to local Ollama).
- `TAVILY_API_KEY` — web search for AI agents.
- `NCBI_API_KEY` — polite-pool E-utilities access.
- `CONTACT_MAILTO` — polite-pool contact for CrossRef/OpenAlex/NCBI.

If you add a new tool that needs a new secret, document it in that tool's `.env.example` AND mention it in the tool's README.

## Commit conventions

- Short, imperative subject line. Optional body for the why.
- When working inside a submodule, the submodule's commits are scoped to that tool. Prefix is optional but useful for cross-tool clarity: `[paper_bot] fix Ollama timeout`.
- The monorepo's commits typically just bump submodule pointers: `Bump deep_research_bot to <short_sha>` or `Add summarize_bot as submodule`.
- No emoji unless the user explicitly asked.

## What to do when uncertain

- Read before writing. Skim the relevant submodule's README and AGENTS.md (if any) before editing files.
- Never delete tests to make CI pass.
- Never comment out failing logic without surfacing it as an open question to the user.
- If a lab convention isn't documented in this repo, ask the user. Don't invent one.

## Reading order for a new agent

If you've just been pointed at this repo and have no context:

1. This file.
2. `README.md` for the human-facing layout.
3. `automation_building_blocks/INDEX.md` to know what's already been written.
4. The README of whichever submodule you've been asked to work in.
5. The submodule's own `AGENTS.md` if it has one.
