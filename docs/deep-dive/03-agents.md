# The Agent System

How agents are defined, built, and configured. The factory pattern, agent modes, model resolution, fallback chains, and tool restrictions.

---

## What Is an Agent?

In OpenCode, an agent is a configured AI assistant with:
- A **system prompt** (instructions that shape its behavior)
- A **model** (which LLM powers it)
- A **mode** (how it can be invoked)
- **Tool permissions** (what tools it can use)
- **Parameters** (temperature, reasoning effort, etc.)

oh-my-opencode registers 11 built-in agents with OpenCode. Each is specialized for a specific type of work, runs on a model chosen for that work, and has tool restrictions that enforce its role.

---

## The Agent Inventory

### Primary Agents (Tab-selectable)

These are the agents you interact with directly. Press Tab to cycle through them.

| Agent | Default Model | Purpose |
|-------|--------------|---------|
| **Sisyphus** | Claude Opus 4.6 | Main orchestrator. Plans, delegates, verifies. The "project manager." |
| **Hephaestus** | GPT-5.3 Codex | Autonomous deep worker. Give it a goal, not a recipe. |
| **Prometheus** | Claude Opus 4.6 | Strategic planner. Interviews you, then writes a detailed plan. |
| **Atlas** | Claude Sonnet 4.6 | Plan executor. Reads Prometheus plans, delegates to workers. |

### Subagents (Delegated to by primary agents)

These are invoked by primary agents or via `@mention`. You don't typically interact with them directly.

| Agent | Default Model | Purpose |
|-------|--------------|---------|
| **Oracle** | GPT-5.4 | Architecture advisor. Read-only, high-IQ consultation. |
| **Librarian** | MiniMax M2.7 | External documentation and OSS code search. |
| **Explore** | Grok Code Fast 1 | Fast codebase grep. Pattern discovery. |
| **Multimodal Looker** | GPT-5.4 | PDF/image/diagram analysis. |
| **Metis** | Claude Opus 4.6 | Pre-planning gap analysis. Catches what Prometheus missed. |
| **Momus** | GPT-5.4 | Plan reviewer. Validates clarity and completeness. |
| **Sisyphus-Junior** | (varies by category) | Category-spawned executor. The actual "hands" that write code. |

---

## The Factory Pattern

Every agent is created by a factory function following the same pattern:

```typescript
const createOracleAgent: AgentFactory = (model: string) => ({
  instructions: "You are a read-only architecture consultant...",
  model,
  temperature: 0.1,
  // ...other config
})
createOracleAgent.mode = "subagent"
```

Key characteristics:
1. **Factory function**: Takes a `model` string, returns an `AgentConfig` object
2. **Static mode property**: The `mode` is set on the function itself, not the returned config. This allows checking the mode before instantiation (e.g., to decide model resolution strategy).
3. **Deterministic**: Given the same model string, the factory always produces the same config.

The factories are registered in `src/agents/builtin-agents.ts`:

```typescript
const agentSources: Record<BuiltinAgentName, AgentSource> = {
  sisyphus: createSisyphusAgent,
  hephaestus: createHephaestusAgent,
  oracle: createOracleAgent,
  librarian: createLibrarianAgent,
  explore: createExploreAgent,
  "multimodal-looker": createMultimodalLookerAgent,
  metis: createMetisAgent,
  momus: createMomusAgent,
  atlas: createAtlasAgent,
  "sisyphus-junior": createSisyphusJuniorAgentWithOverrides,
}
```

---

## Agent Modes

The `mode` property determines how an agent can be used:

| Mode | Behavior | Model Selection | Examples |
|------|----------|----------------|----------|
| `primary` | Tab-selectable in UI | Respects the user's UI-selected model | Sisyphus, Atlas |
| `subagent` | Invoked by other agents or `@mention` | Uses its own fallback chain, ignores UI selection | Oracle, Explore, Librarian |
| `all` | Available in both contexts | Depends on invocation context | Sisyphus-Junior |

**Why this matters**: When you select a model in the OpenCode UI, primary agents use that model. But subagents don't. Oracle always tries GPT-5.4 first regardless of what you selected in the UI, because it's optimized for that model.

---

## Model Resolution

When an agent is created, its model is resolved through a 4-step priority chain:

```
1. User override     (from oh-my-opencode.jsonc config)
   ↓ if not set
2. Category default  (model inherited from assigned category)
   ↓ if not set
3. Fallback chain    (try providers in order until one works)
   ↓ if none available
4. System default    (OpenCode's configured default model)
```

### Fallback Chains

