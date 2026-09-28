# PLAN.md — Homelab Assistant

Research-backed plan for building a personal assistant on self-hosted n8n, with Obsidian as its knowledge layer. Written
to settle what is actually achievable on the hardware available before any more workflows get built.

---

## 1. The goal, stated once

One assistant, reachable by text and by voice, that knows things about my life, can act on external systems, and gets
more useful over time rather than decaying. Obsidian is the knowledge layer — the place facts live and the place answers
get written back to.

Everything below is measured against that sentence. A capability that does not move toward it does not get built.

---

## 2. Verified constraints

These were measured on this machine during this session, not assumed.

| Constraint                  | Value                                                                                                                                                                               | Consequence                                        |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| RAM                         | 16 GB                                                                                                                        | Caps usable model size. See §4.                    |
| CPU                         | Apple M4                                                                                                       | Metal GPU only via native Ollama, not in Docker.   |
| Model in use                | `qwen3:8b`                                                                          | Benchmarked correct at 20s.                        |
| Docker VM ceiling           | 3.82 GB, reduced from 7.65 GB                                                                                | Docker can no longer claim half the machine.       |
| GPU-available memory        | 11.84 GB of 16 GB shared                                                                                                            | Hard ceiling for Metal inference.                  |
| 14B rejected on measurement | `qwen3:14b` loaded at **12 GB**, spilled 7% to CPU, drove swap to **94%** (15 GB of 16 GB) and load average to 17.4 | 14B is not a working configuration at 16 GB.       |
| Ollama context length       | 16384, set in the Ollama app settings                                                                      | Replaces the 4096 default that truncates silently. |
| n8n version                 | `n8nio/n8n:2.38.7`, up 4 days                                                                                                   | Current.                                           |
| n8n binding                 | `127.0.0.1:5678`                                                                                                                    | Localhost only. No inbound webhooks from internet. |
| n8n Public API              | Enabled — `GET /api/v1/workflows` returns `401`, not `404`                                                 | Endpoint exists and requires authentication.       |
| Vault mounts                | `control_plane:ro`, `memory/briefing:ro`, `dashboard`, `memory/learned`, `files`                                                                                            | `notes/` deliberately not mounted.                 |
| Ollama context default      | 4096 tokens, truncates silently from the start                                                                                             | Hard ceiling on retrieved context.                 |

---

## 3. What the research actually says

Four findings change the design. Each is sourced.

### 3.1 n8n has no native Obsidian integration

A keyword search across the n8n template corpus returned **2 templates** mentioning Obsidian out of 2,352 indexed.
 Neither writes to a vault directly — the
higher-rated one saves Markdown to Dropbox and relies on the user symlinking that folder into their vault.

So Obsidian integration is something to build, not something to install. There are two viable mechanisms, and they are
not equivalent.

**Mechanism A — file mounts.** n8n reads and writes Markdown files on a bind-mounted folder. Already partly in place.
Works whether or not Obsidian is open. No plugin, no token, no extra process.

**Mechanism B — Obsidian Local REST API plugin.** Exposes the vault over HTTPS on port `27124`, with an optional plain
HTTP endpoint on `27123`, authenticated by a bearer token from Settings → Local REST API. Endpoints cover `/vault/{path}` CRUD, `/active/`
for the open file, `/search/simple/` and `/search/`, `/commands/`, and `/tags/`.

Mechanism B is strictly more capable — it can search, list tags, and patch a specific heading or frontmatter key rather
than rewriting whole files.
Its cost is a dependency on the Obsidian desktop app running. The plugin documentation does not state this outright, but
the `/active/` and `/commands/` endpoints only make sense with the app live.

**Decision: both, in sequence.** Mounts first because they have no moving parts and work headless. REST API added later
for search and surgical edits.

### 3.2 Multi-agent is expensive, and the cost is not money

n8n's own production guidance states that multi-agent systems can consume **10–20x the tokens** of a single-agent
workflow without deliberate management.

On a paid API that is a bill. On local Ollama it is wall-clock time and context pressure — and with a 4096-token default
that truncates silently, context pressure means the system prompt gets eaten with no error.

Further stated limits worth designing around:

- Agents with more than **5–7 tools** start making worse decisions and become impossible to debug.
- n8n's guidance: if an agent has 15 tools but uses 3 for most requests, split it into focused agents.
- **Two levels** of sub-workflow nesting is comfortable to debug; three makes error attribution guesswork.
- Set explicit timeouts on every sub-workflow call and agent tool invocation; **30 seconds** is a reasonable start.
  Without them an indefinite wait can occur.
- Cap self-correcting loops at **2–3 iterations**. Beyond that the problem is the prompt, not the retry count.
- Prefer **deterministic routing** with Switch nodes over agent-based routing where the categories are known, because a
  Switch never misroutes from misreading a tool description.

The published "Jarvis" multi-agent template is a central orchestrator delegating to five supervisors — Productivity,
Communication, Lifestyle, Insights, Publishing — each with its own sub-agents. It documents no memory or storage mechanism at all. It is a routing demo, not a memory system.

### 3.3 Local models constrain agent *shape*, not agent *count*

Multi-agent is achievable here. What the hardware constrains is how wide each agent may be, and how many *distinct*
models may be loaded — not how many agents exist.

