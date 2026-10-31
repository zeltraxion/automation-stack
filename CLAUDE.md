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
- **An Ollama upgrade can orphan a model that still appears in `ollama list`.** After one upgrade, every request
  returned `400 - 'nimble:9b' was downloaded in a format that is no longer supported. To re-download, run
  'ollama pull nimble:9b'`. Nothing else signalled it: `ollama list` still showed the model and `/api/tags` still
  answered. Three nights of mail went unlabelled and every execution reported `success`, because the failing node
  used `continueErrorOutput` into a branch wired to nothing. **An error output connected to nothing is not error
  handling — it is a silent drop**, and it hides the outage it was supposed to surface. Check the funnel, not the
  status: item counts falling to zero mid-workflow is the signal a green execution will not give you.
- **n8n 2.x disables `ExecuteCommand` and `LocalFileTrigger` by default**, and Code nodes can no longer read environment
  variables. Relevant if a workflow needs to shell out or watch a folder.
- **The port is bound to `127.0.0.1`.** Removing that prefix exposes n8n to every device on the local network; doing so
  also requires a TLS reverse proxy in front of n8n. `N8N_SECURE_COOKIE=false` makes login work over plain HTTP,
  but it strips the Secure attribute from the auth cookie, so the session token travels in clear text and any
  device on the network can read it. Use it only for throwaway local testing — never as the way to expose n8n.

## Current state

- **Git repository**, currently on `main`; PR #7 merged.
- `VAULT_PATH` is referenced on **five lines** of `docker-compose.yml`. The vault is mounted: `control_plane` and
  `memory/briefing` read-only, `dashboard`, `files` and `memory/learned` writable. `notes/` is deliberately not mounted.

- `n8n_workflows/` holds ten exports: `memory_agent.json`, `memory_write.json`, `memory_read.json`,
  `mail_label_draft.json`, `mail_categorizer_jev.json`, `mail_categorizer.json`, `error_handler.json`,
  `read_switches.json`, `test_chat_LLM.json`, `test_nodes_and_obsidian.json`.

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
- **The mail domain is live.** `mail_label_draft`, published, Schedule Trigger once a day at 21:00 IST, calls
  `mail_categorizer_jev` as a sub-workflow. It labels every fetched email into one of nine categories, and writes a Gmail
  draft for the one or two a person is actually waiting on. **Nothing is archived, deleted or sent** — drafts only,
  deliberately, until accuracy is trusted. Documented in `docs/mail_label_draft.md` and
  `docs/mail_categorizer_jev.md`.
- **Classification runs on a decision model, not a Text Classifier.** `mail_categorizer_jev` posts to Ollama's
  `/v1/systemone` with nine named options and gets back a choice plus a probability for every option. Three nodes
  replace fourteen: the ten `cat_*` Set nodes and the Merge existed only because a Text Classifier routes instead of
  returning data. The old `mail_categorizer` stays published as the rollback — one field on the Execute Sub-workflow
  node, since both return the same `category` and `labelId`.
- **A confidence number is the point, not the accuracy gain.** Measured on 25 real emails: 22 labelled, 3 held back,
  **0 confidently wrong** — against 16–17 of 20 with 3–4 wrong labels silently applied. Below a 0.6 floor the model's
  pick is discarded and the email gets no label at all, so `-label:sorted` finds it again on the next run.
- **Shorter category descriptions made it both faster and more accurate.** Cutting the nine from ~1,800 to ~550 tokens
  halved wall clock (1m42s → 48s on six emails) and raised confidence. The long versions carried cross-references and
  tie-break rules written to steer a text generator; a decision model scores all nine at once, so the probabilities
  *are* the tie-break. Concrete nouns and brand names carry the weight.
- **`nimble:9b` over `tev1:4b`, on context not quality.** A request with the original descriptions measured 2,195
  input tokens, past tev1's ~2,000 context and past the ~1,500 it was trained on — and Ollama truncates silently.
  nimble holds 8,192. It fits at `100% GPU` with `offloaded 33/33 layers`, but Ollama logs
  `reason=metal_partial_offload` and disables mmap, so the fit has no headroom and the cold load costs ~9 s.
- **`keep_alive` belongs in the request body, and `0` is the wrong value.** The timer resets on every request, so a
  short value never unloads mid-batch — it only shortens the idle tail. `0` unloaded after each email and made all 25
  requests pay the reload; `5m` leaves 8.9 GB resident long after work ends. `"30s"` is the setting.
