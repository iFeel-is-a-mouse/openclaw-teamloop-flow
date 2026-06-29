# Loop Engineering: Designing Systems That Prompt AI Agents

> Source: https://lushbinary.com/blog/loop-engineering-ai-coding-agents-guide

For about two years, the way you got value out of a coding agent was simple: write a good prompt, share enough context, read what came back, and type the next thing. You held the tool the entire time, one turn after another. In June 2026 that posture started to change. "Loop engineering" is the name now attached to the shift, and it captures a real change in where the leverage lives: you stop being the person who prompts the agent and start being the person who designs the system that prompts it.

The phrase was popularized by Google engineer Addy Osmani, echoing Peter Steinberger's line that "you should be designing loops that prompt your agents" and Anthropic Claude Code lead Boris Cherny's comment that his job is now to write loops rather than to prompt the model directly. It is still early, the token economics can swing wildly, and verification is harder than ever. But the building blocks now ship inside the products you already use, so the pattern is worth understanding whether or not you adopt it fully.

This guide breaks down what loop engineering actually is, how it differs from prompt and context engineering, the Ralph technique that proved the idea before it had a name, the five building blocks (plus memory) that make a loop hold together, how Claude Code and OpenAI Codex implement each piece, how to write a stop condition the loop cannot fake, a maturity ladder for adopting loops safely, what one realistic loop looks like end to end, and the failure modes that get sharper, not easier, as the loop improves. If you write code with agents, this is the next layer of the craft.

## What This Guide Covers

- What Loop Engineering Actually Means
- From Prompt Engineering to Loop Engineering
- The Ralph Technique: Where the Loop Started
- The Five Building Blocks (Plus Memory)
- Automations: The Heartbeat of a Loop
- Worktrees: Parallel Agents Without Collisions
- Skills & Memory: Stop Re-Explaining Your Project
- Sub-Agents: Separate the Maker From the Checker
- What One Loop Looks Like, End to End
- The Risks Loop Engineering Does Not Solve
- FAQ

## What Loop Engineering Actually Means

Loop engineering is replacing yourself as the person who prompts the agent, and designing the system that does it instead. A loop here is a recursive goal: you define a purpose once, and the agent iterates until the work is actually complete. Instead of you typing the next instruction after every response, a small system finds the work, hands it out, checks the result, writes down what is done, and decides the next thing to do. You let that system poke the agent instead of poking it yourself.

The mental model that helps most: a coding agent already runs an inner loop on every turn. It reasons about what to do, takes an action (calls a tool, edits a file, runs a test), observes the result, and loops back to reason again against the new state. That perceive, reason, act, observe cycle is the agentic loop. Loop engineering sits one floor above it. You are no longer steering each turn by hand. You are building an outer loop that runs on a schedule, spawns helpers, feeds itself work, and keeps going across many of those inner cycles without you in the seat for each one.

**💡 The one-sentence definition:** Loop engineering is building a system that prompts your agent on a schedule and against a goal, instead of typing each prompt yourself. The leverage moves from the quality of a single prompt to the design of the system that generates and verifies prompts.

What surprised early adopters is that this is no longer a build-it-yourself effort. A year ago, a loop meant a pile of bash scripts you maintained forever and that only you understood. As of mid 2026, the pieces ship inside the products. Peter Steinberger's checklist of what a loop needs maps almost exactly onto the OpenAI Codex app, and nearly the same list onto Anthropic's Claude Code. Once you notice the shape is identical across tools, you stop arguing about which agent is best and start designing a loop that works no matter which one you happen to be sitting in.

Loop engineering is the agentic sibling of two ideas you have probably already met. If you are coming from vibe coding, loop engineering is what happens when the vibes need to run unattended and survive your laptop closing. If you have read about multi-agent development with Claude Code agent teams, loop engineering is the discipline of wiring those agents into a self-running cycle.

## From Prompt Engineering to Loop Engineering

