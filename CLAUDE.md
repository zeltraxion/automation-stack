---
generated_by: claude
human_requestor: swapnil.bhattacharya@clearroute.io
date: 2026-09-14
review_status: unreviewed
reviewed_by:
---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A locally self-hosted n8n instance defined with Docker Compose, built as a learning project and intended to migrate
later to Oracle Cloud Free Tier managed by Coolify. There is no application source code here — the repository is
infrastructure configuration plus space for exported workflows and future custom nodes. ⟦REVIEW src: docker-compose.yml
and directory listing, read this session⟧

⟦REVIEW⟧

## Commands

Run everything from the repository root. Docker Compose reads `.env` from the working directory, so commands executed
elsewhere will fail to substitute variables. ⟦REVIEW⟧

```bash
docker compose up -d                # start; ALSO applies config changes by recreating the container
docker compose down                 # stop and remove the container (data is safe)
docker compose ps                   # status; the ports column shows the bind address
docker compose logs -f n8n          # follow logs; Ctrl+C stops watching, not n8n
docker compose logs --tail=50 n8n   # last 50 lines — better when hunting an error
docker compose config               # render the file with .env substituted
docker compose exec n8n sh          # shell inside the container
```

`docker compose config` prints the resolved encryption key in plain text — do not paste its output anywhere. ⟦REVIEW⟧

Editor UI: `http://localhost:5678` ⟦REVIEW src: docker-compose.yml, ports mapping⟧

Upgrading n8n: change `N8N_VERSION` in `.env`, then `docker compose pull && docker compose up -d`. ⟦REVIEW⟧

### Verification commands

These were used to validate the setup and are the fastest way to confirm it is still healthy: ⟦REVIEW⟧

```bash
# Encryption key in the volume matches .env — expect "1"
docker compose exec -T n8n sh -c 'grep -c "$N8N_ENCRYPTION_KEY" /home/node/.n8n/config'

# Container clock honours the timezone — expect IST, not UTC
docker compose exec n8n date

# Settings file permissions — expect -rw-------
docker compose exec n8n ls -la /home/node/.n8n/config

# n8n can reach Ollama on the host — expect the model list as JSON
docker compose exec n8n wget -qO- http://host.docker.internal:11434/api/tags
```

⟦REVIEW⟧

## Architecture

### The config split is deliberate

`docker-compose.yml` describes the *shape* of the infrastructure; `.env` holds the values that differ per machine. Keep
it that way — Coolify deploys by reading the compose file from git and injecting environment variables from its own UI,
so hardcoding values into the compose file would break the migration path. ⟦REVIEW src: docker-compose.yml
uses ${N8N_VERSION}, ${GENERIC_TIMEZONE}, ${N8N_ENCRYPTION_KEY}⟧

⟦REVIEW⟧

### Storage: named volume for state, bind mounts for the vault

| Mount                                                     | Type                           | Holds                                          |
|-----------------------------------------------------------|--------------------------------|------------------------------------------------|
| `n8n_data:/home/node/.n8n`                                | named volume, `external: true` | SQLite database, settings file, encryption key |
| `${VAULT_PATH}/files:/vault/files`                        | bind mount                     | scratch files workflows read and write         |
| `${VAULT_PATH}/control_plane:/vault/control_plane:ro`     | bind mount, read-only          | switches you set by hand                       |
| `${VAULT_PATH}/dashboard:/vault/dashboard`                | bind mount                     | output n8n regenerates                         |
| `${VAULT_PATH}/memory/briefing:/vault/memory/briefing:ro` | bind mount, read-only          | facts you give the agent                       |
| `${VAULT_PATH}/memory/learned:/vault/memory/learned`      | bind mount                     | facts the agent saves                          |

⟦REVIEW src: docker-compose.yml, volumes block⟧

The database uses a **named volume rather than a bind mount** because SQLite over a macOS bind mount hits UID-ownership
and file-locking problems. Do not convert it to a bind mount. ⟦REVIEW⟧

`external: true` means Compose neither creates nor deletes this volume. Two consequences: the volume keeps the plain
name `n8n_data` instead of being project-prefixed, and **`docker compose down -v` cannot destroy it**. The volume must
be created by hand before first start: ⟦REVIEW src: docker-compose.yml, volumes: n8n_data: external: true⟧

```bash
docker volume create n8n_data
```

⟦REVIEW⟧

Inside a workflow, always use the **container** path (`/vault/files/report.csv`, `/vault/memory/learned/fact.md`), never
the macOS path. `N8N_RESTRICT_FILE_ACCESS_TO=/vault` permits anything under `/vault` and blocks everything else. ⟦REVIEW
src: docker-compose.yml, environment and volumes blocks⟧

### The encryption key is set once, permanently

`N8N_ENCRYPTION_KEY` encrypts stored credentials before they are written to SQLite. It must be present **before first
boot**. If n8n starts without one it generates its own and writes it into `/home/node/.n8n/config`; supplying a
different value afterwards fails the startup check. Changing it later requires n8n's documented key-rotation procedure,
not an edit to `.env`. ⟦REVIEW src: n8n docs, "Set a custom encryption key" — n8n generates a random key on first launch
and stores it in the .n8n folder⟧

⟦REVIEW⟧

### Ollama runs on the host, not in a container

Local models are served by Ollama installed natively on macOS, reached from n8n at `http://host.docker.internal:11434`.
⟦REVIEW src: verified this session via docker compose exec wget⟧

**Do not containerise Ollama on macOS.** Docker Desktop runs containers inside a Linux VM with no access to Apple's
Metal GPU, so a containerised Ollama would fall back to CPU inference and be dramatically slower. ⟦REVIEW⟧

