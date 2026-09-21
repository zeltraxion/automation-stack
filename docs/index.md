# Docs

Detailed documentation for this repository. Start here. ⟦REVIEW⟧

## Where things are

| File                                     | What it covers                                                         |
|------------------------------------------|------------------------------------------------------------------------|
| [`../README.md`](../README.md)           | Setup, commands, architecture, roadmap                                 |
| [`../PLAN.md`](../PLAN.md)               | Researched architecture decisions, measured constraints, phase order   |
| [`../CLAUDE.md`](../CLAUDE.md)           | Operating guidance for Claude Code in this repo                        |
| [`tars_agent.md`](tars_agent.md)         | The production assistant — model settings, system message, tool design |
| [`memory_write.md`](memory_write.md)     | The memory-writing workflow — flow diagrams, design, security controls |
| [`test_workflows.md`](test_workflows.md) | The two learning workflows, and measured agent-configuration results   |

⟦REVIEW⟧

## Reading order

New to the repository: `README.md` for how to run it, then `PLAN.md` for why it is shaped this way. ⟦REVIEW⟧

Working on the assistant itself: `tars_agent.md`. Building or changing a workflow: `test_workflows.md` for the measured agent-configuration findings, then
`memory_write.md` as the worked example. ⟦REVIEW⟧

## Conventions

Workflows live in `n8n_workflows/` as exported JSON. Each production workflow gets a page here named after it. ⟦REVIEW
src: repository directory listing, read 2026-09-20⟧

Test and learning workflows are documented together in `test_workflows.md` rather than one page each. ⟦REVIEW src:
docs/test_workflows.md, covers both test_nodes_and_obsidian and test_chat_LLM⟧
