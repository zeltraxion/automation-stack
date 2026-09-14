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

# 3. Run
docker compose up -d
docker compose logs -f n8n     # wait for "Editor is now accessible via..."
```

Then open **http://localhost:5678** ⟦REVIEW src: docker-compose.yml, ports "127.0.0.1:5678:5678"⟧

> **Set the encryption key before the first boot.** n8n generates its own if you don't, writes it into the volume, and then refuses to start if you later supply a different one. ⟦REVIEW src: n8n docs, "Set a custom encryption key"⟧

---

<h2 align="center">🗂️ Layout</h2>

```
n8n-project/
├── docker-compose.yml     # infrastructure definition — the shape
├── env.example            # variable template — no values
├── .env                   # your values (gitignored)
├── files/                 # bind mount → /files in the container
├── n8n_workflows/         # exported workflow JSON
├── custom-nodes/          # future TypeScript nodes
└── .github/workflows/     # future CI
```

The split between `docker-compose.yml` and `.env` is deliberate: the compose file describes *what the infrastructure looks like*, and `.env` holds *what differs per machine*. Coolify reads the former from git and injects the latter from its own UI — so keeping values out of the compose file is what makes the migration a non-event. ⟦REVIEW⟧

---

<h2 align="center">🧠 Concepts Worth Knowing</h2>

<div align="center">

| | |
|---|---|
| **Containers are disposable, volumes are not** | Deleting the container loses nothing. The SQLite database, settings, and encryption key live in `n8n_data`. |
| **Container paths, not Mac paths** | A file at `files/report.csv` is `/files/report.csv` to a node. Same file, two names. |
| **`host.docker.internal`, not `localhost`** | Inside the container, `localhost` is the container. That name is how n8n reaches Ollama on the host. |
| **`up -d`, not `restart`** | `restart` reuses the old container with its old config. `up -d` detects the change and recreates. |
| **Ollama stays on the host** | Docker Desktop on macOS can't reach the Metal GPU, so a containerised Ollama would be CPU-only. |

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

| Stage | Status |
|---|---|
| Local Docker Compose setup | ✅ |
| AI agent with tools + memory | ✅ |
| Local models via Ollama | ✅ |
| Obsidian vault integration | 🔜 |
| Workflow exports committed to git | 🔜 |
| Postgres + persistent memory | ⏳ |
| Oracle Cloud + Coolify + real SSL | ⏳ |

</div>

⟦REVIEW⟧

---

<div align="center">
  <sub>Built while learning, one verified step at a time.</sub>
  <br/><br/>
  <img src="https://capsule-render.vercel.app/api?type=waving&color=1F1B2E&height=110&section=footer" width="100%"/>
</div>