Inside the container, `localhost` means the container itself — `host.docker.internal` is the only way to reach the Mac.
⟦REVIEW⟧

### Agent configuration findings

Measured in this repository against the same task and tools: ⟦REVIEW src: execution comparisons run this session in
workflow test_chat_LLM⟧

- **Tool parameters must be marked AI-fillable** (the sparkle icon in the node UI). A tool without them is a fixed
  button the agent cannot steer — it will fall back to guessing, confidently and wrongly.
- **A System Message instructing tool use is required.** Without an explicit rule such as "always use the Calculator for
  arithmetic, never calculate in your head", the agent does arithmetic itself and gets it wrong.
- **Disable "Enable Thinking" and set Sampling Temperature to `0.1`** on the Ollama Chat Model node. Thinking mode took
  4m39s on one query; disabled, with proper tools and a system prompt, the same query took 20s and was correct.
- **Tool names are read by the model when choosing.** Two tools both called "Date & Time" produce ambiguous selection;
  renaming to "Current Date" and "Between Dates" fixed it.

The general rule: when an agent misbehaves, suspect the tool definitions and system prompt before the model. ⟦REVIEW⟧

## Gotchas

- **`docker compose restart` does not apply config changes.** It reuses the existing container with its old settings.
  Use `up -d`, which detects the change and recreates. ⟦REVIEW⟧
- **`down` versus `down -v`.** One is routine, the other destroys volumes, and there is no confirmation prompt.
  `external: true` currently protects `n8n_data` from this. ⟦REVIEW⟧
- **Ollama's default context is 4096 tokens and it truncates silently** from the beginning of the prompt, with no error.
  For agent workloads this can cut the system prompt and tool definitions. Raise it via `OLLAMA_CONTEXT_LENGTH` or a
  derived model with `PARAMETER num_ctx`. **Currently set to 16384** in the Ollama app's settings, confirmed by
  the CONTEXT column of `ollama ps`. ⟦REVIEW src: research this session — Ollama silently truncates at num_ctx;
  agent workloads recommended at 64k+⟧
- **n8n 2.x disables `ExecuteCommand` and `LocalFileTrigger` by default**, and Code nodes can no longer read environment
  variables. Relevant if a workflow needs to shell out or watch a folder. ⟦REVIEW src: n8n 2.x release notes, breaking
  changes⟧
- **The port is bound to `127.0.0.1`.** Removing that prefix exposes n8n to every device on the local network; doing so
  also requires a TLS reverse proxy in front of n8n. `N8N_SECURE_COOKIE=false` makes login work over plain HTTP,
  but it strips the Secure attribute from the auth cookie, so the session token travels in clear text and any
  device on the network can read it. Use it only for throwaway local testing — never as the way to expose n8n. ⟦REVIEW src:
  docker-compose.yml, ports: "127.0.0.1:5678:5678"⟧

## Current state

- **Git repository**, currently on `feature/memory-write`; PR #2 merged. ⟦REVIEW src: `git branch --show-current` and
  `git log --oneline -1`, run 2026-09-20⟧
- `VAULT_PATH` is referenced on **five lines** of `docker-compose.yml`. The vault is mounted: `control_plane` and
  `memory/briefing` read-only, `dashboard`, `files` and `memory/learned` writable. `notes/` is deliberately not mounted.
  ⟦REVIEW src: docker-compose.yml, volumes block⟧
- `n8n_workflows/` holds four exports: `tars_agent.json`, `memory_write.json`, `test_chat_LLM.json`,
  `test_nodes_and_obsidian.json`.
  ⟦REVIEW src: directory listing, 2026-09-20⟧
- `custom-nodes/` and `.github/workflows/` exist but are empty. Git does not track empty directories, so they will not
  survive a clone without `.gitkeep` files. ⟦REVIEW src: directory listing, 2026-09-20⟧
- The environment template is `env.example`. `.gitignore` no longer carries a negation for `.env.example`, so the
  earlier naming mismatch is resolved. ⟦REVIEW src: `ls env.example` and `grep "env.example" .gitignore` returning no
  match, 2026-09-20⟧
- `README.md` is a full document — 201 lines. `docs/index.md` is populated and is the docs entry point. ⟦REVIEW src:
  `wc -lc README.md`, 2026-09-20⟧
- **The assistant is live:** `tars_agent`, published, with `save_memory` wired to `memory_write` as its only tool.
  Verified: it fills all four tool parameters correctly on an explicit request, and declines to call the tool
  otherwise. Documented in `docs/tars_agent.md`. ⟦REVIEW src: workflow executions run 2026-09-21⟧
- **First production sub-workflow is live:** `memory_write`, published, documented in `docs/memory_write.md`. ⟦REVIEW src:
  docs/memory_write.md; n8n_workflows/memory_write.json⟧
- **Model in use: `qwen3:8b`.** `qwen3:14b` was tested and rejected on measurement — 12 GB resident, swap at 94%,
  load average 17.4. 24 GB of RAM is the threshold that reopens the 22–24B tier. See `PLAN.md` §4. ⟦REVIEW src: `ollama ps` output and btop reading, 2026-09-20⟧
- Slack app **T.A.R.S** posts colour-coded block messages: saves and rejections to `#tars_logs`, write failures to
  `#tars_alerts`. n8n's Blocks field accepts an `{ "attachments": [...] }` wrapper despite documenting only
  `{ "blocks": [...] }` — that is the only way to get a colour strip. ⟦REVIEW src: workflow `memory_write`, three Slack nodes⟧
- Slack credential note: The credential is an access token stored in n8n, not in this
  repo. ⟦REVIEW src: docs/memory_write.md, Slack setup section⟧

⟦REVIEW⟧

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