- **`Items per Batch: 1` on any HTTP Request hitting Ollama.** The default fires several at once, Ollama serves one
  model at a time, and the queued requests burn the 5-minute HTTP timeout. Seven of 25 failed that way, scattered
  through the batch rather than at the tail — which is how you tell concurrency from slowness.
- **Downstream of `Continue (using error output)`, a Code node must use `.item`, never `.all()[i]`.** The error
  branch removes items from the success stream, so reply *n* is no longer email *n*. Position pairing silently shifts
  every answer after the first failure — in one run it put the right categories on the wrong emails. `.item` follows
  n8n's paired-item link, and it only resolves in `Run Once for Each Item` mode.
- **Sender history is a better fallback than a guess.** When confidence is below the floor, `map_label` reads the
  `senderHint` sentence the caller built and uses that category instead. `build_sender_map` only records a sender at
  3+ labelled emails with 80% agreement, so it is measured inbox history. Recovered three emails where the model
  picked correctly at 0.41–0.60.

- **The vault control plane is live.** `control_plane/switches.md` holds one boolean property per scheduled
  workflow, rendered as a checkbox by Obsidian's Properties plugin. `read_switches` parses it and answers one
  question — `{ workflow, enabled }` — and a caller gates itself with two nodes, `check_switch` plus an IF.
  `mail_label_draft` is the only caller so far; a gated run ends green in 146 ms. Documented in
  `docs/control_plane.md`.
- **It fails open, and that is a per-workflow decision rather than a default.** A missing or unreadable file lets
  everything run, which takes three things: `onError: continueRegularOutput` on both file nodes and `?? ''` in the
  parser. Correct here because an unwanted run of `mail_label_draft` only writes labels and drafts, while a silent
  stop goes unnoticed for up to a day. Anything that archives, deletes, sends or spends should invert the last line
  of the parser to `=== true`.
- **The switch file is the first thing both n8n and Hermes read.** n8n reaches it through the `:ro` bind mount at
  `/vault/control_plane/switches.md`; `mail-watchdog.py` runs natively and opens the same file at its real path. A
  paused workflow gets a grey card in `#tars_logs` instead of an amber alert, and the watchdog exits before calling
  the n8n API so that pausing mail and stopping Docker together stays quiet.
- **`error_handler` is the shared failure alert.** Three nodes — Error Trigger into a red Slack card on `#tars_alerts`
  naming the workflow, the failing node and a link to the execution. Set as the Error Workflow on all five production
  workflows; deliberately not on itself, since an error handler that errors calls itself. Documented in
  `docs/error_handler.md`.
- **`error_handler` also pushes to Hermes.** A third node, `push_to_hermes`, POSTs the same four facts to
  `http://host.docker.internal:8644/webhooks/n8n-failure`, a Hermes inbound webhook route that renders them into a
  Slack DM with no agent and no model loaded (`deliver_only`). It is chained **after** the Slack node, not parallel
  to it, so a dead gateway cannot cost the red card, and carries `onError: continueRegularOutput` for the same
  reason. The route lives in `~/.hermes/config.yaml` under `platforms.webhook.routes`, not in this repository.
  Documented in `docs/error_handler.md`.
- **Downstream of any node, `$json` is that node's output — not the trigger's.** `push_to_hermes` runs after the
  Slack node, where `$json` is the Slack API response (`{ok, channel, ts, …}`), so every field resolved to its `??`
  fallback and the DM said `unknown` / `no message` while Hermes returned 200 and the execution stayed green.
  `$('Error Trigger').item.json` reaches past the intervening node. The same trap as the `.all()[i]` pairing bug,
  from the opposite direction: there the items shifted, here the source did.
- **A node set to `Continue (using error output)` does not fail the execution, so `error_handler` never sees it.**
  That is the handover between the two logging styles: per-node error outputs report through the workflow's own Slack
  card, `error_handler` covers everything unhandled. Five nodes in `mail_label_draft` work that way.
- **The four Gmail nodes retry three times at two-second intervals.** A dropped connection to Google is exactly what
  retry exists for — an alert now means three consecutive failures, not one bad response.
- **Four gates stand between a fetched email and a model call in the draft branch:** a category gate, a bulk-header
  check, a sender-address check, then a Text Classifier. The first three are deterministic and cut 25 emails to 0–2,
  which is what makes the two model calls after them affordable.
