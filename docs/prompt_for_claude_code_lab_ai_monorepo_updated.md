# Claude Code Prompt: Scaffold the De Ridder Lab AI Automation Monorepo

You are a senior research-software engineer scaffolding a new monorepo for the De Ridder Lab's AI-based automation tooling. The repo must be **portable** — cloneable and runnable on the lab's HPC cluster, on a developer's laptop, and (with documented changes) on a future cloud VM such as a Google instance or a Sprite-style host. The immediate destination is an upcoming lab hackathon, so the primary goal is a clean, welcoming substrate that contributors can build on, not a finished product.

You will act autonomously on read-only investigation and reversible scaffolding, but you MUST stop and confirm before any irreversible action (moving the live cron-driven service that exists today; initializing or rewriting git history; deleting anything).

Read the entire prompt before starting. Then follow the phased plan in `<execution_phases>`. You have full file-system access to the cluster; I (the user) do not have that access to this conversation, but I have access to a parallel session with this repo and can answer questions if you surface them.

---

<context>

## The lab and its environments

The De Ridder Lab is a computational genomics group at UMC Utrecht. Lab members work across three kinds of environments and the monorepo must accommodate all of them:

1. **The UMC Utrecht HPC cluster** (SLURM-based, submit nodes `hpcs05`/`hpcs06`). Many lab members have accounts here and the monorepo will physically live here at first. **Important policy update:** the HPC maintainer is uncomfortable with internet-connected bots running on HPC. So while the monorepo itself can be stored on HPC and individual *batch* jobs can run there, **long-running, internet-facing bot processes (Slack listeners, schedulers polling external APIs) should not be assumed to run on HPC**. Treat HPC as one possible deployment target, not the default.
2. **Local developer machines** (Linux/macOS laptops). Hackathon participants will mostly hack here. Anything in the monorepo must be runnable on a laptop with a local Ollama install and a `.env` file.
3. **Cloud VMs** (future). Likely candidates include a small Google Cloud VM or a Sprite-style host (https://sprites.dev/). Not actively scaffolded now, but documented as a target so we don't paint ourselves into a corner.

The HPC has a mandatory directory convention for projects stored there: `/hpc/compgen/projects/<project>/<subproject>/analysis/<username>/...` with lowercase, underscore-only names. The **monorepo root** will respect this — it lives at `/hpc/compgen/projects/lab_ai_automation/` for now (subject to your confirmation in Phase 1) — but the **internal structure** of the monorepo is flat and platform-agnostic. Do NOT nest tools inside `analysis/dstoker/` directories. The monorepo should look the same when cloned to a laptop as when it sits on HPC.

My HPC username is `dstoker`. GitHub: `DieStok`.

## What exists today

One working tool:

- **Paper-FYI Slackbot** at `/hpc/compgen/projects/slackbot_fyi_papers_info/` — a cron-triggered bot that monitors a Slack channel, enriches paper links with metadata and summaries, and posts back. Remote: `https://github.com/DieStok/slack_paperbot_ridder_lab.git`.
- It is **live** (cron is currently scheduled on the submit node). Whatever you do must not break it.
- It is currently **HPC-coupled** — likely hardcoded to paths under `/hpc/compgen/...`, the submit node's cron, and possibly a specific Ollama setup. **I will refactor it to be portable later; you will not.** When you move it into the monorepo, move it AS-IS. Do not change a single line of its Python code. Only update the cron line and any startup script absolute paths that point to its old location.
- Before making changes, inspect this directory in detail and report back: existing directory tree, the `.venv` or other env setup, how cron invokes it, whether it uses `sbatch`, how Ollama is configured, where secrets live, where logs go, and (especially) what absolute paths appear in scripts/configs/cron that would need updating after the move.

## Where we're going

A monorepo that collects all of the lab's AI automation tools. Properties:

1. Each tool is its own git repo with its own remote, so contributors can work on tools independently.
2. The monorepo aggregates these as submodules so the whole constellation can be cloned at once and coding agents can see everything together.
3. Tool code is platform-agnostic. Any platform-specific concerns (HPC paths, cron formats, sbatch templates, systemd units, Docker compose, cloud run configs) live in **deployment recipes** under `automation_building_blocks/deployment_recipes/`, not inside the tools themselves.
4. Configuration is driven by environment variables and `.env` files with sensible defaults. No hardcoded absolute paths inside tool code.
5. The structure is welcoming for the hackathon: every tool has a stub `README.md`, the monorepo has a clear `CLAUDE.md` for coding agents, and there's a dedicated hackathon directory for participants to land in.

Initial scope: the existing bot + four stub directories + a building blocks library.

- `slack_paperbot_ridder_lab/` — the existing live bot, moved in as-is. Submodule. Keeps its current name to match its GitHub remote.
- `deep_research_bot/` — stub. On-demand deep-research reports triggered by Slack slash commands. (Design notes in `<design_briefs>`.)
- `summarize_bot/` — stub. On-demand paper summarization + adversarial review.
- `coding_agent_installer/` — stub. Tooling to provision Claude Code (and similar) with lab defaults.
- `hackathon/` — playground for the upcoming hackathon. Multiple contributors will push here.
- `automation_building_blocks/` — reusable patterns and deployment recipes. Read-and-copy, not import.

You may propose better names; flag them at Stop Point 1.

## Portability conventions (these are the new house rules)

1. **No hardcoded absolute paths inside tool source code.** Use env vars with defaults. The convention:
   - `LAB_AI_DATA_DIR` — base directory for persistent state. Tools read/write under `$LAB_AI_DATA_DIR/<tool_name>/`. Default to `./data` if unset.
   - `LAB_AI_LOG_DIR` — base directory for logs. Default `./logs`.
   - `OLLAMA_BASE_URL` — Ollama endpoint. Default `http://localhost:11434`. (This is the standard env var name Ollama and most clients already respect.)
   - Tool-specific keys (Slack tokens, API keys) live in each tool's `.env` and are documented in its `.env.example`.
2. **Per-tool, self-contained `uv` venv.** Each tool has its own `pyproject.toml` and `uv.lock`; `uv sync` creates `.venv/` inside the tool directory. No shared/global venv.
3. **Secrets out of git.** Each tool ships a `.env.example` listing the keys it needs (no values). Real `.env` files are gitignored.
4. **Deployment is decoupled from the tool.** Tools expose a small set of entry points (e.g., `python -m mybot.run_once`, `python -m mybot.serve`). Whether those are invoked by HPC cron+sbatch, a systemd timer on a VM, a Docker container, or `python` directly on a laptop is a deployment-time concern handled by recipes, not by the tool itself.
5. **Logs to stdout/stderr by default; file logging optional via `LAB_AI_LOG_DIR`.** Lets the same code work under cron (where stdout goes to a log file), under systemd (where the journal captures stdout), and under Docker (same).
6. **Where data goes is the deployer's choice.** A tool that needs to remember things asks for `LAB_AI_DATA_DIR` and writes under `$LAB_AI_DATA_DIR/<tool_name>/`. On HPC the deployer might point this at `/hpc/compgen/projects/lab_ai_automation/<tool>/data/`; on a laptop, `~/.local/share/lab_ai/`; in Docker, a mounted volume. The tool doesn't care.

## What I've researched (for your awareness)

I did a deep-research pass on deep-research agents, persistent knowledge bases, and prompt engineering. The relevant excerpts are in `<design_briefs>` at the end. Do NOT implement any of those systems now — the task is *scaffolding* only. The excerpts exist so the stub READMEs (especially `deep_research_bot`) reference real, current tooling rather than guessed-at names.

</context>

---

<execution_phases>

Work in these phases, in order. Two explicit STOP points.

### Phase 0 — Orient

1. Confirm `whoami` returns `dstoker` and you are on a submit node (`hpcs05` or `hpcs06`). If not, stop and tell me.
2. List `/hpc/compgen/projects/` and confirm the existing slackbot path.
3. Inspect `/hpc/compgen/projects/slackbot_fyi_papers_info/` thoroughly. Produce a structured report covering:
   - Directory tree (2–3 levels deep).
   - `git remote -v`, `git status`, `git log --oneline -n 5`. Is the working tree clean? Any uncommitted/unpushed work?
   - Cron setup (`crontab -l`): what script is called, how often, what does it do at a high level?
   - Whether the tool already uses `sbatch` or runs everything on the submit node.
   - Where the Python environment lives, what manages it (uv / pip / poetry / conda).
   - Where Ollama is configured (grep for `OLLAMA`, `11434`, `ollama`).
   - Where secrets live (grep for `.env`, `API_KEY`, `SLACK_TOKEN`, `XOXB`, `XAPP`).
   - Where logs and persistent state live.
   - **A complete list of every absolute path** (anything starting with `/hpc/`, `/home/`, etc.) that appears in scripts, configs, the cron line, or any service files. These are what will need updating after the move.

After Phase 0, surface findings as a structured report in your reply. Do not proceed until you have written this down.

### Phase 1 — Propose structure and git strategy  ⬛ **STOP POINT 1** ⬛

Propose:

1. **Concrete monorepo directory layout.** Working assumption (challenge if you see issues):

   ```
   /hpc/compgen/projects/lab_ai_automation/             # monorepo root; one git repo
   ├── README.md
   ├── CLAUDE.md
   ├── .gitignore
   ├── .gitmodules
   │
   ├── slack_paperbot_ridder_lab/                       # existing bot, moved-in submodule
   │
   ├── deep_research_bot/                               # stub submodule
   │   ├── README.md
   │   └── .env.example
   │
   ├── summarize_bot/                                   # stub submodule
   ├── coding_agent_installer/                          # stub submodule
   ├── hackathon/                                       # stub submodule, hackathon playground
   │   ├── README.md
   │   └── teams/
   │       └── .gitkeep
   │
   └── automation_building_blocks/                      # stub submodule
       ├── README.md
       ├── INDEX.md
       └── deployment_recipes/
           └── README.md
   ```

   Note: NO `analysis/<username>/` nesting inside tools. Tools are flat and portable. The HPC convention applies only to the monorepo root's location, not to its internal layout.

2. **Git strategy.** Evaluate at least:
   - **A. Submodules.** Each tool is its own GitHub repo, referenced as a submodule. Preserves the existing bot's `https://github.com/DieStok/slack_paperbot_ridder_lab.git` remote and its history. Familiar warts: submodule UX, detached HEADs, `--recurse-submodules`.
   - **B. Subtree.** Each tool merged into the monorepo with subtree push/pull to external remotes. Cleaner clone UX, more complex maintenance.
   - **C. Plain monorepo, single history.** Loses the existing bot's independent remote unless kept in parallel. Simpler but worse for multi-contributor independent work.

   Recommend one. For the existing bot's remote, **the plan must preserve `https://github.com/DieStok/slack_paperbot_ridder_lab.git` with no force-pushes and no history rewrites**. For the new stubs, the plan should describe how each will be initialized (local repo first, GitHub remote added later by me — or, if you have a strong reason, propose otherwise).

3. **Naming.** Confirm the monorepo name `lab_ai_automation` (or propose better). Confirm directory names for each stub. Confirm the existing bot keeps the directory name `slack_paperbot_ridder_lab` to match its GitHub remote (or propose a cleaner alias and how to handle the mismatch).

4. **Per-tool stub layout.** Each new stub will eventually contain a `pyproject.toml`, `src/`, `tests/`, `.env.example`, `README.md`, and possibly its own `CLAUDE.md`. For Phase 2, you only need to create the README and `.env.example` and a `.gitkeep`-bearing skeleton that signals intent. Confirm this minimal scope.

**Stop here and wait for my explicit approval** before doing any file-system writes or git operations. Present the proposal compactly.

### Phase 2 — Scaffold the empty structure (reversible)

Once I approve Phase 1, do all reversible scaffolding before touching the live bot.

1. Create the monorepo directory and `git init` it.
2. Write `.gitignore` covering: `.venv/`, `.env`, `__pycache__/`, `*.pyc`, `logs/`, `*.log`, `data/`, `.DS_Store`, `node_modules/`, Python build artefacts, and common AI/ML detritus: `*.sqlite`, `*.duckdb`, `.chroma/`, `.langgraph_checkpoints/`, `models/`, `*.gguf`.
3. For each stub directory (`deep_research_bot`, `summarize_bot`, `coding_agent_installer`, `hackathon`, `automation_building_blocks`):
   - Create the directory.
   - Per the approved git strategy, either `git init` it as its own repo or just create the dir (decision from Stop Point 1).
   - Write its `README.md` per the templates in `<stub_specs>`.
   - Write a `.env.example` if applicable (`deep_research_bot`, `summarize_bot`, `hackathon`).
   - Write a stub `.gitignore` mirroring the root's.
   - For `automation_building_blocks/`, also create `INDEX.md` and `deployment_recipes/README.md` per the spec.
   - For `hackathon/`, also create `teams/.gitkeep`.
4. Write the top-level `README.md` and `CLAUDE.md` per `<top_level_files>`.
5. Commit at the monorepo root: `git add -A && git commit -m "Initial scaffolding of lab_ai_automation monorepo"`. Commit each sub-repo similarly if applicable.

Do NOT push to any remote yet. Do NOT touch the existing slackbot yet.

End Phase 2 by showing me `tree -L 3` and `git log --oneline` and waiting for confirmation.

### Phase 3 — Move the existing slackbot  ⬛ **STOP POINT 2** ⬛

This is the irreversible part. **The existing bot's source code will not be modified.** Only its location and the absolute path references that point to its old location.

Before doing anything, present a step-by-step migration plan with explicit commands, including:


1. How you will move the bot's working tree into `lab_ai_automation/slack_paperbot_ridder_lab/` while preserving its `.git` directory and its remote.
2. Exactly which absolute paths (from the Phase 0 inventory) need updating, and where. These are limited to: cron line, any startup/wrapper scripts that reference the old path, any config files that hardcode the old path. **Not Python source.** If you find Python source that hardcodes paths, list it as an open question for me to fix later — do not touch it.
3 How you will re-enable cron only after a manual smoke test.
6. Rollback plan if anything fails (where the old crontab is saved, how to move the directory back).

**Stop and wait for my explicit approval of this plan before executing it.**

After approval and execution:
- Smoke-test from the new location (dry run if the bot supports one; otherwise run its main entrypoint with output going to a temp file rather than Slack, if possible). If no safe dry-run exists, ask me before running.
- Re-enable cron pointing at the new location.
- From the monorepo root, add the bot as a submodule per the approved git strategy (e.g., `git submodule add <remote> slack_paperbot_ridder_lab`).
- Commit at the monorepo root: `Add slack_paperbot_ridder_lab as submodule (moved from /hpc/compgen/projects/slackbot_fyi_papers_info)`.

### Phase 4 — Seed `automation_building_blocks/`

Don't extract anything from the live bot — it's HPC-coupled and I will refactor it later. Instead, create ONE small, fresh, portable example so the building-blocks pattern is visible and the hackathon participants have something to imitate.

1. Create `automation_building_blocks/slack_app_skeleton/` with:
   - `README.md` explaining what this is, when to copy it, what to change.
   - A minimal `pyproject.toml` declaring `slack_bolt` and `python-dotenv` as deps.
   - `.env.example` listing `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`.
   - `app.py` — a 30–60 line Slack Bolt skeleton with one `/hello` slash command handler that responds with "Hello from the lab AI skeleton". Uses Socket Mode so it works without a public URL (important for laptops behind NAT and for HPC where ingress is blocked). Reads tokens from env via `python-dotenv`.
   - A short note explicitly stating that `LAB_AI_DATA_DIR` and `LAB_AI_LOG_DIR` are honored if set, with defaults to `./data` and `./logs`.
2. Create `automation_building_blocks/deployment_recipes/` and populate with placeholder files (each ~20–40 lines, more pointer than implementation):
   - `README.md` — overview of the recipe categories below and the philosophy ("a tool's source code never knows how it's deployed").
   - `hpc_cron_sbatch.md` — explains the pattern: lightweight cron entry on the submit node calls a dispatcher; dispatcher submits an `sbatch` job for the actual work. Includes a minimal sbatch template referencing the lab's standard `--account=compgen` and `%x_%j` log naming. **Marked as one option, not the default.**
   - `local_dev.md` — just `python -m <tool>.run_once` or `uv run <tool>` from the tool directory. The simplest path; what hackathon participants will use.
   - `systemd_timer.md` — placeholder, ~10 lines explaining the pattern and saying "to be filled in when we deploy on a VM."
   - `cloud_vm.md` — placeholder noting Google Cloud and Sprite as candidate hosts; "to be filled in when we have a target."
3. Update `automation_building_blocks/INDEX.md` with a one-line entry per block:
   - `slack_app_skeleton/` — minimal Slack Bolt app with Socket Mode; copy as the starting point for any new bot.
   - `deployment_recipes/` — patterns for running a tool under HPC cron+sbatch, locally, under systemd, or on a cloud VM.

Commit: `Add slack_app_skeleton building block and deployment recipe stubs`.

### Phase 5 — Summary

Final report containing:

- `tree -L 4` of the monorepo.
- `git status` and recent log across the monorepo and each submodule.
- Cron's current state and confirmation that the moved bot has executed at least once successfully in its new location (or, if that hasn't happened yet because of cron timing, a note about when the next run is scheduled).
- A list of "open questions / decisions deferred" — anything you flagged during execution.
- A list of **path updates I (the user) still need to make in the existing bot's Python source** to make it portable. You will NOT make these edits, but you should enumerate them precisely from your Phase 0 inventory: file path, line number, current value, what kind of value the portable version should accept (e.g., env var, config key).
- A suggested ordered next-step list for me before the hackathon, e.g.:
  1. Refactor the live bot to read `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL` from env.
  2. Create GitHub remotes for the new stubs and `git push -u`.
  3. Populate hackathon README with date, goals, ground rules.
  4. Decide cloud VM target.

