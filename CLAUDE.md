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
infrastructure configuration plus space for exported workflows and future custom nodes.

## Commands

Run everything from the repository root. Docker Compose reads `.env` from the working directory, so commands executed
elsewhere will fail to substitute variables.

```bash
docker compose up -d                # start; ALSO applies config changes by recreating the container
docker compose down                 # stop and remove the container (data is safe)
docker compose ps                   # status; the ports column shows the bind address
docker compose logs -f n8n          # follow logs; Ctrl+C stops watching, not n8n
docker compose logs --tail=50 n8n   # last 50 lines — better when hunting an error
docker compose config               # render the file with .env substituted
docker compose exec n8n sh          # shell inside the container
```

`docker compose config` prints the resolved encryption key in plain text — do not paste its output anywhere.

Editor UI: `http://localhost:5678`

Upgrading n8n: change `N8N_VERSION` in `.env`, then `docker compose pull && docker compose up -d`.

### Verification commands

These were used to validate the setup and are the fastest way to confirm it is still healthy:

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

## Architecture

### The config split is deliberate

`docker-compose.yml` describes the *shape* of the infrastructure; `.env` holds the values that differ per machine. Keep
it that way — Coolify deploys by reading the compose file from git and injecting environment variables from its own UI,
so hardcoding values into the compose file would break the migration path.

### Storage: named volume for state, bind mounts for the vault

| Mount                                                     | Type                           | Holds                                          |
|-----------------------------------------------------------|--------------------------------|------------------------------------------------|
| `n8n_data:/home/node/.n8n`                                | named volume, `external: true` | SQLite database, settings file, encryption key |
| `${VAULT_PATH}/files:/vault/files`                        | bind mount                     | scratch files workflows read and write         |
| `${VAULT_PATH}/control_plane:/vault/control_plane:ro`     | bind mount, read-only          | switches you set by hand                       |
| `${VAULT_PATH}/dashboard:/vault/dashboard`                | bind mount                     | output n8n regenerates                         |
| `${VAULT_PATH}/memory/briefing:/vault/memory/briefing:ro` | bind mount, read-only          | facts you give the agent                       |
| `${VAULT_PATH}/memory/learned:/vault/memory/learned`      | bind mount                     | facts the agent saves                          |

The database uses a **named volume rather than a bind mount** because SQLite over a macOS bind mount hits UID-ownership
and file-locking problems. Do not convert it to a bind mount.

`external: true` means Compose neither creates nor deletes this volume. Two consequences: the volume keeps the plain
name `n8n_data` instead of being project-prefixed, and **`docker compose down -v` cannot destroy it**. The volume must
be created by hand before first start:

```bash
docker volume create n8n_data
```

Inside a workflow, always use the **container** path (`/vault/files/report.csv`, `/vault/memory/learned/fact.md`), never
the macOS path. `N8N_RESTRICT_FILE_ACCESS_TO=/vault` permits anything under `/vault` and blocks everything else.

### The encryption key is set once, permanently

`N8N_ENCRYPTION_KEY` encrypts stored credentials before they are written to SQLite. It must be present **before first
boot**. If n8n starts without one it generates its own and writes it into `/home/node/.n8n/config`; supplying a
different value afterwards fails the startup check. Changing it later requires n8n's documented key-rotation procedure,
not an edit to `.env`.

### Ollama runs on the host, not in a container

Local models are served by Ollama installed natively on macOS, reached from n8n at `http://host.docker.internal:11434`.

**Do not containerise Ollama on macOS.** Docker Desktop runs containers inside a Linux VM with no access to Apple's
Metal GPU, so a containerised Ollama would fall back to CPU inference and be dramatically slower.

Inside the container, `localhost` means the container itself — `host.docker.internal` is the only way to reach the Mac.

### Agent configuration findings

Measured in this repository against the same task and tools:

- **Tool parameters must be marked AI-fillable** (the sparkle icon in the node UI). A tool without them is a fixed
  button the agent cannot steer — it will fall back to guessing, confidently and wrongly.
- **A System Message instructing tool use is required.** Without an explicit rule such as "always use the Calculator for
  arithmetic, never calculate in your head", the agent does arithmetic itself and gets it wrong.
- **Disable "Enable Thinking" and set Sampling Temperature to `0.1`** on the Ollama Chat Model node. Thinking mode took
  4m39s on one query; disabled, with proper tools and a system prompt, the same query took 20s and was correct.
