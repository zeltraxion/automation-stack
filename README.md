<div align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=1F1B2E&height=160&section=header&text=n8n-project&fontSize=48&animation=fadeIn&fontAlignY=36&fontColor=EA4B71&desc=Self-Hosted%20Automation%20%7C%20Local%20First%2C%20Cloud%20Later&descAlignY=56&descColor=ffffff" width="100%"/>
</div>

<h3 align="center">
  <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=24&duration=3500&pause=1000&color=EA4B71&center=true&vCenter=true&width=620&lines=Workflow+automation+on+my+own+hardware;Docker+Compose+%2B+SQLite+%2B+local+LLMs;No+API+bills.+No+data+leaving+the+machine.;Destination%3A+Oracle+Cloud+via+Coolify" alt="Typing SVG" />
</h3>

<div align="center">
  <img src="https://img.shields.io/badge/n8n-1F1B2E?style=for-the-badge&logo=n8n&logoColor=EA4B71" alt="n8n" />
  <img src="https://img.shields.io/badge/Docker-1F1B2E?style=for-the-badge&logo=docker&logoColor=EA4B71" alt="Docker" />
  <img src="https://img.shields.io/badge/Ollama-1F1B2E?style=for-the-badge&logo=ollama&logoColor=EA4B71" alt="Ollama" />
  <img src="https://img.shields.io/badge/SQLite-1F1B2E?style=for-the-badge&logo=sqlite&logoColor=EA4B71" alt="SQLite" />
  <img src="https://img.shields.io/badge/Coolify-1F1B2E?style=for-the-badge&logo=coolify&logoColor=EA4B71" alt="Coolify" />
</div>

<br/>

<table align="center" style="border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="58%" valign="top" style="border: none; padding-right: 20px;">
      <h3>🧩 What This Is</h3>
      <p>A self-hosted <strong>n8n</strong> instance and the work running on it. The compose file and environment template define the infrastructure; <code>n8n_workflows/</code> holds the exports, <code>scripts/</code> the few things that deliberately run <em>outside</em> Docker, and <code>docs/</code> a page per production workflow.</p>
      <ul>
        <li>🐳 <strong>Runtime</strong>: Docker Compose, single service</li>
        <li>💾 <strong>Storage</strong>: SQLite on a named volume</li>
        <li>🧠 <strong>Models</strong>: local via Ollama — no API bill, works offline</li>
        <li>🗄️ <strong>Control surface</strong>: an Obsidian vault, one writer per folder</li>
        <li>🔔 <strong>Alerting</strong>: Slack, plus a watchdog that outlives Docker</li>
        <li>🔒 <strong>Exposure</strong>: loopback only — nothing on the LAN</li>
        <li>☁️ <strong>Going to</strong>: Oracle Cloud Free Tier, via Coolify</li>
      </ul>
    </td>
    <td width="42%" valign="top" align="center" style="border: none;">
      <h3>⚙️ Stack</h3>
      <img src="https://skillicons.dev/icons?i=docker,sqlite,python,nodejs,ts,githubactions,bash&theme=dark" alt="Stack" />
      <br/><br/>
      <img src="https://img.shields.io/badge/LangChain-1F1B2E?style=flat-square&logo=langchain&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Obsidian-1F1B2E?style=flat-square&logo=obsidian&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Oracle_Cloud-1F1B2E?style=flat-square&logo=oracle&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Slack-1F1B2E?style=flat-square&logo=slack&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Hermes_Agent-1F1B2E?style=flat-square&logo=gnometerminal&logoColor=EA4B71" />
    </td>
  </tr>
</table>

---

<h2 align="center">🚀 Quick Start</h2>

**Prerequisites:** Docker Desktop running, and `docker compose version` reporting v2 or newer.

```bash
# 1. The volume is created by hand — it is declared `external` on purpose,
#    which also means `docker compose down -v` can never destroy it.
docker volume create n8n_data

# 2. Configure
cp env.example .env
openssl rand -hex 32          # paste into N8N_ENCRYPTION_KEY, and into your password manager
# also set VAULT_PATH to the ABSOLUTE path of your Obsidian vault — "~" is not expanded here

# 3. Run
docker compose up -d
docker compose logs -f n8n     # wait for "Editor is now accessible via..."
```

Then open **http://localhost:5678**

> **Set the encryption key before the first boot.** n8n generates its own if you don't, writes it into the volume, and
> then refuses to start if you later supply a different one.

---

<h2 align="center">🗂️ Layout</h2>

