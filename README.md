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
      <p>Infrastructure-as-config for a self-hosted <strong>n8n</strong> instance. No application code — just the compose file, the environment template, and room for exported workflows and custom nodes.</p>
      <ul>
        <li>🐳 <strong>Runtime</strong>: Docker Compose, single service</li>
        <li>💾 <strong>Storage</strong>: SQLite on a named volume</li>
        <li>🧠 <strong>Models</strong>: local via Ollama, or a cloud API</li>
        <li>🔒 <strong>Exposure</strong>: loopback only — nothing on the LAN</li>
        <li>☁️ <strong>Going to</strong>: Oracle Cloud Free Tier, via Coolify</li>
      </ul>
    </td>
    <td width="42%" valign="top" align="center" style="border: none;">
      <h3>⚙️ Stack</h3>
      <img src="https://skillicons.dev/icons?i=docker,sqlite,nodejs,ts,githubactions,bash&theme=dark" alt="Stack" />
      <br/><br/>
      <img src="https://img.shields.io/badge/LangChain-1F1B2E?style=flat-square&logo=langchain&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Obsidian-1F1B2E?style=flat-square&logo=obsidian&logoColor=EA4B71" />
      <img src="https://img.shields.io/badge/Oracle_Cloud-1F1B2E?style=flat-square&logo=oracle&logoColor=EA4B71" />
    </td>
  </tr>
</table>

---

<h2 align="center">🚀 Quick Start</h2>

**Prerequisites:** Docker Desktop running, and `docker compose version` reporting v2 or newer. ⟦REVIEW⟧

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

Then open **http://localhost:5678** ⟦REVIEW src: docker-compose.yml, ports "127.0.0.1:5678:5678"⟧

> **Set the encryption key before the first boot.** n8n generates its own if you don't, writes it into the volume, and
> then refuses to start if you later supply a different one. ⟦REVIEW src: n8n docs, "Set a custom encryption key"⟧

---

<h2 align="center">🗂️ Layout</h2>

```
n8n-project/
├── docker-compose.yml     # infrastructure definition — the shape
├── env.example            # variable template — no values
├── .env                   # your values (gitignored)
├── docs/                  # detailed docs — start at docs/index.md
├── n8n_workflows/         # exported workflow JSON
├── custom-nodes/          # future TypeScript nodes
└── .github/workflows/     # future CI
```

The split between `docker-compose.yml` and `.env` is deliberate: the compose file describes *what the infrastructure
looks like*, and `.env` holds *what differs per machine*. Coolify reads the former from git and injects the latter from
its own UI — so keeping values out of the compose file is what makes the migration a non-event. ⟦REVIEW⟧

---

<h2 align="center">🔐 The Vault Mounts</h2>

An Obsidian vault is mounted in as the control surface — you command from `control_plane`, n8n reports into `dashboard`,
you brief the agent in `memory/briefing`, the agent remembers into `memory/learned`,
and each folder has exactly one writer. ⟦REVIEW src: docker-compose.yml, volumes block⟧

| Folder             | You                            | n8n           | Mounted as    |
|--------------------|--------------------------------|---------------|---------------|
| `notes/`           | read + write                   | **no access** | *not mounted* |
| `control_plane/`   | read + write — the switches    | read only     | `:ro`         |
| `memory/briefing/` | read + write — facts you give  | read only     | `:ro`         |
| `dashboard/`       | read only — generated output   | read + write  | writable      |
| `memory/learned/`  | read + delete — agent's memory | read + write  | writable      |
| `files/`           | read + write                   | read + write  | writable      |

Three separate mechanisms are at work, and each protects something different — they do not stack on the same folder:
⟦REVIEW⟧

| Mechanism                                | What it actually protects                                                                                                                                                                                        |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Which folders are mounted**            | `notes/` is never mounted, so it does not exist inside the container. This is the **only** thing protecting your notes. ⟦REVIEW src: docker-compose.yml, volumes block — no entry for notes⟧                     |
| **The `:ro` mount option**               | `control_plane/` is mounted read-only, so writes to it fail at the filesystem level, for every process in the container. ⟦REVIEW src: docker-compose.yml, `${VAULT_PATH}/control_plane:/vault/control_plane:ro`⟧ |
| **`N8N_RESTRICT_FILE_ACCESS_TO=/vault`** | Workflow file operations are limited to paths under `/vault` — not `/tmp`, not n8n's own config directory. ⟦REVIEW src: docker-compose.yml, environment block⟧                                                   |

Two consequences worth being precise about. The allowlist grants `/vault` as a whole, so on its own it would happily
permit writes to `control_plane/` — `:ro` is what prevents that. And neither the allowlist nor `:ro` has any bearing on
`notes/`; that folder is safe purely because it was never handed to the container. Mount it and it becomes reachable,
allowlist and all. ⟦REVIEW⟧

`VAULT_PATH` in `.env` points at the vault on this machine. The compose file never knows where it is — which is what
lets the same file deploy elsewhere unchanged. ⟦REVIEW⟧

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

</div>

⟦REVIEW⟧

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
| Control plane — toggles in Obsidian gate real work         | ✅     |
| Workflow exports committed to git                          | ✅     |
| Automated workflow export                                  | 🔜     |
| A first real capability, replacing the test switches       | 🔜     |
| Vault as its own private git repo                          | 🔜     |

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
⟦REVIEW⟧

---

<div align="center">
  <sub>Built while learning, one verified step at a time.</sub>
  <br/><br/>
  <img src="https://capsule-render.vercel.app/api?type=waving&color=1F1B2E&height=110&section=footer" width="100%"/>
</div>