It helps to see loop engineering as the third layer in a stack that has been building for a few years. Each layer wraps the one inside it, and each one moves the leverage point a little further away from the raw model call.

| Layer | What you optimize | Unit of work |
|-------|-------------------|-------------|
| Prompt engineering | How you phrase a single instruction | One turn you type by hand |
| Context engineering | What else goes in the window: docs, history, tool definitions | The conditions around one answer |
| Loop engineering | The system that decides what to prompt and when, and whether the result is acceptable | A self-running cycle across many turns |

Prompt engineering never goes away. A loop is built out of prompts, and a sloppy prompt inside a loop just produces sloppy work faster. Context engineering does not go away either: the loop still has to put the right files, history, and tool definitions in front of the model on each turn. What loop engineering adds is the autonomous control structure around all of that. The harness runs the single agent. The loop runs the harness on a timer, spawns helpers, and feeds itself.

**⚠️ The leverage moved, the work did not get easier:** Boris Cherny's point is not that coding got easier. It is that the highest-value thing you can do shifted from writing prompts to designing loops. A well-designed loop multiplies a good engineer. A badly designed loop multiplies a bad decision just as fast, with less of you watching.

## The Ralph Technique: Where the Loop Started

Before anyone called it loop engineering, there was Ralph. In early 2026 Geoffrey Huntley described running a coding agent inside a plain while loop: feed the agent the same prompt against a written spec, let it pick one task and implement it, then start a fresh instance and feed the identical prompt again. Repeat until the work is done. He named it after Ralph Wiggum, the Simpsons character, because the technique is, in his words, deterministically simple in an unpredictable world. It looks too dumb to work, and it works.

The non-obvious insight is the context reset. A long agent session degrades as the window fills with old reasoning, dead ends, and stale file contents. Ralph sidesteps that entirely: every iteration is a new agent with a clean context that reads the current state of the repo and the task list from disk, does exactly one unit of work, commits it, and exits. The intelligence does not live in a heroic single run. It lives in clear, granular specifications and verifiable outcomes, applied over and over against an external memory the model cannot pollute.

In its rawest form a Ralph loop is a few lines of shell:

```bash
# The original Ralph loop: same prompt, fresh context, until done
while ! grep -q "ALL TASKS DONE" STATUS.md; do
  claude -p "Read PLAN.md and STATUS.md. Pick the next unchecked
  task, implement it, run the tests, commit on success,
  and update STATUS.md. Then stop." \
  --dangerously-skip-permissions
done
```

**💡 Loop engineering is Ralph, productized:** Ralph is the proof of concept that you do not need a clever harness, just persistence, an external state file, and verifiable stopping criteria. Loop engineering is what happens when those exact ideas move inside the tools: the while loop becomes a scheduled automation, the context reset becomes a worktree and a sub-agent, and the "ALL TASKS DONE" check becomes a /goal condition graded by a separate model. Same shape, fewer sharp edges.

## The Five Building Blocks (Plus Memory)

A working loop needs five things, and then one place to remember state:

1. **Automations** that fire on a schedule and do discovery and triage by themselves.
2. **Worktrees** so two agents working in parallel do not step on each other's files.
3. **Skills** to write down the project knowledge the agent would otherwise guess at every session.
4. **Plugins and connectors** to plug the agent into the tools you already use.
5. **Sub-agents** so one of them has the idea and a different one checks it.

The sixth piece is **memory**: a markdown file, a Linear or GitHub board, anything that lives outside a single conversation and holds what is done and what is next. The model forgets everything between runs, so the state has to live on disk, not in the context window. The agent forgets. The repo does not.

Both Claude Code and OpenAI Codex ship all five blocks plus durable memory, with different command names but the same shape.

## Automations: The Heartbeat of a Loop

Automations are what make a loop an actual loop and not just one run you did once. They are the heartbeat: a recurring trigger that surfaces work without you asking.

### In OpenAI Codex
The Codex app has an Automations tab where you pick the project, the prompt to run, the cadence, and whether it runs on your local checkout or a background worktree. Runs that find something land in a Triage inbox; runs that find nothing archive themselves.

