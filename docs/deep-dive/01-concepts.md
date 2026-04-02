# Multi-Agent Orchestration Concepts

Before diving into code, let's understand the problems that multi-agent orchestration solves and the patterns it uses. These concepts apply beyond oh-my-opencode to any system where you need multiple AI agents working together.

---

## The Problem with Single-Agent Systems

Most AI coding tools follow a simple model: one agent, one model, one conversation. You type a request, the model processes it, and it responds. This works fine for simple tasks, but breaks down as complexity increases.

### Problem 1: Context Window Limits

Every LLM has a finite context window (the amount of text it can "see" at once). A single agent doing everything accumulates context rapidly:

```
Turn 1:  Read 5 files to understand codebase     (~15,000 tokens)
Turn 2:  Search for patterns                       (~8,000 tokens)
Turn 3:  Read documentation                        (~10,000 tokens)
Turn 4:  Plan the implementation                   (~5,000 tokens)
Turn 5:  Start writing code...                     (~38,000 tokens used, getting crowded)
```

By the time the agent starts the actual work, it's burned through a significant portion of its context window on research. The early context gets compressed or dropped, and the agent "forgets" what it learned.

**Multi-agent solution**: Research agents operate in their own context windows. They do the exploration, distill findings into a concise summary, and pass only that summary back to the orchestrator. The orchestrator's context stays clean.

### Problem 2: Cognitive Drift

A single agent handling a complex task will drift. It starts implementing feature A, notices a bug in module B, fixes that, then realizes module C needs updating too, and eventually forgets what it was originally doing.

This isn't a model intelligence problem. It's a working memory problem. Humans do this too, which is why we use todo lists, project boards, and delegation.

**Multi-agent solution**: An orchestrator maintains the plan and delegates focused tasks. Each worker agent gets one specific job with clear boundaries. It can't drift because it only knows about its assigned task.

### Problem 3: Model Specialization

Different models have different strengths:

| Strength | Best Models (as of early 2026) |
|----------|-------------------------------|
| Following complex instructions | Claude Opus, Kimi K2.5 |
| Deep logical reasoning | GPT-5.4 |
| Visual/frontend work | Gemini 3.1 Pro |
| Fast cheap utility work | GPT-5-Nano, MiniMax |
| Code search/grep | Grok Code Fast |

A single-agent system forces you to pick one model for everything. A multi-agent system routes different types of work to the model best suited for it.

### Problem 4: Serial Execution

A single agent works sequentially. It can only do one thing at a time: read a file, then search, then read another file, then think. Even when tasks are completely independent, they happen one after another.

**Multi-agent solution**: Fire multiple agents simultaneously. While one agent researches patterns in the codebase, another looks up external documentation, and the orchestrator continues with work that doesn't depend on those results.

---

## Core Patterns

### Pattern 1: Orchestrator-Worker

The most fundamental pattern. One agent (the orchestrator) plans and coordinates. Other agents (workers) execute specific tasks.

```
         Orchestrator
        /     |      \
   Worker   Worker   Worker
  (search) (implement) (verify)
```

**Key principle**: The orchestrator should never do the work itself (except for trivial cases). Its job is to decompose, delegate, verify, and report.

**Why this matters**: The orchestrator maintains the big picture in its context window. Workers can afford to fill their context with task-specific details because they only need to see their own slice of the problem.

**Real-world analogy**: A senior engineer who writes all the code themselves is wasting their architectural understanding. A senior engineer who decomposes the problem, assigns tasks, and reviews results multiplies their impact.

### Pattern 2: Planning-Execution Separation

Planning and execution use different cognitive skills. Mixing them in one agent leads to plans that skip steps (because the agent starts executing mid-plan) or execution that constantly re-plans (because the agent keeps second-guessing itself).

```
Phase 1: Planning (read-only, no code changes)
  - Understand requirements
  - Research existing codebase
  - Identify risks and dependencies
  - Produce a detailed plan

Phase 2: Execution (follow the plan)
  - Execute each task in order
  - Verify each step
  - Accumulate learnings
  - Report completion
```

**Key principle**: The planner should not be able to write code. The executor should not be able to change the plan. This constraint prevents mode-mixing and ensures each phase is done thoroughly.

**In oh-my-opencode**: Prometheus (planner) can only write markdown files in `.sisyphus/plans/`. Atlas (executor) reads those plans and delegates to workers. Neither can do the other's job.

### Pattern 3: Tiered Cost Optimization

Not every task needs the most powerful (and expensive) model. A multi-agent system can route tasks by complexity:

```
Expensive tier:  Architecture decisions, complex debugging
                 (GPT-5.4 high reasoning, Claude Opus)

Mid tier:        Feature implementation, refactoring
                 (Claude Sonnet, GPT-5.3 Codex)

Cheap tier:      File search, typo fixes, documentation
                 (Gemini Flash, GPT-5-Nano, Grok Code Fast)
```

**Key principle**: The routing decision happens at delegation time, not at user request time. The user says "implement dark mode." The orchestrator decides which subtasks need expensive models and which don't.

### Pattern 4: Background Execution

When multiple tasks are independent, run them simultaneously:

```
Orchestrator fires:
  Agent A (background): "Find all authentication code"
  Agent B (background): "Look up JWT best practices"
  Agent C (background): "Check existing test patterns"
  
Orchestrator continues with non-dependent work...

  [Agent A completes] -> Results stored
  [Agent B completes] -> Results stored
  [Agent C completes] -> Results stored

Orchestrator collects results and proceeds with implementation
```