</execution_phases>

---

<top_level_files>

## Template: `lab_ai_automation/README.md`

Tone: concise, welcoming, opinionated about portability, agnostic about specific tools. Target ~150–250 lines. Cover at minimum:

- **What this is.** One paragraph: a monorepo of AI-based automation and research tooling for the De Ridder Lab. Each tool is independently versioned (own git repo) but lives here so contributors can see everything at once and share patterns. Designed to run on HPC, on a laptop, or on a cloud VM with documented deployment recipes.
- **Why portable.** Two-paragraph rationale: HPC is great for batch compute but not ideal for long-running internet-facing bots; laptops are great for hacking but not always available; cloud VMs may be the future. Tool code stays platform-agnostic; deployment is a separate concern.
- **Directory layout.** Code block with one-line explanations.
- **Conventions all tools follow.** Bullet list:
  - No hardcoded absolute paths in source code; use `LAB_AI_DATA_DIR`, `LAB_AI_LOG_DIR`, `OLLAMA_BASE_URL` env vars.
  - Per-tool `uv` venv (`uv sync` inside the tool directory creates `.venv/`).
  - Secrets in `.env` (gitignored); `.env.example` in git lists required keys.
  - Logs default to stdout/stderr; optional file logging via `LAB_AI_LOG_DIR`.
  - Lowercase, underscore-only directory names.