**Why agent count is not limited by RAM.** An n8n orchestrator calls sub-agents sequentially, passing each one's output
into the next. Every agent
can share one loaded model, so ten agents on `qwen3:8b` cost the same memory as one.

**Why narrow agents are the fix, not the casualty.** Reported behaviour for this model class: single-tool and basic
multi-tool calls are stable with `qwen3:8b`, while complex 3+ tool chains degrade. Qwen3-8B is described as a standard choice for basic multi-tool
calling, scoring 80.4% function-selection accuracy on TaskBench DailyLife. n8n's own guidance points the same way: an agent holding 15 tools but using 3
should be split into focused agents with fewer tools.
Splitting work across narrow agents is therefore the remedy for small-model tool calling, not something blocked by it.

**Measured against this repository, `qwen3:8b` does better than the published tier suggests.** It chained two
`save_memory` calls in one turn, with correct filenames and topics on both — 738ms and 580ms inside a 20.6s run.
 That did **not** come from a rule. The instruction *"call
save_memory twice, once per fact"* produced either a dropped fact or two facts merged into one file; a **worked
example** in the system message produced two correct calls. For this model class, show the shape rather than describe
the outcome.

**Where it does still fall short is fuzzy matching, not chaining.** Deciding that "bump the RAM on my cloud box"
refers to a memory filed as `oracle-server-details` succeeds about four times in five. The information is all present —
`recall_memory` returns every filename — so this is judgement, and more prompt rules bought little. It is handled by
making the failure harmless (date-aware conflict resolution) rather than impossible.

**A router is still not needed.** The split that matters is by **domain**, never by operation: `save_memory` and
`recall_memory` are one domain and belong to one agent, because a single message can need both — *"what's my server IP,
and remember I upgraded the RAM"*. Splitting read from write would send that message down one branch and silently drop
the other half. A router earns its two extra model round-trips when a second domain exists, and `Text Classifier`
returns a category rather than prose to parse.

Published local stacks assign different models per role — for example a 1.5B classifier, a 7B reasoning agent, and a
code-specialised model.
7B–8B models are reported to need 8–16 GB RAM and to suit classification, routing, and summarisation — precisely the
supervisor roles in a multi-agent design.

**The real limits.** Three, and they concern model loading and latency rather than capability:

- **One model, many agents.** `OLLAMA_MAX_LOADED_MODELS` permits concurrent models only where they fully fit in
  available memory; otherwise Ollama evicts the least-recently-used model, and the next request reloads it from disk
  with a 1–3 second penalty. Four distinct models on 16 GB means swap thrashing. Use one shared model; add a second only when measurement
  justifies it.
- **Latency is additive.** Five sequential agents at 20 seconds each is 100 seconds. The 10–20x token multiplier from
  §3.2 shows up as wall-clock time, not a bill.
- **Tool-calling failures are real and documented.** Community reports on n8n AI Agent nodes over Ollama — `qwen2.5:7b`,
  `qwen2.5:14b`, `llama3.1:8b`, `mistral:7b` — describe chains where the second tool never executes, models that
  advertise tool support but reject calls at runtime, and models that reply with prose while making zero tool calls.
 Keeping each agent
  to one or two tools is the mitigation.

**Conclusion: build multi-agent — as many narrow agents sharing one model, not a few wide agents each with its own.**

An earlier draft of this section concluded that multi-agent was unachievable on this hardware. That was wrong. It
applied model-size tiering for a *single wide* agent as though it governed agent *count*, which it does not.

### 3.4 Known failure modes to design against

A field guide catalogues 16 recurring failure modes for n8n + AI + RAG stacks. The ones that apply directly here:

| Mode                          | Applies here because                 | Guard                                                                                                                                   |
|-------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Memory breaks across sessions | Simple Memory is RAM-only            | Durable memory store keyed per conversation, with TTL policy                                  |
| Hallucination & chunk drift   | Stale notes outrank current ones     | Document freshness checks and metadata filters                                                |
| Multi-agent chaos             | Agents overwrite each other's state  | Single source of truth, explicit ownership per phase, idempotent writes                      |
| Prompt injection              | Notes and emails are untrusted input | Input isolation, tool-call whitelists                                                         |
| Bootstrap ordering            | Querying before the index is ready   | Gate the first LLM node on an ingestion-ready flag                                      |
| Debugging is a black box      | Live API calls in every test         | Result Check node after vector search, triggering fallback on empty or near-zero scores |

Note that "multi-agent chaos" has the same remedy already chosen for the vault: **one writer per folder**.

---

### 3.5 This design has prior art, and one known cost

The markdown-files-as-memory pattern is not novel. Independent implementations of the same shape:

