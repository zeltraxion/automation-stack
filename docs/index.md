# Docs

Detailed documentation for this repository. Start here. ⟦REVIEW⟧

## Where things are

| File                                     | What it covers                                                         |
|------------------------------------------|------------------------------------------------------------------------|
| [`../README.md`](../README.md)           | Setup, commands, architecture, roadmap                                 |
| [`../PLAN.md`](../PLAN.md)               | Researched architecture decisions, measured constraints, phase order   |
| [`../CLAUDE.md`](../CLAUDE.md)           | Operating guidance for Claude Code in this repo                        |
| [`memory_agent.md`](memory_agent.md)     | The memory-domain chat agent — model settings, system message, tool design |
| [`memory_write.md`](memory_write.md)     | The memory-writing workflow — flow diagrams, design, security controls |
| [`memory_read.md`](memory_read.md)       | The memory-reading workflow — why it takes no parameters, measured n8n behaviour |
| [`mail_sort.md`](mail_sort.md)           | The Gmail labelling workflow — label set, classifier design, safety rules |
| [`test_workflows.md`](test_workflows.md) | The two learning workflows, and measured agent-configuration results   |

⟦REVIEW⟧

## Reading order

New to the repository: `README.md` for how to run it, then `PLAN.md` for why it is shaped this way. ⟦REVIEW⟧

Working on an agent: `memory_agent.md`. Building or changing a workflow: `test_workflows.md` for the measured agent-configuration findings, then
`memory_write.md` and `memory_read.md` as the two worked examples — one writing, one reading. ⟦REVIEW⟧

`mail_sort.md` is the third worked example and the only one that is not about memory — read it for the classifier
design, the two-axis label set, and the safety rules that govern anything which hides mail. ⟦REVIEW⟧

## Conventions

Workflows live in `n8n_workflows/` as exported JSON. Each production workflow gets a page here named after it. ⟦REVIEW
src: repository directory listing, read 2026-09-20⟧

Test and learning workflows are documented together in `test_workflows.md` rather than one page each. ⟦REVIEW src:
docs/test_workflows.md, covers both test_nodes_and_obsidian and test_chat_LLM⟧