```
automation-stack/
├── docker-compose.yml     # infrastructure definition — the shape
├── env.example            # variable template — no values
├── .env                   # your values (gitignored)
├── PLAN.md                # researched decisions and measured constraints
├── CLAUDE.md              # operating guidance, and every finding worth keeping
├── docs/                  # a page per production workflow — start at docs/index.md
├── n8n_workflows/         # exported workflow JSON
├── scripts/               # Python that runs outside Docker, mirrored to ~/.hermes/scripts/
├── custom-nodes/          # future TypeScript nodes
└── .github/workflows/     # future CI
```

The split between `docker-compose.yml` and `.env` is deliberate: the compose file describes *what the infrastructure
looks like*, and `.env` holds *what differs per machine*. Coolify reads the former from git and injects the latter from
its own UI — so keeping values out of the compose file is what makes the migration a non-event.

---

<h2 align="center">⚡ What Actually Runs</h2>

Every production workflow has its own page in [`docs/`](docs/index.md). This is the map.

| Workflow | Trigger | What it does |
|---|---|---|
| `mail_label_draft` | daily, 21:00 IST | Labels inbox mail into nine categories and writes a draft reply for the one or two genuinely waiting on you. **Never archives, deletes or sends** |
| `mail_categorizer_jev` | sub-workflow | The classifier. A decision model scores nine options at once and returns a probability for each; below a 0.6 floor the pick is discarded |
| `memory_agent` | chat | Saves and recalls facts as markdown in the vault, through two tool sub-workflows |
| `memory_write` · `memory_read` | sub-workflows | The two halves of the memory loop |
| `read_switches` | sub-workflow | Reads the vault control plane and answers one question: should this workflow run? |
| `error_handler` | Error Trigger | The shared failure alert — a red Slack card naming the workflow, the failing node and a link to the execution |

Two things run **outside** Docker on purpose:

| | What it is | Why not in n8n |
|---|---|---|
| `scripts/mail-watchdog.py` | A Hermes cron job at 07:00. One Slack card a day saying whether `mail_label_draft` actually ran | A watchdog that dies with the thing it watches is not a watchdog. `error_handler` needs an execution to fire against, so it is structurally blind to a run that never started |
| Hermes gateway | A Slack-facing agent in `#tars_chat`, on a local model | Socket Mode opens an outbound WebSocket, so there is no public URL to expose and no tunnel to maintain |

Alerts land in two Slack channels and the colour carries the meaning:

| Channel | Gets |
|---|---|
| `#tars_logs` | green *healthy*, grey *paused* — things that went to plan |
| `#tars_alerts` | amber *stale*, red *failed* — things that did not |

Exactly one watchdog card arrives each morning, which is what makes **nothing in either channel** a signal in its own right: the watchdog itself did not run.

---

<h2 align="center">🔐 The Vault Mounts</h2>

An Obsidian vault is mounted in as the control surface — you command from `control_plane`, n8n reports into `dashboard`,
you brief the agent in `memory/briefing`, the agent remembers into `memory/learned`,
and each folder has exactly one writer.

| Folder             | You                            | n8n           | Mounted as    |
|--------------------|--------------------------------|---------------|---------------|
| `notes/`           | read + write                   | **no access** | *not mounted* |
| `control_plane/`   | read + write — the switches    | read only     | `:ro`         |
| `memory/briefing/` | read + write — facts you give  | read only     | `:ro`         |
| `dashboard/`       | read only — generated output   | read + write  | writable      |
| `memory/learned/`  | read + delete — agent's memory | read + write  | writable      |
| `files/`           | read + write                   | read + write  | writable      |

Three separate mechanisms are at work, and each protects something different — they do not stack on the same folder:

| Mechanism                                | What it actually protects                                                                                                                                                                                        |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Which folders are mounted**            | `notes/` is never mounted, so it does not exist inside the container. This is the **only** thing protecting your notes.                     |
| **The `:ro` mount option**               | `control_plane/` is mounted read-only, so writes to it fail at the filesystem level, for every process in the container. |
| **`N8N_RESTRICT_FILE_ACCESS_TO=/vault`** | Workflow file operations are limited to paths under `/vault` — not `/tmp`, not n8n's own config directory.                                                   |

Two consequences worth being precise about. The allowlist grants `/vault` as a whole, so on its own it would happily
permit writes to `control_plane/` — `:ro` is what prevents that. And neither the allowlist nor `:ro` has any bearing on
`notes/`; that folder is safe purely because it was never handed to the container. Mount it and it becomes reachable,
allowlist and all.

`control_plane/switches.md` is where that read-only mount earns its keep. One boolean property per scheduled
workflow, rendered as a checkbox by Obsidian, and a workflow reads it before doing anything. Untick the box and
tonight's run ends in 146 ms without touching Gmail; the watchdog posts a grey *paused* card instead of shouting.

It **fails open** — a missing or unreadable vault lets everything run, because an unwanted run of a workflow that
only writes labels and drafts is cheaper than a silent stop nobody notices for a day. That reasoning is per-workflow
and does not transfer to anything that deletes or sends. See [`docs/control_plane.md`](docs/control_plane.md).