| Project | Shape |
|---|---|
| [`memspace`](https://github.com/meaurisch/memspace) | One markdown fact per file, generated indexes, reviewed in a pull request |
| [`agents-memo`](https://github.com/misiekhardcore/agents-memo) | Markdown in an Obsidian vault, retrieved with grep — no embeddings, no vector index |
| [`vaultmem`](https://github.com/jayantak/vaultmem) | Agent memory over an Obsidian vault, bash + ripgrep, no database |
| [`obsidian-memory-for-ai`](https://github.com/jrcruciani/obsidian-memory-for-ai) | Persistent AI memory in Obsidian so the assistant knows you across sessions |

Manus, OpenClaw and Claude Code are reported to use the filesystem as their memory layer in production.

**Deferring the vector store is the documented recommendation, not just ours:** early-stage agents have a curation problem rather than a retrieval problem, and building retrieval before understanding access patterns builds the wrong thing quickly.

**Tracking `memory/learned/` in git is validated by the same sources** — plain text means the agent's knowledge can be committed, reverted when it goes wrong, and branched.

**The one cost this approach carries.** Quoted, because it is the failure mode that would otherwise go unnoticed:

> A markdown file will happily outlive the truth it recorded [...] A memory that names a file, a flag or a workflow that no longer exists is actively harmful. Entries need a date **and something that prompts revalidation.**

The `created` field supplies the date. Nothing yet supplies the revalidation prompt. That gap becomes **Phase 1.5**.

**A second warning, relevant to the Stage 2 design below:** once storage and search are separate systems they can disagree, and a stale index that returns a memory the file already corrected is worse than no memory at all. Any summary index must therefore be **generated, never hand-maintained** — and generated on **create, update and delete**, not only on write. This design deletes stale facts rather than superseding them, so deletion is a first-class operation: an index refreshed only on write would keep returning files that no longer exist, and would keep an obsolete summary for a file edited in place. Either regenerate after every mutation, or rebuild and validate the index before each read.

---

## 4. Architecture decision

Given §3.3, the shape is:

> **Many narrow agents, one shared model. Deterministic routing. Files before databases. Local by default, cloud only
where the local option is proven inadequate.**

Concretely:

- **Start with one agent, split as capabilities are added.** Splitting is expected, not a fallback — it is how small
  models stay reliable.
- **One or two tools per agent.** Not five. 3+ tool chains are where `qwen3:8b` degrades.
- **One shared Ollama model across all agents** — `qwen3:8b` — to avoid LRU eviction and reload penalties.

**Model decision: `qwen3:8b`.** Settled on evidence, not preference. It is instruct-tuned rather than reasoning-tuned,
is named in the tool-calling-trained families, ships in the Ollama library, and is already benchmarked correct at 20s in
`test_chat_LLM`.

Four alternatives were checked against `llmfit` and rejected:

| Candidate                | Why rejected                                                                                                                           |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `qwen3:14b`              | Measured 12 GB resident; swap hit 94%, load average 17.4               |
| Mistral-Small-3.1-24B    | 93.5% memory, 7.2 tok/s, context squeezed 131k→15k, rated Marginal                        |
| Qwen3.5-27B-Opus-Distill | Reasoning model; 82.4% memory; recommended RAM 20.6 GB exceeds the machine              |
| Qwen3.5-9B-Opus-Distill  | Fits at 56%, but reasoning-tuned, MLX runtime not Ollama, and license listed as Unknown |

Two traps worth recording. First, `llmfit` sorts by a Quality subscore that rewards reasoning benchmarks, so
chain-of-thought distills dominate any score-ordered list — and chain-of-thought is precisely what an n8n tool-calling
agent pays to disable.
Second, `llmfit`'s memory estimates assume an idle machine; the measured Ollama footprint for 14B was 12 GB against an
8.2 GB estimate.

**24 GB of RAM is the threshold that reopens the 22–24B tier.** Until then, model shopping is not the lever —
`docs/test_workflows.md` shows three of four failures were tool configuration and system prompt, not the model.

- **Budget latency, not memory.** Count sequential agent hops per request and keep the chain short.
- **Switch nodes** decide which workflow handles an incoming message. The model is not asked to route.
- **Sub-workflows** for each capability, called as tools, one nesting level only.
- **30-second timeouts** on every tool call.
- **Escape hatch:** if a task genuinely needs stronger reasoning, that specific sub-workflow may call a cloud model. The
  decision is per-capability and explicit, never a silent default.

### Folder ownership — extends the existing control-plane rule

| Path               | Writer | Mount                                                                            |
|--------------------|--------|----------------------------------------------------------------------------------|
| `notes/`           | me     | **not mounted** |
| `memory/briefing/` | me     | `:ro`                                                                            |
| `memory/learned/`  | agent  | writable                                                                         |
| `control_plane/`   | me     | `:ro`                            |
| `dashboard/`       | agent  | writable                         |
| `files/`           | both   | scratch                          |

### Memory file format

Adopted because these are cheap now and painful to retrofit across hundreds of files:

- One fact per file. Never one growing log.
- Descriptive filename — the folder listing becomes a table of contents.
- Frontmatter: `created`, `source` (`stated` vs `inferred`), `topic`, `summary`, and `reviewed` once Phase 1.5 exists — absent until a fact is first confirmed, and read as equal to `created` when missing.
- Stale facts are **deleted**, not superseded in place. The `created` date is what keeps the store trustworthy at the
  one-year mark.

Deferred deliberately: category taxonomies beyond a handful of topic values, typed links between notes, and any vector
database. All three are cheap to add later, because indexing runs *over* files that already exist.

---

## 5. Phases

Each phase ends in something demonstrable. No phase starts before the previous one works.

### Phase 0 — Memory substrate *(done)*

- Create `memory/briefing/` and `memory/learned/` in the vault.
- Add both as mounts in `docker-compose.yml` — `briefing` read-only, `learned` writable.
- `docker compose up -d`. Not `restart`, which reuses the old container with old settings.
- **Do not** add `memory/learned/` to the vault `.gitignore`. It is tracked deliberately. The rule in that file is
  "ignore what rebuilds itself" — `dashboard/` regenerates from `control_plane/` on the next run, whereas
  `memory/learned/` accumulates and nothing can recreate it. Committing it is the backup, and `git log` becomes a record
  of what the agent learned and when.

`N8N_RESTRICT_FILE_ACCESS_TO=/vault` already permits these paths; no change needed there.

**Done when:** both folders exist inside the container at `/vault/memory/`.

### Phase 1 — Memory read and write, no database

**Order matters here.** Wire `memory_write` to the agent *before* building `memory_read`. It is one node — a Call n8n Workflow Tool — and it tests the largest untested assumption in the project: whether `qwen3:8b` can reliably fill four tool parameters. If it cannot, the four-field design needs rethinking, and that is worth discovering after one node rather than after two more workflows.

**`memory_read` takes zero parameters. This is deliberate.** Research on Qwen function calling shows the failure mode is malformed *arguments*, not wrong tool selection — one harness moved Qwen models from a 6.75% first-try success rate to 99.8% by making invalid values structurally impossible rather than by prompt tuning. A tool with no parameters has no arguments to malform.

A search or topic parameter is specifically rejected: at this corpus size a filter can only lose information, and a filter that misses is indistinguishable from an empty memory store — the agent would confidently report knowing nothing. Returning everything moves the filtering into the model's reading comprehension, which is the task it is good at.

The narrowing path, when volume demands it:

| Stage | Files | Shape |
|---|---|---|
| 1 — now | 0–30 | `memory_read`, zero parameters, returns everything |
| 2 | 30–100 | `list_memories` (no parameters) returns a **generated** index of filenames and summaries; `get_memory(name)` takes a filename the model just read — copying a value, not inventing one |
| 3 | 100+ | Embeddings and a vector store; matching by meaning rather than keyword, so "server" finds "Oracle box" |

Stage 2 is why every memory file carries a `summary` field from day one.

- Sub-workflow: write one memory file to `memory/learned/`, using `n8n-nodes-base.readWriteFile`, operation "Write File
  to Disk".
- Sub-workflow: read all files from `memory/` and return their contents.
- Attach both to the agent as tools. Saving is **explicit** — triggered by an instruction, never by the model's own
  judgement. Automatic capture fills the store with noise, and noise degrades every later answer.
- Raise `OLLAMA_CONTEXT_LENGTH` before the read tool returns more than a few files.

A Tool variant of `readWriteFile` did not appear in the node search, so the write step likely needs wrapping in a
sub-workflow called via the Call n8n Workflow Tool. To be confirmed against the live node list, not assumed.

**Done when:** I state a fact in chat, and the assistant recalls it in a new conversation the next day.

### Phase 1.5 — Surface stale memories

The revalidation prompt the markdown-memory pattern is documented as needing.

A scheduled workflow scans `memory/`, compares each entry's age threshold against **`max(created, reviewed)`** — treating a missing `reviewed` as equal to `created` — and writes anything older than the threshold to `dashboard/stale_memories.md` for review — or posts a digest to `#tars_logs`. Reuses the control-plane pattern already built and tested in `test_nodes_and_obsidian`.

Nothing is deleted automatically. The workflow surfaces candidates; the human decides.

**Review must persist an outcome, or the same entries resurface forever.** A memory kept because it is still true has an unchanged `created` date, so the next scan selects it again. Two acceptable rules:

1. **Review is destructive.** Keeping a fact means deleting and re-saving it, which sets a fresh `created`. No new field; relies on discipline.
2. **Review is recorded.** Add a `reviewed` date to the frontmatter, and have the scan select on `max(created, reviewed)` rather than `created`. Confirming a fact is then a one-field edit.

**Option 2.** Option 1 makes the cheap action — confirming something is still true — the most destructive one. `reviewed` is absent until first use, so the scan must treat a missing value as equal to `created`.

**Done when:** a memory older than the threshold appears in a review list without being asked for.

### Phase 2 — Obsidian as a first-class surface

- Install the Local REST API plugin, enable it, generate a token.
- From the container, Obsidian is reached at `host.docker.internal:27123` (or `27124` for HTTPS) — the same pattern
  already proven for Ollama.
- Add a search tool backed by `/search/simple/`. This is the first capability the file mounts cannot provide.
- Keep file mounts for writes. They work when Obsidian is closed; the REST API does not.

**Done when:** the assistant answers a question by searching the vault rather than reading everything.

### Phase 3 — Input channels

**Slack, not Telegram.** Changed after building `memory_write`, which already posts failures to `#tars_logs` via the
T.A.R.S bot. One app, one token, many channels — logs and conversation in a surface already open all day.

Outbound works today. **Inbound does not**, and that is this phase's real prerequisite: n8n's Slack Trigger needs Slack
to POST to a reachable Request URL, and n8n is bound to `127.0.0.1:5678`.

Options, in preference order:

1. **Cloudflare Tunnel** — public HTTPS for n8n without opening a router port. Unblocks Slack, webhooks and remote
   access together.
2. **Wait for Phase 5** — Oracle gives a real public instance, but leaves no chat interface until then.
3. **Poll Slack** — a scheduled read of `#tars_chat`. No infrastructure, but laggy and wasteful.

Two-way also needs scopes the T.A.R.S app does not currently hold — `app_mentions:read`, `channels:history`,
`im:history`, `im:write` — plus a **Signature Secret** to verify inbound requests genuinely come from Slack. Neither is
granted before it is used.

Channels are split by urgency rather than by workflow, so the layout stays fixed as workflows multiply: `#tars_chat`,
`#tars_logs` (muted), `#tars_alerts` (notifications on).

Text first, voice second.

- **Voice.** A published local-only template routes Telegram voice messages to a local Whisper ASR endpoint at
  `http://localhost:9000/asr`, then to an Ollama Chat Model, with a Simple Memory node for conversation context. That is the reference implementation to copy.
- **Speech out** is deferred. Piper TTS was archived in October 2025 and is read-only on GitHub; Kokoro TTS and Coqui
  XTTS are the suggested replacements for new projects. Text replies are enough until the rest works.

The referenced Whisper template is Telegram-based, but the transcription half transfers unchanged — a voice file reaches
a local ASR endpoint the same way regardless of which chat surface delivered it.

**Done when:** a voice note from my phone produces a correct answer back in `#tars_chat`.

### Phase 4 — Integrations, one at a time

Each integration is a sub-workflow, added only when a real recurring need exists, and only if it keeps the agent at five
tools or fewer. Candidates in rough order of value: GitHub, calendar, email triage, RSS and reading capture, server and
homelab status checks.

The rule: **build the capability, use it for two weeks, then decide whether it stays.** Anything unused after two weeks
gets deleted.

### Phase 4.1 — Mail organizer

The first Phase 4 integration, and chosen to run **before** Phase 3. Gmail polls outward, so it needs none of the
inbound-webhook work that the chat channels are blocked on.

**Personal Gmail only.** A work mailbox would need an IT conversation about third-party OAuth access before anything is
built.

#### Labels are two independent axes

The first draft mixed a *kind* of mail with a *priority* — `important` alongside `bills` and `newsletter`. A bill is
important, a security alert is important, a rejection is not. Overlapping labels turn classification from a single
choice into a judgement, and single choice is what makes it reliable at 8B.

**Axis 1 — category. Exactly one, always:**

| Label               | What lands there                                       | Archived in v1 |
|---------------------|--------------------------------------------------------|----------------|
| `jobs/application`  | "thanks for your interest", portal confirmations       | no             |
| `jobs/interview`    | invitations, scheduling links, assessments             | no             |
| `jobs/rejected`     | declines                                               | no             |
| `jobs/alert`        | "12 new jobs matching your search"                     | no             |
| `bills`             | invoices, receipts, statements, payment confirmations  | no             |
| `security`          | password resets, login alerts, 2FA, account warnings   | no             |
| `newsletter`        | subscriptions and digests                              | no             |
| `general`           | nothing else fits                                      | no             |

**Nothing is archived in v1 — the column is all `no` on purpose.** A wrong label leaves the mail visible; archiving is
the only step that can hide it. So the archive step ships separately, once the labels are trusted, and the set is cut
to `newsletter` alone. Nothing under `jobs/*` is ever archived, at any stage.

**`jobs/interview` was added during the build**, beyond this plan. Interview mail is frequent and time-sensitive, and
it is the one job state that changes what you do today. `jobs/accepted` was considered and rejected — offers are rare,
and an 8B model over-fires on rare positive categories, so an interview invitation would be labelled as an offer.

**Axis 2 — flags, zero or more, independent:** `needs_reply`.

Nested `jobs/` because job hunting is temporary and three top-level labels would crowd the sidebar permanently; one
parent folds away when it is over.

`security` is deliberately separate from routine service mail. A "new login from an unknown device" and a "your build
finished" come from the same sender and have opposite urgency — archiving the first is how people get compromised.

`general` stays in the inbox for the same reason `memory_write` normalises an unknown topic to `general` rather than
rejecting the save: **the escape hatch must be visible.** Unclassifiable mail that silently archives is an invisible
failure.

Avoid the label name `important` — Gmail has its own `IMPORTANT` marker and the collision confuses both the UI and the
API. Confirm what labels already exist with the Gmail node's `Label → Get Many` before creating any.

#### Archiving is removing a label

Gmail has no archive operation because Gmail has no archive. Archiving *is* removing the `INBOX` label, so it is the
same `Message → Remove Label` node used to add one.

#### Two workflows, one agent

Same split as `control_plane` and `dashboard`: one side acts, the other regenerates, and neither writes the other's
files.

**`mail_label_draft`** — built and published, Schedule Trigger every 30 minutes. It calls **`mail_categorizer`** as a
sub-workflow for the classification half. Full node-by-node detail in `docs/mail_label_draft.md` and
`docs/mail_categorizer.md`; the shape as built:

```
Schedule Trigger (30 min)
  → history_lookup      Gmail · q: label:sorted · 100 · Simplify ON
  → build_sender_map    Code · sender → usual category, 3+ emails and 80% agreement
  → fetch_mail          Gmail · q: -label:sorted in:inbox newer_than:7d · 25 · Simplify OFF
  → trim_body           Code · strip URLs, 700 chars, attach senderHint
  → mail_categorizer    sub-workflow · 9 categories + Other · returns category + labelId
  → dedupe_and_flag     Code · one per id, build the label list
  ├─► apply_labels      Gmail · Add Label
  └─► filter_repliable  Code · category gate + sender address, no model
        → reply_check   Text Classifier · needs_reply / no_reply_needed
        → write_draft   Basic LLM Chain · placeholders, never invented facts
        → create_draft  Gmail · Draft → Create, threaded
```

Four differences from the sketch above, each forced by measurement:

**Limit is 25, not 100.** At ~16–18s per email a batch of 100 takes ~26 minutes, which overruns the schedule interval.
25 finishes in about 7 minutes and still covers 1,200 emails a day.

**Simplify is off, and the body is trimmed.** Gmail's `snippet` stops at ~200 characters, which is *before* the
decisive sentence in a rejection — "thank you for your application for …" arrives first and the decline arrives after.
Every rejection would read as `job_application`.

**A Merge node is required before the Code node.** Nine inbound connections make n8n run a Code node once per branch
with data, so any state it holds resets each time. Merge in Append mode, with `numberInputs` set to 9, collapses them
into one execution. Merge defaults to 2 inputs and silently drops the rest.

**Sender history was pulled forward from "later" to v1.** Template 11518's trick — look at what this sender usually
gets — is the real fix for the job-board-versus-recruiter confusion that description tuning could not solve. Built as
one lookup per run rather than one per email: `history_lookup` runs on the trigger, so it is a single API call whose
result every email reads.

**Schedule Trigger rather than Gmail Trigger.** The Gmail trigger only yields mail that is *new since the last poll*;
this needs "the last hundred, whatever their state", which is a search.

**A `sorted` marker label is the idempotency key.** Every processed message gets it on top of its real label, and the
query excludes it, so nothing is classified twice. Stateless and self-healing — remove `sorted` from a message and it
gets reclassified on the next run. The same trick as the `_mount_check` sentinels: a label's presence encoding a fact
the system cannot otherwise see.

**Text Classifier, not an AI Agent.** It takes categories and routes to one output per category, so an invalid label is
structurally impossible rather than prompt-discouraged. Parsing a label out of free text is the exact failure mode the
rest of this plan is built to avoid.

**`mail_digest`** — Schedule Trigger, daily, writes `dashboard/mail_tracker.md`. Separate from `mail_label_draft`
because a regenerated file must not be rewritten once per email.

**`mail_agent`** — its own chat trigger, its own system message, one zero-parameter tool that reads
`dashboard/mail_tracker.md`. Mail is the second *domain*, which is when the agent split earns its place (§3.3). No
router yet: a router costs an extra model round-trip on every message, and two front doors are cheaper until one
annoys.

**The cron writes, the agent only reads.** One writer per file, the same rule that governs `briefing/` and `learned/`.

#### Drafts are written, never sent

`Draft → Create` only, and only when `needs_reply` is set. A draft cannot leave by accident, which makes this safe by
construction rather than by rule — the same reasoning as the `:ro` mount on `briefing/`.

**Never for `jobs/*`.** A mediocre auto-draft to a recruiter costs a real opportunity, and that is not a place for an 8B
model writing in someone else's voice.

Every draft opens with a line naming what TARS thought the mail was — `[draft by TARS — reason: asked for
availability]` — deleted before sending. It stops a half-written draft going out unnoticed and makes a misclassification
visible at the moment it matters.

Expect scaffolding, not finished prose: a greeting, a structure, the obvious acknowledgement. Worth it for the
blank-page problem, not worth it if the expectation is send-ready replies.

#### `dashboard/mail_tracker.md`

Counts per label, then collapsible cards for anything needing attention — the Jira behaviour of a title line that
expands into detail:

```markdown
## needs_reply (3)

<details>
<summary><b>Interview availability</b> — talent@acme.com · 2h ago</summary>

Asking which days next week suit a 45-minute technical round. Draft created.

[Open in Gmail](https://mail.google.com/mail/u/0/#inbox/MESSAGE_ID)
</details>
```

**Obsidian renders `<details>` natively — no plugin.** A real Kanban board would need the Obsidian Kanban plugin, which
reads a specific markdown layout that n8n can generate, but the board view only appears if the plugin is installed. The
cron owns the file either way, so the same data can be regenerated into that shape later without touching anything
else.

**Done when:** a day's mail lands under the right labels, the inbox holds only what stays, and `mail_tracker.md` opens
to a list of what actually needs answering.

**Both build-time questions are now answered.**

`Simplify` **does** carry the labels, but under a different name and shape in each mode: `labels: [{id, name}]` when
on, `labelIds: ['INBOX', …]` when off. One Gmail node plus a Code node produces the count table either way — no
per-label search needed.

The Gmail deep link has no reliable form. The token in a real Gmail URL is a *view token* that maps to nothing in the
API. `https://mail.google.com/mail/u/0/#all/{messageId}` works in practice and thread links need hex conversion, so
`mail_tracker.md` links will be best-effort rather than guaranteed.

**Status:** `mail_label_draft` and `mail_categorizer` are live, labelling and drafting. The `needs_reply` label, the
archive step, Slack logging, `mail_digest` and `mail_agent` are unbuilt.

**Drafts turned out to need three gates, not one.** A category gate and a sender-address check, both deterministic,
cut a 25-email batch to 0–2 before any model runs — which is what makes the two model calls after them affordable. A
single LLM pass over every email, as the one community precedent does it, costs 25 calls to answer a question that two
lines of code answer for 23 of them.

**An 8B model invents facts about the user when drafting**, and rules alone do not stop it. Asked to reply to an email
requesting availability and notice period, `qwen3:8b` answered with specific days and "15 days", both fabricated. Two
worked examples fixed it — the same lever that fixed two-call chaining in `memory_agent`.

### Phase 5 — Oracle Cloud via Coolify

Unchanged from the original roadmap: Postgres instead of SQLite, real webhooks, SSL, persistent memory.

Note for planning: Oracle's ARM free tier has no GPU, so inference there would be CPU-bound. Ollama stays on this Mac,
and the Oracle instance reaches it only if a tunnel exists. This needs deciding before migration, not during.

### Phase 6 — Split into a multi-agent network

Expected, not hypothetical. Once Phase 4 has several capabilities attached, the single agent will exceed one or two
tools and start misrouting. That is the trigger to split.

Shape when it happens:

- A **Switch node** does first-level routing on message category, so the model is not asked to route.
- Each capability becomes its own sub-workflow agent with one or two tools, all sharing `qwen3:8b`.
- **One nesting level** — orchestrator calls specialist, specialist does not call another specialist.
- 30-second timeout on every call; structured `success` field in every response so the caller can route failures.

**Done when:** a request that needs two different capabilities is handled correctly without either agent seeing the
other's tools.

---

## 6. Explicitly not doing

Recording these so they do not get relitigated.

| Not doing                                 | Why                                                                                                                                            |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Multi-agent *before* Phase 1 works        | Splitting is planned (§3.3), but a single memory tool has to work before there is anything to split                                   |
| A separate Ollama model per agent         | LRU eviction and 1–3s reload penalties on 16 GB |
| Vector database (Qdrant, pgvector)        | Memory store starts at zero files; indexing runs over files that already exist, so nothing is lost by waiting                         |
| 16-category note taxonomy                 | A filing system for a corpus that does not exist yet                                                                                  |
| Typed links between notes                 | n8n vector search does not traverse links; no payoff in this stack                                                                    |
| Mounting `notes/`                         | Personal learning journal, deliberately private                               |
| Automatic memory capture                  | Fills the store with noise that degrades later answers                                                                                |
| Cloud vector stores (Neon, MongoDB Atlas) | Notes would leave the machine                                                                                                         |

---

## 7. Open decisions

1. **Cloud escape hatch** — is a cloud model acceptable for specific sub-workflows where local 8B demonstrably fails, or
   is local-only absolute? This changes what Phase 4 can contain.
2. **What goes in `memory/briefing/` on day one** — the handful of facts the assistant should know before it learns
   anything.
3. ~~**Larger local model**~~ — **closed.** `qwen3:14b` was tested and rejected on measurement: 12 GB resident, 94%
   swap, load average 17.4. See §4.

---

## 8. Immediate next actions

1. ~~Wire `memory_write` into the agent.~~ **Done, and the assumption held.** Built as a new workflow, `memory_agent` (named `tars_agent` until 2026-09-26), rather than bolting production tools onto the `test_chat_LLM` scaffold. `qwen3:8b` filled all four tool parameters correctly on an explicit request, and declined to call the tool on an unrelated question. Published guidance places the comfortable tool-calling tier at 22B and above, so this was not a given — the four-field design stands. See `docs/memory_agent.md`.
2. ~~**Build `memory_read`**~~ **Done — nine nodes, zero parameters.** The empty case turned out not to work as predicted: `Read/Write Files from Disk` never returns zero items, it returns `{ "error": "No file(s) found" }`, so `Always Output Data` guards nothing and combining it with `onError: continueErrorOutput` fires both branches at once. The node runs `continueRegularOutput` and an `If` branches on the item instead. See `docs/memory_read.md`.
3. ~~Wire `memory_read` to the agent.~~ **Done.** `recall_memory`, no parameters, alongside `save_memory`.
4. ~~Test the full loop across a container restart~~ **Done and passing** — a fact stated before `docker compose restart` is recalled after it.
5. ~~**Phase 4.1 — build the mail organizer.**~~ **Done, labels and drafts.** `mail_label_draft` on a 30-minute schedule, calling `mail_categorizer` for classification. It labels every fetched email and writes a Gmail draft for the one or two a person is waiting on. Nothing is archived, deleted or sent. Classification accuracy sits around 16–17 of 20 and plateaued after two rounds of category-description tuning, which is why sender history was pulled forward from "later" into v1. See `docs/mail_label_draft.md` and `docs/mail_categorizer.md`.

6. ~~**Slack logging and failure alerting.**~~ **Done.** `label_success` and `draft_success` post green cards to `#tars_logs`; five nodes route failures to a red `error_logs` card on `#tars_alerts`; and `error_handler`, a two-node workflow on an Error Trigger, catches everything unhandled across all five production workflows. The Gmail nodes retry three times first. See `docs/error_handler.md`.

7. **Finish the mail domain.** *Next action*, in this order: the archive step for `newsletter` only, once the labels have been watched for a week; then `mail_digest` → `dashboard/mail_tracker.md` and `mail_agent`. The `needs_reply` label was dropped — Gmail already shows a Draft chip on any thread carrying one.

8. **Phase 1.5 — surface stale memories.** Now load-bearing rather than optional: updating a memory is about 80% reliable when the user's wording differs from the stored summary, and the other 20% leaves a stale near-duplicate on disk. The scan is what catches those.

   **Scope extended to cross-folder contradictions.** `briefing/` holds facts you wrote; `learned/` holds facts the agent saved. Nothing syncs between them and nothing should — the `:ro` mount on `briefing/` is what stops a model that misidentifies the target file one time in five from overwriting your own notes. But the agent can learn something in `learned/` that contradicts what you wrote in `briefing/`, and today nothing tells you.

   Build it deterministic first, with no model at all: a scheduled workflow reads `/vault/memory/**/*.md`, groups by the closed `topic` set, and flags every topic holding entries in **both** folders where the `learned/` entry is newer. That alone catches the case worth catching, with no false positives from model reasoning. Write the list to `dashboard/stale_memories.md` and post an amber Slack card only when it is non-empty.

   A model pass is the second step, and only on pairs the deterministic stage already flagged — never the whole corpus. One pair per call, closed output set (`CONTRADICTS` / `CONSISTENT`) plus one line of explanation. That is the shape `qwen3:8b` handles well, and a nightly batch has no user waiting on it, so a wrong answer lands in a review file rather than in an answer you act on.

   Two rules from the start: **never auto-edit and never auto-delete** — the output is a report, and `briefing/` being `:ro` enforces half of that for free. And compare against `max(created, reviewed)` so a pair you have checked and stamped stops reappearing every night.
9. Phase 3 — replace the chat trigger with Slack.

---

## 9. Sources

- [n8n Blog — Production AI Playbook: Complex Agent Patterns](https://blog.n8n.io/production-ai-playbook-complex-agent-patterns/)
- [n8n template 6186 — Jarvis: AI personal assistant with multi-agent network via WhatsApp](https://n8n.io/workflows/6186-jarvis-ai-personal-assistant-with-multi-agent-network-via-whatsapp/)
- [n8n template 6012 — Create a privacy-focused AI assistant with Telegram, Ollama, and Whisper](https://n8n.io/workflows/6012-create-a-privacy-focused-ai-assistant-with-telegram-ollama-and-whisper/)
- [n8n template 3886 — Send YouTube Video Summaries to Obsidian via Dropbox](https://n8n.io/workflows/3886)
- [coddingtonbear/obsidian-local-rest-api](https://github.com/coddingtonbear/obsidian-local-rest-api)
- [DEV Community — The Hidden Failure Modes in n8n/Make/Zapier with AI & RAG](https://dev.to/onestardao/-the-hidden-failure-modes-in-n8n-make-zapier-with-ai-rag-a-field-guide-mit-problem-42kj)
- [n8n Community — Tool calling chain with local Ollama models (7b/14b)](https://community.n8n.io/t/tool-calling-chain-with-local-ollama-models-7b-14b-2nd-tool-never-executed/280320)
- [Local LLM Tool Calling (2026): Reliable Function Calls, GBNF Grammars & Fixes](https://llmconfigurator.com/en/guides/coding-agents/tool-calling-local-models)
- [Ollama FAQ — model loading and memory](https://docs.ollama.com/faq)
- [memspace — one markdown fact per file](https://github.com/meaurisch/memspace)
- [agents-memo — markdown memory in an Obsidian vault, grep retrieval](https://github.com/misiekhardcore/agents-memo)
- [vaultmem — agent memory over an Obsidian vault, no database](https://github.com/jayantak/vaultmem)
- [obsidian-memory-for-ai](https://github.com/jrcruciani/obsidian-memory-for-ai)
- [Multi-Agent Memory Without a Vector Database](https://dev.to/whoffagents/multi-agent-memory-without-a-vector-database-the-markdown-first-approach-2lo0)
- [AI Agent Memory Management — When Markdown Files Are All You Need](https://dev.to/imaginex/ai-agent-memory-management-when-markdown-files-are-all-you-need-5ekk)
- [The Markdown File That Beat a $50M Vector Database](https://dev.to/codingsimba/the-markdown-file-that-beat-a-50m-vector-database-separating-storage-and-search-in-agent-memory-3d14)
- [Function Calling Harness: From 6.75% to 100%](https://typia.io/blog/function-calling-harness-qwen-meetup-korea/)
- [Qwen function calling documentation](https://qwen.readthedocs.io/en/latest/framework/function_call.html)
- [Run Multiple Ollama Models: Parallelism and Memory Tuning](https://eastondev.com/blog/en/posts/ai/20260406-ollama-multi-model-deployment/)
- [Best Ollama Models for n8n AI Agents (2026 Guide)](https://www.iloveblogs.blog/guides/n8n-ai-agent-workflows-ollama)
- [Best Ollama Model for Tool Calling Agent 2026: Comparison & Benchmarks](https://webscraft.org/blog/yaku-model-ollama-obrati-dlya-agenta-z-tool-calling-porivnyannya-i-benchmarki?lang=en)
- [Building your first multi-agent system with n8n](https://medium.com/mitb-for-all/building-your-first-multi-agent-system-with-n8n-0c959d7139a1)
- [Wade11s/n8n-nodes-obsidian](https://github.com/Wade11s/n8n-nodes-obsidian)
- Local measurements this session: `ollama list`, `sysctl -n hw.memsize`, `docker compose ps`,
  `curl http://localhost:5678/api/v1/workflows`
- In-repo: `docker-compose.yml`, `CLAUDE.md`, `README.md`