- **Running a tool.** Three short subsections:
  - **Locally**: clone, `cd <tool>`, copy `.env.example` to `.env`, fill in keys, `uv sync`, `uv run python -m <tool>.run_once` (or whatever the tool's entry point is).
  - **On HPC**: same as local, plus see `automation_building_blocks/deployment_recipes/hpc_cron_sbatch.md` for cron+sbatch pattern. Note the policy: long-running internet-facing processes are not ideal on HPC; prefer batch invocations.
  - **On a cloud VM**: planned, see `automation_building_blocks/deployment_recipes/cloud_vm.md` (currently a placeholder).
- **How to clone.** Both commands: `git clone --recurse-submodules <monorepo>` for the whole thing, and `git clone <tool_remote>` if you only want one tool.
- **How to add a new tool.** Short recipe: create GitHub repo, copy a stub README, copy the slack_app_skeleton if it's a Slack bot, add as a submodule under `lab_ai_automation/`.
- **Building blocks.** One paragraph pointing at `automation_building_blocks/` and explaining: read-and-copy library of patterns and deployment recipes. Not a package, not imported.
- **Current tools.** Table or list: name → one-line description → status (live, stub, hackathon).
- **Hackathon.** Brief pointer to `hackathon/README.md`.
- **Maintainer.** `dstoker` for now; lab Slack as primary channel.

Do NOT include implementation details for any specific tool. Those live in each tool's README.

## Template: `lab_ai_automation/CLAUDE.md`

Audience: Claude Code and similar coding agents. Length 200–400 lines. Written in second person. Concrete, lab-specific, opinionated — not generic agent boilerplate. (Recent research from ETH Zurich found that naïve LLM-generated `CLAUDE.md` files hurt performance in 5 of 8 settings; this file must be curated content, not filler.)

Cover at minimum:

- **Repo purpose.** Two sentences.
- **Absolute rules** (numbered list):
  1. Never commit `.env`, model weights, or bulk data.
  2. Never force-push or rewrite history on any submodule. The `slack_paperbot_ridder_lab` submodule is live in production; its remote is shared.
  3. No hardcoded absolute paths in tool source code. Use the documented env vars.
  4. Directory names lowercase, underscores only.
  5. When changing a tool's behavior, update its README and `.env.example` in the same commit.
- **Platform awareness.** A short section: this repo runs on HPC, laptop, and (eventually) cloud VM. Don't assume any specific platform inside tool code. If you need to do something platform-specific (e.g., submit an sbatch job), write a deployment recipe under `automation_building_blocks/deployment_recipes/` rather than baking it into a tool.
- **Environment expectations.** `uv` for package management; Python 3.11+; Ollama may run on localhost (laptop), on a GPU compute node (HPC, started via the lab's sbatch template), or on a remote endpoint (cloud). Always read `OLLAMA_BASE_URL` from env; never hardcode `http://localhost:11434`.
- **Per-tool overrides.** If a subdirectory has its own `CLAUDE.md` or `AGENTS.md`, that file's instructions take precedence within that subtree.
- **Where reusable patterns live.** Point at `automation_building_blocks/INDEX.md`. Before writing a new Slack handler or deployment script, check the index and prefer copying.
- **Secrets.** `.env` / `.env.example` pattern. Each tool maintains its own `.env.example`.
- **Commit conventions.** Short imperative messages. When working inside a submodule, the submodule's commits are scoped to that tool. The monorepo's commits typically just bump submodules: `Bump deep_research_bot to <short_sha>`.
- **The existing bot.** Note that `slack_paperbot_ridder_lab/` is currently HPC-coupled and being kept that way deliberately. The user (`dstoker`) will refactor it for portability separately. Do NOT modify its source unless explicitly asked.
- **What to do when uncertain.** Read before writing. Never delete tests to make CI pass. Never comment out failing logic without surfacing it. If a lab convention isn't documented here, ask the user.

</top_level_files>

---

<stub_specs>

Each stub `README.md` is 40–120 lines. Goal: a hackathon participant or future coding agent can read it and understand (1) what the tool will become, (2) what conventions it follows, (3) what concrete next steps would be. Do NOT write implementation code — write a scoped design brief.

Standard sections for every stub:

- **Status:** stub / in development / live.
- **Purpose:** 2–4 sentences.
- **Interaction model:** how is it triggered? slash command, cron, CLI, etc.
- **Planned stack:** current best guess, marked "subject to revision".
- **Deployment options:** which recipe(s) under `automation_building_blocks/deployment_recipes/` apply.
- **Configuration:** which env vars and `.env` keys it expects (link to `.env.example`).
- **Open questions:** 3–6 bullets.
- **Next steps:** short ordered list.

Tool-specific content:

### `slack_paperbot_ridder_lab/README.md`

This is the moved-in existing bot. Its README likely already has real content. **Do not overwrite it.** At most, prepend a short "Location" section noting that this tool is now a submodule of `lab_ai_automation` and that the new monorepo path is `/hpc/compgen/projects/lab_ai_automation/slack_paperbot_ridder_lab/`. Mention that the bot is currently HPC-coupled and will be refactored for portability separately. Everything else stays.

### `deep_research_bot/README.md`

Purpose: triggered by a Slack slash command (`/deep_research [workflow_name] [prompt] [starting_contexts]`), produce a comprehensive research report combining web search, literature search (PubMed, OpenAlex, arXiv, bioRxiv), and local LLM reasoning. Output: versioned `.md` + `.pdf` report, structured metadata, written to a lab-central knowledge store searchable via `/search`. Eventually cross-links related prior reports.

Planned stack (per `<design_briefs>`): LangGraph for orchestration, LiteLLM for multi-provider LLM access, Ollama for local models by default with optional remote-API fallback, Tavily for web search, academic-source APIs, Crawl4AI for page reading, Obsidian vault + BM25 + embeddings for the knowledge store, MCP to expose that store to coding agents. Subject to revision.

Interaction: Slack listener (Socket Mode, runs anywhere) accepts the command, validates, writes a job spec, dispatches the heavy work to whichever execution backend is configured (local subprocess, HPC sbatch, cloud worker), responds to Slack with "started, will post when done." The dispatch boundary is the platform abstraction.

Deployment options: any of the recipes; HPC cron+sbatch is suitable for the heavy compute step but not for the Slack listener (HPC is not a good home for a long-lived internet-facing process). Most likely deployment in practice: Slack listener on a small VM or laptop, heavy compute on HPC via sbatch.

Open questions: report visibility (lab-internal vs. public); metadata schema (initial prompt, expanded prompt, tool-call trace, final report — versioned); freshness/staleness policy; rate limiting; cost tracking when remote APIs are in the mix.

### `summarize_bot/README.md`

Purpose: triggered by `/summarize [model_name_optional] [instructions_optional]` (with a paper URL or attachment), produce a full-text summary plus adversarial review. Optionally pull the associated codebase if a repo is linked. Outputs go to the same knowledge base as `deep_research_bot`.

Planned stack: shares Slack/LLM/dispatch infrastructure with `deep_research_bot`. Adds a PDF acquisition pipeline (arXiv, bioRxiv, publisher sniffing) and a codebase fetcher (`git clone` for GitHub links). "Adversarial review" implemented as a second LLM pass with a skeptic system prompt, not a separate agent for now.

Open questions: paywalled PDF handling; representation of "review" vs. "summary" in stored artefacts; whether to support follow-up Q&A in-thread.

### `coding_agent_installer/README.md`

Purpose: provision coding-agent environments (Claude Code primarily; others as added) on whatever platform the user is on, with lab defaults pre-configured — MCP servers connected to the knowledge base, `CLAUDE.md` / `AGENTS.md` pointing at shared conventions, `uv` installed, `OLLAMA_BASE_URL` auto-detected when possible, project skills in the right place. Likely a small CLI (`lab-agent-install <flavor>`) plus templated config files.

Open questions: per-user vs. shared install model; container packaging; how to keep templates in sync with lab conventions over time; license boundaries for proprietary agent products.

### `hackathon/README.md`

Purpose: the playground for the upcoming lab hackathon. Multiple contributors push here. Contains example building blocks, seed questions, conventions, and guardrails. Forked or branched per team.

Sections:

- **Goals.** Placeholder; I will fill in.
- **Date and logistics.** Placeholder.
- **Ground rules.** No secrets in git. No force-pushes to `main`. Branch-per-team (`team/<n>`). PRs reviewed by at least one other participant.
- **Starter resources.** Pointer to `automation_building_blocks/slack_app_skeleton/` as the easiest starting point. Pointer to the moved-in `slack_paperbot_ridder_lab/` as a real-world example (with the caveat that it is currently HPC-coupled). Pointer to `CLAUDE.md`.
- **Ideas worth trying.** A few stub bullets (I will populate).
- **Setup instructions.** A short "from zero to running a hello-world Slack bot in 10 minutes" recipe using `slack_app_skeleton`. Concrete commands.

Create `teams/.gitkeep` so each team has an obvious place to land their work.

### `automation_building_blocks/README.md`

Purpose: a library of reference patterns and deployment recipes. **Read-and-copy, not import.** Each block is a small, self-contained example with its own mini-README explaining what it does, what to change when copying, and which tool (if any) it was extracted from.

Structure:

```
automation_building_blocks/
├── README.md
├── INDEX.md
├── slack_app_skeleton/
│   ├── README.md
│   ├── app.py
│   ├── pyproject.toml
│   └── .env.example
└── deployment_recipes/
    ├── README.md
    ├── hpc_cron_sbatch.md
    ├── local_dev.md
    ├── systemd_timer.md
    └── cloud_vm.md
```

"When to extract a block": (a) two or more tools need the same pattern, (b) the pattern is non-trivial, (c) it encodes a lab convention worth stabilizing.

Non-goals: not a package, no `pyproject.toml` at the root, not installed, not imported. Each block stands alone.

</stub_specs>

---

<design_briefs>

The following excerpts are the relevant portions of a deep-research synthesis I did on deep-research agents, persistent knowledge bases, and prompt engineering in 2026. Use them when writing the "planned stack" section of the `deep_research_bot` stub README. Do NOT implement any of this now.

## Deep research agents: stack recommendations (2026)

**Recommended stack for a customizable framework (2–4 weeks dev time):**
Agent framework: **LangGraph** (dominant orchestration substrate, ~24,600 stars, MIT). Search infrastructure: **Tavily** ($8/1K queries, built for AI agents) + **Semantic Scholar API** + **arXiv API** + **PubMed E-utilities**. Web scraping: **Crawl4AI** (self-hosted, 6× faster) or **Firecrawl** (managed). Memory system: **Mem0** (easiest integration, ~48K stars) or **Zep/Graphiti** (better temporal reasoning, 63.8% vs 49.0% on LongMemEval). Knowledge base: **Obsidian vault via MCP** or **Neo4j via MCP**. LLM access: **LiteLLM** for unified API across OpenAI, Anthropic, Google, and local providers.

**Architecture flow:** User Query → Prompt Expander (seed-expand-refine) → Orchestrator (LangGraph state machine) → parallel dispatch to Planner, Memory Manager, Supervisor, and Tools (Web Search, Academic Search, ClinicalTrials, bioRxiv/medRxiv, Page Reader, Code Executor) → Synthesizer → Knowledge Base Writer → Structured Research Report.

**Key empirical findings:** AgentFold (October 2025) achieved 92% context reduction via two-scale folding. Yunque DeepResearch (January 2026) established simultaneous SOTA on BrowseComp (62.5), BrowseComp-ZH (75.9), and HLE (51.7). MiroFlow (February 2026) reached 82.4% on GAIA at $0.07/query vs OpenAI Deep Research at $2.00/query. Open-source has reached parity with commercial. The single strongest predictor of task failure is **forgetting** (coefficient −0.843, p = 0.014), not reasoning ability. Structured memory management matters more than larger models. ChatGPT-o3 with basic web search beat OpenAI's own Deep Research on FutureSearch DRB (0.60 vs 0.44) — base model + simple search can outperform elaborate scaffolding.

## Persistent knowledge bases

**Wiki-type — Obsidian dominates.** Plain Markdown + YAML frontmatter + wikilinks. LLM-native format. Ecosystem: Obsidian Skills (~19,500 stars, agent skill specs for Claude Code), Obsidian Copilot (Agent Mode), Smart Connections (local-embedding semantic maps), Obsilo (autonomous agent layer with 55+ tools). Strengths: local-first, Git-compatible, LLM-native. Limitation: plugin fragmentation; performance degrades past ~10,000 notes.

**Graph databases — Neo4j enables cross-document reasoning.** Schema for research: Papers, Authors, Methods, Datasets, Claims, Evidence with AUTHORED, CITES, USES_METHOD, SUPPORTS_CLAIM relationships. Native vector indexes since v5.11+. Hybrid retrieval: Microsoft GraphRAG (corpus-wide thematic, expensive); LightRAG (EMNLP 2025, 10–100× cheaper, incremental); Graphiti/Zep (~24,400 stars, bi-temporal modeling, 94.8% on DMR, P95 <300ms, no LLM calls at query time).

**Agent memory frameworks.** Mem0 (~48K stars, AWS Agent SDK exclusive) uses an Add/Update/Delete/Nothing cycle; 49.0% on LongMemEval. Letta (~13K stars, formerly MemGPT) implements "LLM as OS" with Core/Archival/Recall memory; agents are active participants in memory management. Letta Code (March 2026) is #1 model-agnostic open-source agent on Terminal-Bench. Zep (built on Graphiti) hits 63.8% on LongMemEval — 15-point gap over Mem0.

## MCP — the connective tissue

**Model Context Protocol**, announced by Anthropic November 2024, now stewarded by Linux Foundation, adopted by Anthropic, OpenAI, Google DeepMind, Microsoft. JSON-RPC 2.0 over stdio (local) or Streamable HTTP (remote). Servers expose Tools (model-controlled functions), Resources (application-controlled data with URIs), Prompts (user-controlled instruction templates). Obsidian and Neo4j can both be exposed via MCP; any MCP-compatible coding agent can then read/write them.

## Context engineering and project knowledge files

Karpathy's framing: LLM is CPU, context window is RAM, context engineering is the OS. Anthropic research: token usage explains 80% of performance variance on BrowseComp. A focused 300-token context often outperforms an unfocused 113,000-token context.

`CLAUDE.md` is Claude Code's native context mechanism (hierarchical: user-level → subdir-specific). `AGENTS.md` is the cross-tool standard donated by OpenAI to the Linux Foundation, adopted by 20,000+ GitHub repos, compatible with Codex, Jules, Cursor, Aider, Zed.

**Important caveat (ETH Zurich):** LLM-generated context files hurt performance in 5 of 8 settings, adding 2.45–3.92 extra steps per task. Curated by humans or high-quality research agents — never naïvely auto-generated. This is why the `CLAUDE.md` written for this monorepo must be concrete and lab-specific.

## Automatic prompt expansion: seed-expand-refine

**Seed** (raw query) → **Expand** (web/academic/GitHub/KB search; sub-question decomposition; entity extraction) → **Refine** (structured research prompt with objectives, scope, output format, source criteria, search directives) → **Validate** (intent preservation, scope creep, hallucinated requirements, actionability).

Five decomposition methods: Least-to-Most (99% on SCAN vs 16% CoT); Self-Ask (routes follow-ups to search APIs); Plan-and-Solve (LangChain default); Tree of Thoughts (74% vs 4% CoT on Game of 24); Decomposed Prompting (modular handlers). Validation principles (Agent-Oriented Planning, ICLR 2025): solvability, completeness, non-redundancy.

**The 23–47 point lever:** the DETAIL framework showed detailed prompts improve accuracy by 23–47 percentage points over vague prompts. OpenAI's Deep Research API uses gpt-4.1 for prompt rewriting before handoff to the research model.

## Anchor citations for the deep_research_bot README

- Yao et al. (2023). ReAct. arXiv:2210.03629.
- Mialon et al. (2023). GAIA. arXiv:2311.12983.
- Shao et al. (2024). STORM. arXiv:2402.14207.
- Zheng et al. (2025). DeepResearcher. arXiv:2504.03160.
- Tongyi DeepResearch Team (2025). arXiv:2510.24701.
- Cai et al. (2026). Yunque DeepResearch. arXiv:2601.19578.
- Khattab et al. (2024). DSPy. arXiv:2310.03714.
- Anthropic (2024). Contextual Retrieval. anthropic.com/news/contextual-retrieval.
- Edge et al. (2024). Microsoft GraphRAG. arXiv:2404.16130.

Use these sparingly so the design notes are grounded in real prior art without becoming a literature review.

</design_briefs>

---

<guardrails>

## Action bias and safety

- Take local, reversible actions freely: creating directories under the new monorepo, writing new files inside it, running read-only inspection commands.
- For destructive or shared-system actions, **stop and confirm**:
  - Moving or modifying anything inside `/hpc/compgen/projects/slackbot_fyi_papers_info/`.
  - Editing or disabling the cron job.
  - Force-pushing or rewriting any git history. **Do not do this under any circumstances.**
  - Pushing to any GitHub remote.
  - Deleting any file.

## Hard constraints on the existing bot

- Do not modify a single line of the existing bot's Python source. The user will refactor it for portability separately.
- The only acceptable changes during the move are: (a) the cron line pointing at the new path, (b) wrapper or launcher scripts that contain absolute paths to the old location. Anything else is an "open question" for the user.
- If the bot's source contains hardcoded paths, list them precisely in the Phase 5 report so the user can fix them deliberately.

## Hallucination minimization

- Never speculate about what the existing bot does. Read its code before describing it.
- Never invent file paths, script names, or configurations in the report. Only mention things you have actually seen.
- If you need information that requires web search (e.g., a specific `slack_bolt` idiom for Socket Mode, or a `uv` invocation pattern), flag it and ask rather than guess.

## Staying in scope

- Scaffolding only. Don't implement LangGraph code, don't set up Ollama, don't build the knowledge base.
- Don't refactor the existing bot.
- Don't extract building blocks from the live bot — it's HPC-coupled. The Phase 4 building block (`slack_app_skeleton`) is fresh and portable.
- If you see other things that "should" be cleaned up, put them in the Phase 5 open-questions list.

## Tool-use etiquette

- For independent read-only commands, run them in parallel.
- When modifying files, prefer minimal diffs; don't reformat files you're not actively changing.

## When you're done

Hand back control with the Phase 5 report. Do not push past it. A clean stopping point with good documentation is more valuable than more half-finished work.

</guardrails>

---

Begin with Phase 0.
