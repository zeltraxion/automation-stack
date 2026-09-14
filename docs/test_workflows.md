---
generated_by: claude
human_requestor: swapnil.bhattacharya@clearroute.io
date: 2026-09-14
review_status: unreviewed
reviewed_by:
---

# Test Workflows

Learning scaffolds, not production automation. The switches here control nothing real — `backup_enabled` gates a workflow that only writes a timestamp, because there is no backup. They are kept as working references for patterns that will be reused in real workflows. ⟦REVIEW⟧

Exports live in [`n8n_workflows/`](../n8n_workflows). ⟦REVIEW src: repository directory listing⟧

---

## `test_nodes_and_obsidian`

Two independent sections on one canvas, grouped with sticky notes. Each has its own trigger, and n8n only runs the branch belonging to the trigger that fired. ⟦REVIEW⟧

### Section A — Basic node tests

```
Manual Trigger → Code → Edit Fields → Aggregate → Edit Fields1
```

Demonstrates n8n's data model: ⟦REVIEW⟧

| Concept | Where it shows up |
|---|---|
| Every node receives and returns an **array of items** | the Code node returns three `{ json: {...} }` objects |
| Most nodes **run once per item** | Edit Fields executes three times; `$json` means *the current item* |
| Expressions are real JavaScript | `{{ $json.years >= 5 ? 'senior' : 'junior' }}` |
| **Include Other Input Fields** decides whether upstream fields survive | off by default — the usual cause of fields "disappearing" mid-workflow |
| Some nodes **collapse many items into one** | Aggregate turns 3 items into 1, nesting them under a configurable field name |
| Reaching a node that isn't feeding you | `{{ $('Edit Fields').first().json.name }}` |

⟦REVIEW⟧

After Aggregate the shape changes, so `$json.name` becomes undefined and the data lives at `$json.<output field>[0].name`. The aggregated value is a plain JavaScript array, so `map`, `filter` and `join` work on it directly. ⟦REVIEW⟧

### Section B — Obsidian file-based toggle

The control-plane pattern end to end. ⟦REVIEW⟧

```
Schedule Trigger
  → Read/Write Files from Disk   (read /vault/control_plane/services.md)
  → Extract from File            (binary → text)
  → Get Contents                 (Code: parse YAML frontmatter)
  ├→ Build the Markdown → Convert to File → Write /vault/dashboard/status.md
  └→ If (backup_enabled is true)
       └ true → Code → Convert to File → Write /vault/dashboard/backup_last_run.md
       └ false → nothing
```

⟦REVIEW⟧

**The switch lives in YAML frontmatter**, not in a markdown table. Obsidian renders boolean frontmatter as a real checkbox in its Properties panel, so flipping a switch is a click — while on disk it is a one-word change that n8n can read without parsing markdown. ⟦REVIEW⟧

```yaml
---
backup_enabled: true
daily_summary_enabled: false
---
```

**The frontmatter parser is hand-rolled** in the `Get Contents` node. n8n 2.x blocks Code nodes from importing external modules by default, so `js-yaml` is unavailable without widening `NODE_FUNCTION_ALLOW_EXTERNAL`. The parser handles flat `key: value` pairs only — which is a reasonable constraint, since nested YAML would be unpleasant to edit in Obsidian's Properties panel anyway. ⟦REVIEW⟧

**Two writers are never pointed at one file.** `control_plane/` is written only by a human, `dashboard/` only by n8n. That is what makes this safe to put through git later without merge conflicts. ⟦REVIEW⟧

**The IF node is the point of the exercise.** With the switch off, the nodes after the `true` branch show zero executions while the status chain still runs. A toggle in a text editor causing a node not to execute is the whole control-plane idea in one observation. ⟦REVIEW⟧

---

## `test_chat_LLM`

```
When chat message received → AI Agent
                               ├─ Ollama Chat Model (qwen3:8b)
                               ├─ Simple Memory
                               └─ Tools: Calculator, Current Date, Between Dates
```

⟦REVIEW⟧

### What an agent run actually looks like

The model is called **once per decision**, not once per question. A single tool use means two model calls: one to choose the tool, one to turn the result into a sentence. Cost and latency scale with the number of tool calls. ⟦REVIEW src: execution logs observed in this workflow⟧

Memory is not clever — the entire prior conversation is re-sent with every message. Token counts grow accordingly, and Simple Memory holds it in RAM, so a container restart clears it. ⟦REVIEW⟧

### Measured results

Same question, same tools, varying configuration: ⟦REVIEW src: workflow executions run 2026-09-13 and 2026-09-14⟧

| Configuration | Result | Time |
|---|---|---|
| llama3.1:8b, defaults, tool with no AI-fillable params | wrong | ~19s |
| Gemini, same broken tool | correct | ~17s |
| qwen3:8b, thinking **on**, no system prompt | correct, by luck | **4m 39s** |
| qwen3:8b, thinking **off**, no system prompt | badly wrong | 13s |
| qwen3:8b, thinking off + proper tools + system prompt | **correct** | 20s |

⟦REVIEW⟧

**The model mattered least.** Tool design and instructions mattered most. Three things moved it from unreliable to correct: ⟦REVIEW⟧

1. **Tool parameters marked AI-fillable** (the sparkle icon). Without them a tool is a fixed button the agent cannot steer, so it guesses instead.
2. **A System Message that mandates tool use** — "always use the Calculator for arithmetic, never calculate in your head".
3. **Distinct tool names.** Two tools both called "Date & Time" produced ambiguous selection; renaming to "Current Date" and "Between Dates" resolved it.

Settings that matter for local models: **Enable Thinking off**, **Sampling Temperature 0.1**. Tool calls are structured JSON, and a default chat temperature encourages the variation that breaks parsing. ⟦REVIEW⟧

---

## Gotchas hit while building these

| Symptom | Cause |
|---|---|
| `Access to the file is not allowed` | n8n's file allowlist was empty. Fixed with `N8N_RESTRICT_FILE_ACCESS_TO=/vault`. Separate from Docker's mounts. |
| File not found at `/vault/control_plane/services.md` | Obsidian appends `.md` on creation, so the file was `services.md.md` |
| Read node returns no readable text | files arrive as **binary**; `Extract from File` converts to text, `Convert to File` converts back for writing |
| A switch that looks "off" doesn't behave as false | empty YAML value is `null`, not `false` — set it explicitly |
| Config change appears to do nothing | `docker compose restart` reuses the old container; `up -d` recreates it |
| Second branch overwrote the first branch's output | one Write node means one destination path — each output file needs its own Convert + Write pair |

⟦REVIEW⟧

---

## What these do not cover

Postgres, persistent memory across restarts, webhooks, queue mode, and anything requiring a public address. Those arrive with the move to Oracle Cloud. ⟦REVIEW⟧