- **Tool names are read by the model when choosing.** Two tools both called "Date & Time" produce ambiguous selection;
  renaming to "Current Date" and "Between Dates" fixed it.

The general rule: when an agent misbehaves, suspect the tool definitions and system prompt before the model.

## Gotchas

- **`docker compose restart` does not apply config changes.** It reuses the existing container with its old settings.
  Use `up -d`, which detects the change and recreates.
- **`down` versus `down -v`.** One is routine, the other destroys volumes, and there is no confirmation prompt.
  `external: true` currently protects `n8n_data` from this.
- **Ollama's default context is 4096 tokens and it truncates silently** from the beginning of the prompt, with no error.
  For agent workloads this can cut the system prompt and tool definitions. Raise it via `OLLAMA_CONTEXT_LENGTH` or a
  derived model with `PARAMETER num_ctx`. **Currently set to 16384** in the Ollama app's settings, confirmed by
  the CONTEXT column of `ollama ps`.
- **n8n 2.x disables `ExecuteCommand` and `LocalFileTrigger` by default**, and Code nodes can no longer read environment
  variables. Relevant if a workflow needs to shell out or watch a folder.
- **The port is bound to `127.0.0.1`.** Removing that prefix exposes n8n to every device on the local network; doing so
  also requires a TLS reverse proxy in front of n8n. `N8N_SECURE_COOKIE=false` makes login work over plain HTTP,
  but it strips the Secure attribute from the auth cookie, so the session token travels in clear text and any
  device on the network can read it. Use it only for throwaway local testing — never as the way to expose n8n.

## Current state

- **Git repository**, currently on `feature/mail_organizer`; PR #5 merged.
- `VAULT_PATH` is referenced on **five lines** of `docker-compose.yml`. The vault is mounted: `control_plane` and
  `memory/briefing` read-only, `dashboard`, `files` and `memory/learned` writable. `notes/` is deliberately not mounted.

- `n8n_workflows/` holds eight exports: `memory_agent.json`, `memory_write.json`, `memory_read.json`,
  `mail_label_draft.json`, `mail_categorizer.json`, `error_handler.json`, `test_chat_LLM.json`,
  `test_nodes_and_obsidian.json`.

- `custom-nodes/` and `.github/workflows/` exist but are empty. Git does not track empty directories, so they will not
  survive a clone without `.gitkeep` files.
- The environment template is `env.example`. `.gitignore` no longer carries a negation for `.env.example`, so the
  earlier naming mismatch is resolved.
- `README.md` is a full document — 201 lines. `docs/index.md` is populated and is the docs entry point.
- **The memory loop is closed:** `memory_agent`, published, with two tools — `save_memory` → `memory_write` and
  `recall_memory` → `memory_read`. Verified across a container restart: facts stated before the restart are recalled
  after it. An 8B model chains two `save_memory` calls in one turn, which only worked once a **worked example** was put
  in the system message — rules alone produced a dropped fact or two facts merged into one file. Documented in
  `docs/memory_agent.md`.
- **Updating a memory is ~80% reliable** when the user's wording differs from the stored summary; the rest create a
  near-duplicate and leave the old file stale. Mitigated by a date-aware conflict rule in the system message, resolved
  manually until Phase 1.5.
- **Both production sub-workflows are live:** `memory_write` and `memory_read`, published, documented in
  `docs/memory_write.md` and `docs/memory_read.md`.
- **The mail domain is live.** `mail_label_draft`, published, Schedule Trigger every 30 minutes, calls
  `mail_categorizer` as a sub-workflow. It labels every fetched email into one of nine categories, and writes a Gmail
  draft for the one or two a person is actually waiting on. **Nothing is archived, deleted or sent** — drafts only,
  deliberately, until accuracy is trusted. Documented in `docs/mail_label_draft.md` and `docs/mail_categorizer.md`.

- **`error_handler` is the shared failure alert.** Two nodes — Error Trigger into a red Slack card on `#tars_alerts`
  naming the workflow, the failing node and a link to the execution. Set as the Error Workflow on all five production
  workflows; deliberately not on itself, since an error handler that errors calls itself. Documented in
  `docs/error_handler.md`.
- **A node set to `Continue (using error output)` does not fail the execution, so `error_handler` never sees it.**
  That is the handover between the two logging styles: per-node error outputs report through the workflow's own Slack
  card, `error_handler` covers everything unhandled. Five nodes in `mail_label_draft` work that way.
