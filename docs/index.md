# Docs

Detailed documentation for this repository. Start here.

## Where things are

| File                                     | What it covers                                                         |
|------------------------------------------|------------------------------------------------------------------------|
| [`../README.md`](../README.md)           | Setup, commands, architecture, roadmap                                 |
| [`../PLAN.md`](../PLAN.md)               | Researched architecture decisions, measured constraints, phase order   |
| [`../CLAUDE.md`](../CLAUDE.md)           | Operating guidance for Claude Code in this repo                        |
| [`memory_agent.md`](memory_agent.md)     | The memory-domain chat agent — model settings, system message, tool design |
| [`memory_write.md`](memory_write.md)     | The memory-writing workflow — flow diagrams, design, security controls |
| [`memory_read.md`](memory_read.md)       | The memory-reading workflow — why it takes no parameters, measured n8n behaviour |
| [`mail_label_draft.md`](mail_label_draft.md) | The Gmail labelling and drafting workflow — flow, gates, safety rules |
| [`mail_categorizer_jev.md`](mail_categorizer_jev.md) | The live classification sub-workflow — decision model, confidence floor, accuracy |
| [`mail_categorizer.md`](mail_categorizer.md) | The superseded Text Classifier version, kept as the rollback — category set, label IDs |
| [`error_handler.md`](error_handler.md)   | The shared failure alert — what it catches, and what it deliberately does not |
| [`mail_watchdog.md`](mail_watchdog.md)   | The Hermes cron watchdog — the only check that runs outside n8n, and why that matters |
| [`hermes_gateway.md`](hermes_gateway.md) | The Slack-facing agent — Socket Mode, `#tars_chat`, the persona file, the model chain |
| [`test_workflows.md`](test_workflows.md) | The two learning workflows, and measured agent-configuration results   |

## Reading order

New to the repository: `README.md` for how to run it, then `PLAN.md` for why it is shaped this way.

Working on an agent: `memory_agent.md`. Building or changing a workflow: `test_workflows.md` for the measured agent-configuration findings, then
`memory_write.md` and `memory_read.md` as the two worked examples — one writing, one reading.

The mail pages are the worked examples that are not about memory. `mail_label_draft.md` covers the scheduled workflow
end to end — the four gates before a model runs, and the safety rules that govern anything which could hide mail.
`mail_categorizer_jev.md` covers the sub-workflow it calls, and is the one to read for how a decision model differs
from a Text Classifier. `mail_categorizer.md` is its predecessor, kept as the rollback.

`error_handler.md` is worth reading before adding any workflow — it is the failure alert every workflow points at, and
knowing what it does not catch is what tells you when a node needs its own error output instead.

`mail_watchdog.md` is its counterpart and reads best straight after it. `error_handler` covers executions that start
and fail; the watchdog covers the one case it structurally cannot see — a scheduled run that never started at all.
It is also the only page here about something that runs outside n8n and Docker, which is the whole reason it works.

`hermes_gateway.md` is the other half of that outside-n8n story: the agent you actually talk to, over Slack. Read it
for why Socket Mode removed the public-URL problem that stalled the Telegram plan, how `#tars_chat` differs from the
home DM, and where the persona and model chain are configured. The two Hermes pages share a workspace and nothing
else — different transports, different credentials, which is explained in both.

## Conventions

Workflows live in `n8n_workflows/` as exported JSON. Each production workflow gets a page here named after it.

Test and learning workflows are documented together in `test_workflows.md` rather than one page each.

Not everything documented here is an n8n workflow. `mail_watchdog.md` covers a Hermes cron job whose source lives in
`scripts/`, mirrored into `~/.hermes/scripts/` because Hermes only runs scripts from its own directory. Anything that
monitors this repository but runs outside it belongs here too, named the same way.