**Key principle**: Background agents should be cheap and fast. Don't fire an expensive model in the background for a simple search task.

### Pattern 5: Constrained Delegation

Workers should have the minimum permissions needed for their task. A search agent doesn't need write access. A planning agent doesn't need to execute commands.

```
Oracle (architecture advisor):
  CAN:    Read files, search code, analyze
  CANNOT: Write files, edit code, delegate to other agents

Explore (codebase search):
  CAN:    Read files, search with grep/glob
  CANNOT: Write, edit, delegate, run commands

Sisyphus-Junior (task executor):
  CAN:    Read, write, edit, run commands
  CANNOT: Delegate to other agents (prevents infinite delegation)
```

**Key principle**: Constraints aren't about trust. They're about focus. An agent that can't write code won't accidentally start coding when it should be advising. An agent that can't delegate won't create sub-sub-sub-tasks indefinitely.

---

## The Intent Gate: Classifying Before Acting

One of the most impactful patterns is classifying user intent before taking action. Most AI tools take your prompt literally and start executing immediately. An intent gate adds a classification step:

```
User: "Look into the auth module"

Without intent gate:
  -> Agent starts reading auth files (might be correct, might not)

With intent gate:
  -> Classify: Is this research, implementation, or investigation?
  -> "Look into" = investigation intent
  -> Route: Fire explore agents, gather findings, present summary
  -> Don't start implementing anything
```

**Why this matters**: The same words can mean very different things:

| What the user says | What they probably mean | Correct action |
|---|---|---|
| "Look into X" | Investigate, report findings | Research only, don't change code |
| "Fix X" | Something is broken | Diagnose, make minimal fix |
| "Improve X" | Open-ended enhancement | Assess current state, propose approach, wait for approval |
| "Add X" | Specific feature request | Plan then implement |

Without intent classification, agents tend to default to "start coding" for everything.

---

## Prompt Engineering for Orchestrators

The orchestrator's system prompt is the most critical piece of a multi-agent system. It defines:

1. **Available agents**: What workers exist and what they're good at
2. **Delegation rules**: When to use which agent (a decision table)
3. **Workflow patterns**: How to decompose tasks, verify results, handle failures
4. **Constraints**: What the orchestrator must never do (e.g., suppress type errors, commit without permission)

**Key insight**: The orchestrator's prompt is not static. It's assembled dynamically based on:
- Which agents are actually available (model availability varies by provider)
- Which skills are installed (project-specific and user-specific)
- Which categories are configured
- The current environment (directory, git state, timezone)

This dynamic assembly is one of the most powerful patterns in oh-my-opencode and is covered in detail in [04 - Dynamic Prompt Assembly](./04-prompt-engineering.md).

---

## Anti-Patterns to Avoid

### Anti-Pattern: Infinite Delegation

Orchestrator -> Worker A -> Worker B -> Worker C -> ...

If workers can delegate to other workers, you get runaway delegation chains. Each level adds latency, cost, and potential for misinterpretation.

**Fix**: Workers cannot delegate. Only the orchestrator delegates.

### Anti-Pattern: The God Agent

One agent that does everything: plans, codes, searches, reviews, deploys. This is just a single-agent system with extra steps.

**Fix**: Enforce role separation through tool restrictions, not just instructions. An agent without write tools *cannot* write, no matter what its prompt says.

### Anti-Pattern: Over-Exploration

Firing 10 search agents for every trivial question. More agents isn't always better.

**Fix**: The orchestrator should classify task complexity first. Trivial tasks (known file, simple edit) should be done directly. Only fire explorers when you genuinely don't know where something is.

### Anti-Pattern: Context Pollution

Passing every piece of information from every agent to every other agent. This defeats the purpose of separate context windows.

**Fix**: Agents should return *distilled summaries*, not raw output. The orchestrator should pass only relevant context to downstream workers.

---

## How oh-my-opencode Implements These Patterns

| Pattern | oh-my-opencode Implementation |
|---------|------------------------------|
| Orchestrator-Worker | Sisyphus orchestrates; Oracle, Explore, Librarian, Junior execute |
| Planning-Execution Separation | Prometheus plans (read-only); Atlas executes (delegates to workers) |
| Tiered Cost | Categories map to models: `quick` uses cheap models, `ultrabrain` uses expensive ones |
| Background Execution | `BackgroundManager` runs agents in parallel with per-provider concurrency limits |
| Constrained Delegation | Tool deny-lists per agent enforced at the plugin level |
| Intent Gate | `keyword-detector` hook classifies intent before Sisyphus acts |
| Dynamic Prompt Assembly | Agent metadata, skills, and categories are composed into Sisyphus's prompt at runtime |

The next doc ([02 - Plugin Architecture](./02-architecture.md)) shows exactly how these are wired together in the codebase.

---

## Key Takeaways

1. **Multi-agent orchestration solves real problems**: context limits, cognitive drift, model specialization, and serial execution. It's not just complexity for complexity's sake.

2. **The orchestrator's job is to decompose and delegate**, not to do the work itself. The smarter the orchestrator's routing, the better the results.

3. **Constraints are features, not limitations**. An agent that can't write code is an agent that won't accidentally break your codebase.

4. **Dynamic prompt assembly** is the secret sauce. Static prompts can't adapt to available agents, installed skills, or project-specific context.

5. **These patterns are transferable**. You don't need oh-my-opencode to use orchestrator-worker, planning-execution separation, or intent classification. The patterns work in any multi-agent system.

---

Next: [02 - Plugin Architecture](./02-architecture.md)