- **The four Gmail nodes retry three times at two-second intervals.** A dropped connection to Google is exactly what
  retry exists for — an alert now means three consecutive failures, not one bad response.
- **Three gates stand between a fetched email and a model call in the draft branch:** a category gate, a sender-address
  check, then a Text Classifier. The first two are deterministic and cut 25 emails to 0–2, which is what makes the two
  model calls after them affordable.
- **An 8B model invents facts about the user when drafting.** Asked to reply to an email requesting availability and
  notice period, `qwen3:8b` answered with specific days and "15 days", both fabricated and stated as fact. Rules alone
  did not stop it; **two worked examples did** — the same finding as `memory_agent`. The fix is a named-placeholder
  instruction plus examples, producing `[YOUR AVAILABILITY]` instead of a guess.
- **A Gmail draft needs `options.threadId` or it is orphaned.** Without it the draft sits alone in Drafts with no link
  to the email; with it, it appears inside the conversation.
- **`Execute Sub-workflow` stores `mappingMode: defineBelow` with an empty value object, and that is not a bug** when
  the target trigger runs `Accept all data` — n8n skips input mapping and every field passes through.
- **Merge caps at 10 inputs.** An eleventh branch needs a second Merge node chained after the first.
- **Gmail: `Simplify` changes field names, not just size.** On, you get `labels: [{id, name}]`, `Subject`, `From` and
  a ~200-character `snippet` with **no body**. Off, you get `labelIds: ['INBOX', …]`, `subject`, `from.text`, and
  `text` / `html`. Any `IMPORTANT`/`STARRED` guard must match the mode it sits in.
- **Gmail's `snippet` truncates before the decisive sentence.** A rejection opens with "thank you for your application
  for …" and puts the decision after ~200 characters, so every rejection looks like an application update when
  classified from the snippet. `mail_label_draft` therefore runs Simplify **off** and trims the real body to 700 characters
  with URLs stripped.
- **A Code node with nine inbound connections runs nine times, not once.** n8n executes it once per branch that has
  data — visible as "Run 5 of 5" — so any state held in the node (a `Set` of seen IDs) resets each run. A **Merge**
  node in Append mode with `numberInputs` set to the real count collapses them into one execution. Merge defaults to
  **2 inputs** and silently drops the rest while still drawing the connection lines.
- **Text Classifier defaults to discarding unmatched items.** `When No Clear Match` is `Discard Item` unless set to
  `Output on Extra, 'Other' Branch` — the default drops an email with no label and no log.
- **Category-description tuning plateaus around 16–17/20 on an 8B model.** Two rounds helped; a third moved errors
  around rather than reducing them. Tuning works when the mechanism is nameable — a brand name in a description
  outweighing an abstract rule — and shuffles errors when it is not.
- **`Read/Write Files from Disk` never returns zero items.** A pattern matching nothing is reported as
  `{ "error": "No file(s) found" }`. Two consequences: `Always Output Data` is pointless on it, and combining that
  toggle with `onError: continueErrorOutput` lights **both** branches on one failed run. Use
  `continueRegularOutput` and branch on the item instead.
- **Mount sentinels.** `memory/briefing/_mount_check_briefing.md` and `memory/learned/_mount_check_learned.md` must
  always exist. `memory_read` counts them and refuses to report an empty memory store unless both are present —
  without them a broken bind mount is indistinguishable from an empty folder, and the agent would say nothing is
  saved while every memory sits safe on the host. One per mount because the two fail independently, and matched by
  **full path** rather than filename, so a stray copy in the wrong folder cannot satisfy the other mount's check —
  which is why `extract_text` keeps `Keep Source: Both`, since the directory exists only at `binary.data.directory`.
  n8n cannot test for a directory: the file node only globs, the Code node has no `fs`, and `ExecuteCommand` is
  disabled in 2.x.
- **Model in use: `qwen3:8b`.** `qwen3:14b` was tested and rejected on measurement — 12 GB resident, swap at 94%,
  load average 17.4. 24 GB of RAM is the threshold that reopens the 22–24B tier. See `PLAN.md` §4.
- Slack app **T.A.R.S** posts colour-coded block messages: saves and rejections to `#tars_logs`, write failures to
  `#tars_alerts`. n8n's Blocks field accepts an `{ "attachments": [...] }` wrapper despite documenting only
  `{ "blocks": [...] }` — that is the only way to get a colour strip.
- Slack credential note: The credential is an access token stored in n8n, not in this
  repo.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