Each agent has a built-in fallback chain that tries multiple providers. This is critical because users have different provider subscriptions.

Example: Oracle's fallback chain:
```
1st try: openai/gpt-5.4 (high)
2nd try: google/gemini-3.1-pro (high)
3rd try: anthropic/claude-opus-4-6 (max)
4th try: opencode-go/glm-5
```

The system uses fuzzy matching against connected providers. If you have an OpenAI key, Oracle gets GPT-5.4. If you only have Google, it falls back to Gemini. If you only have Anthropic, it falls back to Claude Opus.

### Cold Cache Behavior

On first run, the plugin doesn't yet know which models are available (the provider cache hasn't been built). In this case, the model resolution returns `{ skipped: true }` and defers to the system default model. This prevents false failures during initial setup.

### User Overrides

Users can override any agent's model in their config:

```jsonc
{
  "agents": {
    "oracle": {
      "model": "anthropic/claude-opus-4-6",
      "variant": "max"
    },
    "explore": {
      "model": "openai/gpt-5-nano"
    }
  }
}
```

User overrides take absolute priority. Even on cold cache, explicit user configuration overrides the fallback chain.

---

## Tool Restrictions

Agents are constrained by tool deny-lists. This isn't about trust; it's about focus. An agent without write access physically cannot modify your code, regardless of what its prompt says.

| Agent | Denied Tools | Why |
|-------|-------------|-----|
| **Oracle** | write, edit, task, call_omo_agent | Read-only advisor. Must not change code or delegate. |
| **Librarian** | write, edit, task, call_omo_agent | Research only. Must not change code or delegate. |
| **Explore** | write, edit, task, call_omo_agent | Search only. Must not change code or delegate. |
| **Multimodal Looker** | Everything except `read` | Vision analysis only. Minimal tool access. |
| **Atlas** | task, call_omo_agent | Orchestrator that delegates via its own mechanism, not the task tool. |
| **Momus** | write, edit, task | Review only. Must not change code or delegate. |
| **Sisyphus-Junior** | task, call_omo_agent | Worker. Must not sub-delegate (prevents infinite delegation chains). |

The "cannot delegate" restriction on workers (Sisyphus-Junior) is particularly important. Without it, you'd get:
```
Sisyphus delegates to Junior-A
  Junior-A delegates to Junior-B
    Junior-B delegates to Junior-C
      ... (infinite chain, each losing context)
```

---

## Agent Prompt Metadata

Each agent carries metadata that Sisyphus uses to build its delegation decisions. This is defined in the `AgentPromptMetadata` type:

```typescript
interface AgentPromptMetadata {
  category: "exploration" | "specialist" | "advisor" | "utility"
  cost: "FREE" | "CHEAP" | "EXPENSIVE"
  triggers: DelegationTrigger[]     // When to use this agent
  useWhen?: string[]                // Detailed use cases
  avoidWhen?: string[]              // When NOT to use
  dedicatedSection?: string         // Special prompt section (e.g., Oracle's usage guide)
  keyTrigger?: string               // Phase 0 trigger for Sisyphus
}
```

This metadata is used by the dynamic prompt builder (covered in [04 - Dynamic Prompt Assembly](./04-prompt-engineering.md)) to generate Sisyphus's delegation table, tool selection guide, and key triggers. When you add or remove an agent, the prompt updates automatically.

---

## How Agents Discover Each Other

oh-my-opencode's agents coexist with OpenCode-native agents. The discovery flow:

```
1. oh-my-opencode creates its built-in agents (Sisyphus, Oracle, etc.)
2. OpenCode registers agents from opencode.json and .opencode/agents/*.md
3. oh-my-opencode parses these as customAgentSummaries
4. Non-duplicate, non-disabled custom agents are added to availableAgents
5. Sisyphus's prompt is built with ALL available agents (built-in + custom)
```

This means if you create a `security-auditor` agent in `.opencode/agents/security-auditor.md`, Sisyphus will see it in its delegation table and can route work to it.

The parsing is tolerant: hidden agents (`hidden: true`), disabled agents (`disabled: true` or `enabled: false`), and agents that conflict with built-in names are filtered out.

---

## Environment Context

After an agent's config is built, environment context is appended to its prompt:

```typescript
applyEnvironmentContext(agentConfig, directory, options)
// Appends:
// <omo-env>
//   Timezone: America/New_York
//   Locale: en-US
// </omo-env>
```

This small block gives agents awareness of the user's timezone and locale. It's appended to every agent that has it enabled (can be disabled via `experimental.disable_omo_env`).

---

