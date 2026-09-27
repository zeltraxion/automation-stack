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
| [`mail_categorizer.md`](mail_categorizer.md) | The classification sub-workflow — categories, label IDs, accuracy |
| [`test_workflows.md`](test_workflows.md) | The two learning workflows, and measured agent-configuration results   |

## Reading order

New to the repository: `README.md` for how to run it, then `PLAN.md` for why it is shaped this way.

Working on an agent: `memory_agent.md`. Building or changing a workflow: `test_workflows.md` for the measured agent-configuration findings, then
`memory_write.md` and `memory_read.md` as the two worked examples — one writing, one reading.

The mail pages are the worked examples that are not about memory. `mail_label_draft.md` covers the scheduled workflow
end to end — the three gates before a model runs, and the safety rules that govern anything which could hide mail.
`mail_categorizer.md` covers the sub-workflow it calls.

## Conventions

Workflows live in `n8n_workflows/` as exported JSON. Each production workflow gets a page here named after it.

Test and learning workflows are documented together in `test_workflows.md` rather than one page each.
