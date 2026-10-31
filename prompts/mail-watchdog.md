The mail_label_draft watchdog found a problem. Your context block holds what it saw.

Work out what broke. Check in this order, stopping once you find the cause:

1. The execution in n8n — GET http://localhost:5678/api/v1/executions/<id>
   header: X-N8N-API-KEY: $N8N_API_KEY
2. Is the container up — `docker compose ps` in ~/Documents/playground/n8n-project
3. Is Ollama reachable — `curl -s http://127.0.0.1:11434/api/tags`
4. Has the Gmail credential expired — it dies every 7 days until the Google app
   is published.

Two facts that are NOT faults, and must never be reported as one:

- nimble:9b and qwen3:8b are absent from `ollama ps` by design. They carry
  keep_alive 30s and are evicted between runs, because 16 GB cannot hold both
  resident. Only a model missing from `ollama list`, or /api/tags failing,
  is a fault.
- A stale workflow whose last run SUCCEEDED did not fail — it never started.
  The usual cause is the Mac asleep at 21:00, which produces no execution and
  no error anywhere.

Reply with exactly these four lines, in this order, nothing before or after.
Plain text. No headings, no tables, no bullet lists, no escaped newlines.

**What broke** — one sentence
**Layer** — one of: n8n, Ollama, Gmail, schedule, unknown
**Evidence** — what you ran and what it returned
**Next action** — one thing, or "none" if nothing is needed

The two examples below show the SHAPE only. Their workflow names, execution
numbers, ages and error strings are invented. Never reuse them. Every value in
your Evidence line must come from a command you actually ran this run.

Worked example, a stale workflow whose last run was fine:

**What broke** — example_workflow did not start last night; nothing failed.
**Layer** — schedule
**Evidence** — execution 10001 is the latest and its status is success, 31.0h old. No execution exists for the 21:00 slot. docker compose ps shows n8n Up 9 days; /api/tags lists nimble:9b and qwen3:8b.
**Next action** — none until the Mac stops sleeping at 21:00; the run cannot fire with the lid closed.

Second worked example, a real failure:

**What broke** — the example_node step could not reach Ollama.
**Layer** — Ollama
**Evidence** — execution 10002 failed with "connect EXAMPLEERROR 127.0.0.1:99999". curl /api/tags returned nothing.
**Next action** — start Ollama, then re-run mail_label_draft.

Say "unknown" in the Layer line rather than guessing. Do not report anything as
broken without evidence in the Evidence line.

Change nothing. No restarts, no re-runs, no config edits. This run diagnoses only.
