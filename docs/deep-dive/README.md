# How oh-my-opencode Works: A Deep Dive

A learning path for engineers who want to understand multi-agent LLM orchestration, how this plugin implements it, and how to reproduce or extend these patterns in their own projects.

---

## Who This Is For

- Engineers learning to work with AI coding tools who want to go beyond "type a prompt and hope"
- Engineers who want to understand the internals well enough to extend the plugin
- Engineers who want to extract patterns for their own multi-agent systems

## Prerequisites

- Basic familiarity with TypeScript
- Experience using an LLM coding tool (Claude Code, Cursor, Copilot, etc.)
- You've at least installed oh-my-opencode and used it once

---

## The Learning Path

### Part 1: Concepts

Start here if you're new to multi-agent orchestration or want to understand *why* things are built this way.

| Doc | What You'll Learn | Time |
|-----|-------------------|------|
| [01 - Multi-Agent Orchestration Concepts](./01-concepts.md) | Why one agent isn't enough. The fundamental problems that multi-agent systems solve: context limits, cognitive drift, specialization. Core patterns like orchestrator-worker, planning-execution separation, and parallel execution. | 15 min |

### Part 2: Architecture

How the plugin is structured and how the pieces connect.

| Doc | What You'll Learn | Time |
|-----|-------------------|------|
| [02 - Plugin Architecture](./02-architecture.md) | The 5-step initialization pipeline. How agents, hooks, tools, and MCPs are created and wired together. The OpenCode plugin interface and the 8 hook handlers. | 20 min |
| [03 - The Agent System](./03-agents.md) | How agents are defined using factory functions. The three agent modes (primary, subagent, all). Model resolution with 4-step fallback chains. Tool restriction patterns. | 20 min |
| [04 - Dynamic Prompt Assembly](./04-prompt-engineering.md) | How Sisyphus's system prompt is built dynamically from agent metadata, available skills, categories, and environment context. The most transferable knowledge in this series. | 25 min |

### Part 3: Systems

The major subsystems that make orchestration work.

| Doc | What You'll Learn | Time |
|-----|-------------------|------|
| [05 - The Hook System](./05-hooks.md) | The 3-tier hook architecture (Core, Continuation, Skill). How hooks intercept and modify behavior across the agent lifecycle. Hook event types and composition patterns. | 20 min |
| [06 - Task Delegation](./06-delegation.md) | The category system that routes tasks to optimal models. How skills inject specialized knowledge and MCP tools. Background agent execution and concurrency management. | 25 min |
| [09 - Context Management](./09-context-management.md) | How context is injected (AGENTS.md, README.md, rules), truncated (tool output limits, dynamic budgets), and recovered (compaction, multi-strategy overflow recovery). The full lifecycle of what goes into the agent's context window. | 25 min |

### Part 4: Practice

Hands-on guides for extending the system and extracting patterns.

| Doc | What You'll Learn | Time |
|-----|-------------------|------|
| [07 - Extending oh-my-opencode](./07-extending.md) | Step-by-step: create a custom agent, add a skill with embedded MCP, write a hook, register a command. Real code examples for each. | 30 min |
| [08 - Patterns for Your Own Projects](./08-patterns.md) | Extractable patterns you can use anywhere: prompt engineering for orchestrators, multi-model routing, agent-as-function, background execution, intent classification. | 25 min |

---

## How to Read This

**If you want to understand the big picture**: Read 01 and 02, then skip to 08.

**If you want to extend oh-my-opencode**: Read 02, 03, and 07.

**If you want to build your own multi-agent system**: Read 01, 04, 06, and 08.

**If you want to understand context management**: Read 09 (standalone, references 05 for hook background).

**If you want to understand everything**: Read them in order. Each doc builds on the previous.

---

## Relationship to Existing Docs

This series goes deeper than the user-facing docs:

| Existing Doc | This Series Equivalent |
|---|---|
| [Overview](../guide/overview.md) | Covered in 01 (concepts) and 02 (architecture), with more depth |
| [Orchestration Guide](../guide/orchestration.md) | Covered in 03 (agents), 04 (prompts), and 06 (delegation) |
| [Features Reference](../reference/features.md) | Features are *explained* here, not just listed |
| [Configuration Reference](../reference/configuration.md) | Config is covered in context throughout, especially 07 |

The existing docs tell you *what* to configure. This series explains *why* it works that way and *how* to think about it.
