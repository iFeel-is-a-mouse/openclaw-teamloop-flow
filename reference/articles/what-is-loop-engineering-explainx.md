# What Is Loop Engineering? The New Paradigm Beyond Prompt Engineering

> Source: https://www.explainx.ai/blog/what-is-loop-engineering-ai-agents-2026  
> Note: This page is JS-rendered; full article content could not be extracted. Below is a summary based on available metadata and search results, including cross-references from other deep dives.

## Core Definition (from explainx.ai)

> "What system should I build so the agent finds the work, does it, verifies it, and remembers what it did — without me in the loop at all?"

## Four Generations of AI Engineering

| Generation | Focus | Era | Anchor |
|-----------|-------|-----|--------|
| **Prompt Engineering** | How you speak to the model (few-shot, CoT, role-play) | 2022-2024 | OpenAI Cookbook, PromptingGuide.ai |
| **Context Engineering** | What you show the model (history, RAG, state, memory) | 2025 | Coined by Shopify CEO Tobi Lütke; Anthropic's Effective Context Engineering |
| **Harness Engineering** | Scaffolding for a single session (tools, constraints, feedback) | Early 2026 | Karpathy-adjacent "Software 3.0" discourse |
| **Loop Engineering** | Who orchestrates whom, when, and how often | **June 2026 onwards** | Addy Osmani, Boris Cherny, Cobus Greyling |

## Common Threads Across Definitions

- The human stops being the prompter
- The agent runs itself
- Verification and state persistence are built in
- Termination and next-action decisions are automated

## Inner Loop vs Outer Loop

- **Inner Loop (classical ReAct)**: reason → act → observe → repeat, within one session. A single LLM + tool-call cycle.
- **Outer Loop (Loop Engineering)**: The system that schedules, dispatches, verifies, persists state, and decides what to do next — across many inner-loop cycles.

## Key Building Blocks

1. **Automations** - Scheduled triggers that surface work without human prompting
2. **Worktrees** - Isolated git worktrees for parallel agent execution without collisions
3. **Skills** - Durable project knowledge encoded as reusable instructions
4. **Plugins/Connectors** - Integration with existing tools and services
5. **Sub-agents** - Maker-checker pattern: one agent creates, another verifies
6. **Memory/Durable State** - External state (markdown files, issue boards) that persists across sessions

## Current Status (June 2026)

Anthropic, OpenAI, and Cognition have not yet officially adopted the term in product docs, but a coherent definition is shared across multiple primary and secondary sources. The term has become the interpretive frame for an existing feature set rather than a new product line.

Both Claude Code and OpenAI Codex already ship the building blocks:
- Claude Code: `/loop`, `/goal`, worktrees, skills, hooks, sub-agents
- OpenAI Codex: Automations tab, `/goal` (CLI 0.128.0+), worktree support

---

*Full article available at: https://www.explainx.ai/blog/what-is-loop-engineering-ai-agents-2026*