`VAULT_PATH` in `.env` points at the vault on this machine. The compose file never knows where it is — which is what
lets the same file deploy elsewhere unchanged.

---

<h2 align="center">🧠 Concepts Worth Knowing</h2>

<div align="center">

|                                                |                                                                                                              |
|------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **Containers are disposable, volumes are not** | Deleting the container loses nothing. The SQLite database, settings, and encryption key live in `n8n_data`.  |
| **Container paths, not Mac paths**             | A note at `zeltrax_hub/dashboard/status.md` is `/vault/dashboard/status.md` to a node. Same file, two names. |
| **`host.docker.internal`, not `localhost`**    | Inside the container, `localhost` is the container. That name is how n8n reaches Ollama on the host.         |
| **`up -d`, not `restart`**                     | `restart` reuses the old container with its old config. `up -d` detects the change and recreates.            |
| **Ollama stays on the host**                   | Docker Desktop on macOS can't reach the Metal GPU, so a containerised Ollama would be CPU-only.              |
| **Green is not the same as correct**           | A node set to *Continue* reports success while dropping every item. Check the funnel — item counts falling to zero mid-workflow — not the status. |
| **Silence is the failure nobody notices**      | Every check reports even when all is well, so an absent card means the checker stopped, not that nothing is wrong. |

</div>

---

<h2 align="center">🛠️ Everyday Commands</h2>

```bash
docker compose ps                   # status — the ports column shows the bind address
docker compose logs --tail=50 n8n   # best first move when something breaks
docker compose exec n8n sh          # shell inside the container
docker compose config               # resolved config (⚠️ prints your encryption key)
docker compose down                 # stop; data is safe

# upgrade: bump N8N_VERSION in .env, then
docker compose pull && docker compose up -d
```

The pieces outside Docker have their own:

```bash
hermes cron list              # scheduled jobs, last run, last recorded failure
hermes cron run <job_id>      # fire the watchdog now instead of waiting for 07:00
hermes gateway restart        # after any config.yaml or .env change — a restart is required

# scripts live in two places; Hermes only runs scripts from its own directory
cp scripts/*.py ~/.hermes/scripts/
```

---

<h2 align="center">🗺️ Roadmap</h2>

<div align="center">

### 🧪 Phase 1 — Local

*Learn the mechanics where mistakes are free.*

| Stage                                                      | Status |
|------------------------------------------------------------|--------|
| Docker Compose setup, pinned version, external volume      | ✅     |
| n8n data model — items, expressions, cross-node references | ✅     |
| AI agent with tools and memory                             | ✅     |
| Local models via Ollama                                    | ✅     |
| Obsidian vault mounted with enforced boundaries            | ✅     |
| Workflow exports committed to git                          | ✅     |
| Automated workflow export                                  | 🔜     |
| A first real capability, replacing the test switches       | ✅     |
| Long-term memory — save, recall, survives a restart        | ✅     |
| Gmail auto-labelling on a schedule                         | ✅     |
| Classification that reports how sure it is, and holds back | ✅     |
| Draft replies written locally, never sent                  | ✅     |
| One shared failure alert every workflow points at          | ✅     |
| A watchdog that keeps working when Docker does not         | ✅     |
| A chat surface — Slack, local model, no public URL         | ✅     |
| Control plane — toggles in Obsidian gate real work         | ✅     |
| Vault as its own private git repo                          | ✅     |
| Funnel check — catch a green run that labelled nothing     | 🔜     |
| Mail digest into Obsidian, and a mail agent                | 🔜     |
| Keeping the host awake, so a sleeping Mac stops costing a night | 🔜 |

### 🚀 Phase 2 — Production

*Oracle Cloud Free Tier, managed by Coolify.*

| Stage                                                     | Status |
|-----------------------------------------------------------|--------|
| Oracle Cloud instance provisioned                         | ⏳     |
| Coolify deploying this repo from git                      | ⏳     |
| Postgres instead of SQLite                                | ⏳     |
| Agent memory that survives restarts                       | ⏳     |
| Reverse proxy and real SSL                                | ⏳     |
| Webhooks — no tunnel needed once there's a public address | ⏳     |
| Vault reaching the server (git pull, or stays local)      | ⏳     |

</div>

The split is deliberate. Everything in Phase 1 runs on one machine with no public address, so mistakes cost nothing.
Phase 2 adds the things that only make sense once something is reachable from the internet and expected to stay up.

---

<div align="center">
  <sub>Built while learning, one verified step at a time.</sub>
  <br/><br/>
  <img src="https://capsule-render.vercel.app/api?type=waving&color=1F1B2E&height=110&section=footer" width="100%"/>
</div>