### In Claude Code
Claude Code reaches the same place through scheduling and hooks. The `/loop` command schedules a recurring prompt on an interval, hooks fire shell commands at points in the agent lifecycle, and you can push the whole thing to GitHub Actions so it keeps running after you close the laptop.

```bash
# Claude Code: run a recurring triage prompt every weekday at 9am
/loop "Read yesterday's CI failures and open issues, write findings
 to TODO.md, and draft fixes for anything labeled quick-win"
 --schedule "0 9 * * 1-5"

# Claude Code: run until a verifiable stopping condition holds
/goal "All tests in test/auth pass and lint is clean"

# OpenAI Codex: persisted long-running objective (CLI 0.128.0+)
codex /goal "Migrate the billing module to the new pricing API,
 keep all existing tests green"
```

### Write the stop condition like a contract, not a wish

| Contract field | Weak version | Verifiable version |
|---------------|-------------|-------------------|
| End state | "Improve test coverage" | "Coverage for src/billing is at or above 90%" |
| Evidence | "It looks done" | "npm test exits 0 and the coverage report confirms the number" |
| Constraints | (unstated) | "Do not touch public APIs or delete existing tests" |
| Budget | (unbounded) | "Stop after 25 turns or $5, whichever comes first" |

## Worktrees: Parallel Agents Without Collisions

The moment you run more than one agent, files start colliding. A git worktree solves it: a separate working directory on its own branch that shares the same repo history.

```bash
# Spin up two isolated checkouts from the same repo
git worktree add ../app-fix-login -b fix/login-flake
git worktree add ../app-bump-deps -b chore/bump-deps

# When each branch is green, merge and clean up
git merge fix/login-flake
git worktree remove ../app-fix-login
```

## Skills & Memory: Stop Re-Explaining Your Project

A skill is how you stop re-explaining the same project context every session. Both tools use the same format: a folder with a SKILL.md file holding instructions and metadata, plus optional scripts, references, and assets.

```yaml
# .claude/skills/triage-ci/SKILL.md
---
name: triage-ci
description: Read overnight CI failures and open issues, then write
 a prioritized findings list to TODO.md. Read-only on code.
---
1. Run `gh run list --status failure --limit 20` and read the logs.
2. Cross-reference open issues with `gh issue list --label bug`.
3. Group failures by root cause, not by individual test.
4. Append findings to TODO.md under "## Open", newest first.
5. Label anything fixable in one file as "quick-win".
6. Do NOT edit application code. This skill only triages.
```

## Sub-Agents: Separate the Maker From the Checker

Sub-agents let you separate concerns: one agent writes the code, a different one reviews it. The verifier model checks whether the work meets the goal condition without the bias of having written it.

## What One Loop Looks Like, End to End

A realistic loop:
1. Automation fires on schedule (e.g., every morning at 9am)
2. Triage agent reads CI failures and open issues, writes findings to TODO.md
3. Implementation agent picks the top task, creates a worktree, implements the fix
4. Verification agent runs tests and lint, checks against the goal condition
5. On success: commits, creates PR, updates status
6. On failure: logs the error, moves to next task, or escalates
7. Loop repeats until TODO.md is empty or budget is exhausted

## The Risks Loop Engineering Does Not Solve

- **Multiplied mistakes**: A bad decision in the loop design gets applied at scale
- **Token costs**: Unattended agents can burn through budget quickly
- **Verification gaps**: An agent that wrote the code grading its own work is unreliable
- **Review bottleneck**: Your bandwidth to review merged work is still the ceiling
- **Drift**: Without clear constraints, agents can wander off-spec over many iterations

## Why Loop Engineering Matters

Loop engineering is the next layer of the craft after vibe coding. It's what happens when the vibes need to run unattended and survive your laptop closing. The pieces now ship inside the products you already use, so the pattern is worth understanding whether or not you adopt it fully today.