- **`List-Unsubscribe` is the general bulk signal; a domain list is not.** One night's inbox carried 17 distinct
  sender domains, so any hardcoded list of platforms is obsolete the week it is written. `trim_body` sets
  `isBulk` from the `list-unsubscribe`, `list-id` and `feedback-id` headers instead — measured at 20 of 25,
  against 19 for the `BULK_WORDS` local-part check. The two overlap but neither contains the other, so both run.
  `feedback-id` is the loosest of the three: Google defines it for Postmaster Tools, so it means *sent by bulk
  infrastructure* rather than *this is a mailing list*. Amazon and Groww stamp transactional notices with it and
  set no `list-unsubscribe` at all, which is the gap it closes. `headers` does not
  survive `trim_body`, which rebuilds the item, so the flag must be computed there and carried as a field.
  Senders that set no bulk header at all exist — HSBC marketing is one — but they never reach the gate, because
  their category is not in `REPLYABLE`.
- **An email the categorizer could not decide gets no label whatsoever.** Not even `sorted`. `dedupe_and_flag`
  drops it, so `-label:sorted in:inbox newer_than:7d` finds it again the next night and it ages out of the window
  by itself after seven days. The marker has to be the outcome, never the fact that the workflow ran — tagging on
  execution marked the emails the model understood least as permanently handled.
- **An 8B model invents facts about the user when drafting.** Asked to reply to an email requesting availability and
  notice period, `qwen3:8b` answered with specific days and "15 days", both fabricated and stated as fact. Rules alone
  did not stop it; **two worked examples did** — the same finding as `memory_agent`. The fix is a named-placeholder
  instruction plus examples, producing `[YOUR AVAILABILITY]` instead of a guess.
- **A Gmail draft needs `options.threadId` or it is orphaned.** Without it the draft sits alone in Drafts with no link
  to the email; with it, it appears inside the conversation.
- **`Execute Sub-workflow` stores `mappingMode: defineBelow` with an empty value object, and that is not a bug** when
  the target trigger runs `Accept all data` — n8n skips input mapping and every field passes through.
- **Merge caps at 10 inputs.** An eleventh branch needs a second Merge node chained after the first. No longer binding
  on the mail domain — `mail_categorizer_jev` returns the category as data, so there are no branches to merge.
  `travel` (`Label_14`) stays unused by choice: a tenth category would cost prompt tokens on every request to catch
  mail that arrives a few times a year.
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
  outweighing an abstract rule — and shuffles errors when it is not. **The plateau was partly an instrument problem:**
  a text generator returns a word and no measure of how sure it was, so every round was tuned blind. With
  probabilities visible, two added clauses fixed two long-standing misses on the first attempt.
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
- **Models in use: `qwen3:8b` for generation, `nimble:9b` for classification.** `qwen3:14b` was tested and rejected on
  measurement — 12 GB resident, swap at 94%, load average 17.4. 24 GB of RAM is the threshold that reopens the 22–24B
  tier. The two current models cannot be resident together on 16 GB, which is why `nimble` carries `keep_alive: "30s"`
  and gets evicted between runs. See `PLAN.md` §4.
- **Each question re-reads the whole prompt, so questions are not free.** Ollama's docs: *"Every question is scored
  with the full state and question set in its prompt."* Two questions prefill the state and all nine criteria twice,
  so `input_tokens` is roughly double the material actually sent. Budget a question like a separate request that
  happens to share a round trip.
- **Do not move work in front of a deterministic gate.** `mail_categorizer_jev` carried a `needs_reply` question at
  first, on the reasoning that one request answering two questions beats two model calls. Wrong: `reply_check` runs
  *after* `filter_repliable` has cut 25 emails to 0–2 with no model at all, so folding it in would trade two model
  calls for twenty-five extra prefills. The question was removed. The four gates are what make the draft branch
  affordable.
- **Ollama 0.35 added `/v1/systemone` for decision models.** Up to 64 named questions about one `state`, of type
  `choice` (2–26 options), `noul` (yes/no with a probability) or `score`. Request bodies cap at 64 KiB. n8n has no node
  for the endpoint, so it is a plain HTTP Request to `http://host.docker.internal:11434` — local, no credential.
- **Build the JSON body with `JSON.stringify`, not raw interpolation.** Email bodies carry quotes and newlines;
  `{{ $json.body }}` dropped straight in breaks the body with *"The value in the JSON Body field is not valid JSON"*.
  `JSON.stringify` supplies the surrounding quotes, so the template has none.
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