## Sisyphus vs. Hephaestus: Two Philosophies

The two primary agents represent fundamentally different approaches:

| Aspect | Sisyphus | Hephaestus |
|--------|----------|------------|
| **Philosophy** | Decompose and delegate | Explore and execute |
| **Model family** | Claude-optimized | GPT-optimized |
| **How it works** | Classifies intent, creates todos, delegates to specialized agents | Researches the codebase itself, plans internally, executes end-to-end |
| **Delegation** | Heavy. Fires explore/librarian agents, delegates via categories. | Moderate. Uses explore/librarian for research, does implementation itself. |
| **Best for** | Teams, complex multi-step work, when you want visibility into the plan | Deep architectural work, complex debugging, when you want autonomous execution |
| **Prompt style** | Structured roles, phases, rules, constraints | Goal-oriented, principle-driven, autonomous |

Sisyphus has two prompt variants:
- **Default** (`buildDefaultSisyphusPrompt`): For Claude, Kimi, GLM and similar instruction-following models
- **GPT-5.4** (`buildGpt54SisyphusPrompt`): A specialized 8-block structured prompt for GPT-5.4's reasoning style

Hephaestus only runs on GPT models. The `no-hephaestus-non-gpt` hook prevents it from being used with non-GPT models.

---

## Configuration Reference

### Agent Override Options

| Option | Type | Description |
|--------|------|-------------|
| `model` | string | Model override (provider/model format) |
| `fallback_models` | string or array | Fallback chain on errors |
| `temperature` | number | Sampling temperature (0.0-2.0) |
| `top_p` | number | Nucleus sampling |
| `prompt` | string | Replace system prompt (supports file:// URIs) |
| `prompt_append` | string | Append to system prompt (supports file:// URIs) |
| `variant` | string | Model variant (max, high, medium, low, xhigh) |
| `thinking` | object | Anthropic extended thinking config |
| `reasoningEffort` | string | OpenAI reasoning effort |
| `disable` | boolean | Disable this agent entirely |

### Disabling Agents

```jsonc
{
  "disabled_agents": ["oracle", "multimodal-looker"]
}
```

Disabled agents are not created, not registered with OpenCode, and not included in Sisyphus's delegation table.

---

## Source File Map

| File | Purpose |
|------|---------|
| `src/agents/types.ts` | AgentFactory, AgentMode, AgentPromptMetadata types |
| `src/agents/builtin-agents.ts` | Agent registry, createBuiltinAgents() |
| `src/agents/sisyphus.ts` | Sisyphus factory + dynamic prompt builder |
| `src/agents/sisyphus/gpt-5-4.ts` | GPT-5.4 specialized Sisyphus prompt |
| `src/agents/sisyphus/default.ts` | Default (Claude) Sisyphus prompt |
| `src/agents/hephaestus/` | Hephaestus factory |
| `src/agents/oracle.ts` | Oracle factory |
| `src/agents/librarian.ts` | Librarian factory |
| `src/agents/explore.ts` | Explore factory |
| `src/agents/metis.ts` | Metis factory |
| `src/agents/momus.ts` | Momus factory |
| `src/agents/atlas/` | Atlas factory |
| `src/agents/sisyphus-junior/` | Sisyphus-Junior factory |
| `src/agents/dynamic-agent-prompt-builder.ts` | Prompt section builders |
| `src/agents/env-context.ts` | Environment context generator |
| `src/agents/builtin-agents/sisyphus-agent.ts` | Sisyphus conditional creation |
| `src/agents/builtin-agents/general-agents.ts` | Non-Sisyphus agent collection |
| `src/agents/custom-agent-summaries.ts` | OpenCode-native agent discovery |

---

## Key Takeaways

1. **Agents are factory functions** that take a model and return a config. The factory pattern keeps agent definitions pure and testable.

2. **Three modes** (primary, subagent, all) control how agents are invoked and how their models are selected.

3. **Model resolution is a 4-step chain**: user override, category default, fallback chain, system default. Fallback chains make the system resilient to missing providers.

4. **Tool restrictions enforce roles** at the API level. An agent that can't write code won't accidentally modify your codebase, regardless of its prompt.

5. **Agent metadata drives prompt assembly**. Adding a new agent automatically updates Sisyphus's delegation table, tool selection guide, and key triggers.

6. **OpenCode-native agents coexist**. Custom agents defined in `opencode.json` or `.opencode/agents/*.md` are auto-discovered and added to Sisyphus's available agents list.

---

Next: [04 - Dynamic Prompt Assembly](./04-prompt-engineering.md)
