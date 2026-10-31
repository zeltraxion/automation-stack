# error_handler

One workflow that fires whenever any other workflow fails, posts a red card to `#tars_alerts`, and pushes the same
failure to Hermes. It is set as the Error Workflow on every production workflow in this repository.

Three nodes. It is the smallest workflow here and the one with the widest reach.

## Why it exists

Before this, a failed execution left no trace outside n8n's own execution list. A scheduled workflow that died at
03:00 stayed dead until somebody opened the UI and noticed. The failure that prompted it:

```
Problem in node 'history_lookup'
The connection to the server was closed unexpectedly, perhaps it is offline.
```

Gmail dropped a connection, the run died at the first node, and nothing was labelled that cycle. Nothing said so.

## Flow

```mermaid
flowchart LR
    T["<b>Error Trigger</b><br/>no parameters"]
    S["<b>alert_logs</b><br/>Slack · #tars_alerts · red"]
    H["<b>push_to_hermes</b><br/>POST · 127.0.0.1:8644<br/>continueRegularOutput"]
    T --> S --> H
```

The Slack card comes first on purpose. It is the alert that must never be lost, and a node that has already run
cannot be stopped by one that fails after it.

## The Error Trigger

It takes no configuration. n8n calls the nominated Error Workflow after an execution fails, passing the failure
details as the trigger's output:

```json
{
  "execution": {
    "id": "231",
    "url": "http://localhost:5678/execution/workflow/1/231",
    "retryOf": "34",
    "error": { "message": "…", "stack": "…" },
    "lastNodeExecuted": "history_lookup",
    "mode": "trigger"
  },
  "workflow": { "id": "tCzqvauu1uCOzt8O", "name": "mail_label_draft" }
}
```

Executing the workflow by hand injects n8n's own sample values — `Example Workflow`, `Node With Error`, execution
`231`. That is how the card was built and tested without breaking anything real.

## The card

Four facts, chosen because they are what you need before deciding whether to care:

| Field | Source | Why |
|---|---|---|
| Workflow | `workflow.name` | which one died |
| Failed at | `execution.lastNodeExecuted` | the node that threw |
| Error | `execution.error.message` | n8n's message, verbatim |
| Link | `execution.url` | straight to that execution, red node visible |

Every expression is written defensively — `$json.execution?.error?.message ?? 'no message'`. An unexpected payload
shape produces a card saying `unknown`, not a second failure inside the error handler.

The error text is wrapped in `JSON.stringify(...).slice(1,-1)` so quotes inside it cannot break the Block Kit JSON.
Same technique as the memory workflows.

## The push to Hermes

`push_to_hermes` POSTs the same four facts to a Hermes inbound webhook route:

```
POST http://host.docker.internal:8644/webhooks/n8n-failure
```

Hermes renders them into a Slack DM. No agent runs and no model is loaded — the route is `deliver_only`, so the
template *is* the message. Zero tokens, sub-second.

Why a second alert at all, when the card already exists: this is the path that will later wake the mail watchdog
instead of writing text. Swapping `deliver_only: true` for `cron_job: mail-watchdog` on the route changes what the
POST does without touching this workflow. The message is the proof the pipe works; the pipe is the point.

### Two settings that are not optional

**`onError: continueRegularOutput`.** Hermes being down must not turn a reported failure into a failed error handler.
With it, a dead gateway costs the DM and nothing else — the card has already gone.

**`$('Error Trigger').item.json`, never `$json`.** This node runs *after* the Slack node, so `$json` is the Slack
API response — `{ok, channel, ts, …}` — not the failure payload. Written with `$json` every field resolves to its
`??` fallback and the DM arrives saying `unknown` / `no message`, with a 200 from Hermes and a green execution.
Silent success, the worst shape a bug can take.

The body is built with `JSON.stringify` rather than string interpolation, because error messages carry quotes and
newlines that break a hand-written JSON template.

### Where the route lives

`~/.hermes/config.yaml`, under `platforms.webhook.routes.n8n-failure`. Not in this repository — it is machine
configuration, like the Ollama endpoint. The adapter ships with Hermes; the route is data telling it what to do with
one path.

The gateway binds to `127.0.0.1` and the route uses `INSECURE_NO_AUTH`, which Hermes permits only on a loopback
bind and refuses to start otherwise. Docker still reaches it through `host.docker.internal`, the same way n8n
reaches Ollama on `127.0.0.1:11434`. When the gateway moves to a host where loopback no longer protects it, that
route needs a real HMAC secret — which in turn needs `NODE_FUNCTION_ALLOW_BUILTIN=crypto` on this container.

## Which workflows point at it

Set under **Settings → Error Workflow** on each:

```
mail_label_draft      memory_write
mail_categorizer      memory_read
                      memory_agent
```

`error_handler` deliberately does **not** point at itself. An error handler that errors would call itself, and that
loop has no exit.

## What it does and does not catch

**Catches:** anything that fails an execution and is not handled inside the workflow — a Code node throwing, Ollama
unreachable, a sub-workflow erroring, an expression referencing a node that did not run.

**Does not catch:** a node set to `Continue (using error output)`. That node routes its failure down a second wire
instead of failing the execution, so from n8n's point of view nothing went wrong. Those failures are reported by the
workflow's own Slack card instead.

In `mail_label_draft` five nodes work that way — `history_lookup`, `fetch_mail`, `mail_categorizer`, `apply_labels`
and `create_draft` — and their failures land in `#tars_alerts` via that workflow's `error_logs` node, not via this
one.

**Retries run first.** The Gmail nodes retry three times at two-second intervals, so a card means three consecutive
failures, not one bad response. The connection error above would most likely never have produced one.

## One duplicate worth expecting

If `mail_categorizer` fails internally you get two alerts: its own execution fails, so this workflow fires, **and**
the calling node in `mail_label_draft` routes to that workflow's `error_logs`.

Not a bug. One card names the sub-workflow, the other names the caller, and knowing which layer failed is worth a
duplicate.

## What this does not do

| Gap | Note |
|---|---|
| No severity levels | Every failure is red. A retry-exhausted Gmail blip and a broken credential look identical |
| No deduplication | A workflow failing on every run posts on every run |
| No recovery | It reports. Anything that should keep running past a failure needs a per-node error output instead |
| Two alerts per failure | The red card and the Hermes DM carry the same four facts. Deliberate while the push path is being proven; it stops once the route fires a cron job instead of delivering text |
