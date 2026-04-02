# Dynamic Prompt Assembly

How Sisyphus's system prompt is built at runtime from agent metadata, available skills, categories, and environment context. This is the most transferable knowledge in this series.

---

## Why Dynamic Prompts Matter

A static system prompt breaks when the environment changes. Consider what happens when:
- A user doesn't have an OpenAI key (Oracle isn't available, but the prompt still references it)
- A project has custom skills installed (the prompt doesn't mention them)
- A new agent is added to the plugin (the delegation table is stale)

oh-my-opencode solves this by assembling the prompt at agent creation time. The prompt adapts to:
- Which agents are actually available (based on connected providers)
- Which skills are installed (project-specific and user-specific)
- Which categories are configured (built-in and custom)
- The current environment (directory, timezone, locale)

The result: Sisyphus always has an accurate, complete picture of its capabilities.

---

## The Assembly Flow

```
maybeCreateSisyphusConfig()          [src/agents/builtin-agents/sisyphus-agent.ts]
  |
  +--> Resolve model (fallback chain)
  +--> Collect availableAgents, availableSkills, availableCategories
  |
  v
createSisyphusAgent(model, agents, skills, categories)   [src/agents/sisyphus.ts]
  |
  +--> Is model GPT-5.4?
  |      YES --> buildGpt54SisyphusPrompt()    [src/agents/sisyphus/gpt-5-4.ts]
  |      NO  --> buildDefaultSisyphusPrompt()  [src/agents/sisyphus/default.ts]
  |
  v
buildDynamicSisyphusPrompt(model, agents, tools, skills, categories)
  |
  +--> buildKeyTriggersSection(agents, skills)
  +--> buildToolSelectionTable(agents, tools, skills)
  +--> buildExploreSection(agents)
  +--> buildLibrarianSection(agents)
  +--> buildCategorySkillsDelegationGuide(categories, skills)
  +--> buildDelegationTable(agents)
  +--> buildOracleSection(agents)
  +--> buildParallelDelegationSection(model, categories)
  +--> buildUltraworkSection(agents, categories, skills)
  +--> buildHardBlocksSection()
  +--> buildAntiPatternsSection()
  +--> buildAntiDuplicationSection()
  |
  v
Final prompt string (template literal interpolating all sections)
  |
  v
applyOverrides(config, userOverrides)     [user's prompt_append, etc.]
  |
  v
applyEnvironmentContext(config, dir)      [appends <omo-env> block]
  |
  v
AgentConfig.instructions = final prompt
```

---

## The Section Builders

All section builders live in `src/agents/dynamic-agent-prompt-builder.ts`. Each takes the available agents/skills/categories and produces a markdown string that gets interpolated into the prompt template.

### Key Triggers Section

```typescript
buildKeyTriggersSection(availableAgents, availableSkills)
```

Produces the "Phase 0" triggers that tell Sisyphus when to fire specific agents immediately:

```markdown
- External library/source mentioned -> fire `librarian` background
- 2+ modules involved -> fire `explore` background
- Ambiguous or complex request -> consult Metis before Prometheus
```

These are built from each agent's `keyTrigger` metadata. If Oracle is disabled (not in `availableAgents`), its trigger doesn't appear.

### Tool Selection Table

```typescript
buildToolSelectionTable(availableAgents, tools, skills)
```

Produces the table Sisyphus uses to decide which agent to fire:

```markdown
| Agent | Cost | Description |
|-------|------|-------------|
| explore | FREE | Contextual grep for codebases |
| librarian | CHEAP | Multi-repo analysis, documentation lookup |
| oracle | EXPENSIVE | Read-only consultation agent |
```

Cost tiers are from each agent's `cost` metadata. The table only includes agents that are actually available.

### Delegation Table

```typescript
buildDelegationTable(availableAgents)
```

Maps work domains to agent recommendations:

```markdown
| Domain | Agent | When |
|--------|-------|------|
| Architecture decisions | oracle | Multi-system tradeoffs, unfamiliar patterns |
| Hard debugging | oracle | After 2+ failed fix attempts |
| External docs/code | librarian | Unfamiliar packages/libraries |
| Codebase search | explore | Find patterns and styles |
```

Built from each agent's `triggers` metadata (an array of `{ domain, trigger }` objects).

### Category + Skills Delegation Guide

```typescript
buildCategorySkillsDelegationGuide(availableCategories, availableSkills)
```

Generates the category table and available skills list:

```markdown
Available categories:
| Category | Description |
|----------|-------------|
| visual-engineering | Frontend, UI/UX, design |
| ultrabrain | Hard logic, architecture |
| quick | Trivial tasks |

Available skills:
- playwright: Browser automation via Playwright MCP
- git-master: Atomic commits, rebase surgery
- frontend-ui-ux: Designer-turned-developer persona
```

If a user adds a custom category (e.g., `korean-writer`), it appears here automatically. Same for custom skills.

### Dedicated Agent Sections

Some agents have detailed usage guides that get their own prompt sections:

```typescript
buildOracleSection(availableAgents)     // Oracle's consultation protocol
buildExploreSection(availableAgents)    // Explore's usage as "contextual grep"
buildLibrarianSection(availableAgents)  // Librarian's "reference grep" role
```

These use the `dedicatedSection` field from agent metadata. If Oracle is disabled, the entire Oracle section is omitted from the prompt.

### Static Sections

Some sections don't depend on runtime state:

```typescript
buildHardBlocksSection()           // "NEVER violate" rules
buildAntiPatternsSection()         // Common mistakes to avoid
buildAntiDuplicationSection()      // Don't repeat explore agent's work
```

These are always included and provide consistent guardrails.

---

## The Two Prompt Variants

Sisyphus has two completely different prompt templates depending on the model:

### Default Prompt (Claude, Kimi, GLM)

Used for instruction-following models. The prompt is structured as phases:

```
Phase 0: Intent Gate (classify before acting)
Phase 1: Codebase Assessment
Phase 2A: Exploration & Research
Phase 2B: Implementation
Phase 2C: Failure Recovery
Phase 3: Completion

+ Oracle Usage Guide
+ Task Management Rules
+ Communication Style
+ Hard Constraints
```

This prompt is optimized for models that follow structured instructions well. It uses XML-like tags, numbered phases, and explicit decision trees.

### GPT-5.4 Prompt

Used when the resolved model is GPT-5.4. This is a completely different prompt with an 8-block architecture:

```
Block 1: Identity & Principles
Block 2: Intent Classification
Block 3: Execution Loop
Block 4: Delegation System
Block 5: Task Management
Block 6: Communication Style
Block 7: Constraints
Block 8: Special Sections (Oracle, Explore, etc.)
```

This prompt is optimized for GPT-5.4's principle-driven reasoning style. Instead of step-by-step phases, it uses declarative principles and lets the model reason about application.

**Both variants use the same section builders.** The builders produce the dynamic content (delegation tables, skill lists, etc.). The templates arrange that content differently for each model family.

---

## How Inputs Flow to the Prompt

### availableAgents

**Origin**: `collectPendingBuiltinAgents()` in `src/agents/builtin-agents/general-agents.ts`

Each entry has:
```typescript
type AvailableAgent = {
  name: string                    // "oracle", "explore", etc.
  description: string             // Short description
  metadata?: AgentPromptMetadata  // Cost, triggers, dedicated sections
}
```

Built-in agents get their metadata from static definitions. OpenCode-native agents (from `opencode.json` or `.opencode/agents/*.md`) get auto-generated metadata via `buildCustomAgentMetadata()`, which assigns them `category: "specialist"` and `cost: "CHEAP"` by default.

### availableSkills

**Origin**: `createSkillContext()` in `src/plugin/skill-context.ts`

Discovers skills from 5 scopes (in priority order):
1. `.opencode/skills/*/SKILL.md` (project, OpenCode native)
2. `~/.config/opencode/skills/*/SKILL.md` (user, OpenCode native)
3. `.claude/skills/*/SKILL.md` (project, Claude Code compat)
4. `.agents/skills/*/SKILL.md` (project, Agents convention)
5. `~/.agents/skills/*/SKILL.md` (user, Agents convention)

Each entry has:
```typescript
type AvailableSkill = {
  name: string           // "playwright", "git-master", etc.
  description: string    // From SKILL.md frontmatter
  location: string       // "builtin", "project", "user", etc.
}
```

### availableCategories

**Origin**: `createAvailableCategories()` in `src/create-tools.ts`

Built from the merged category config (built-in + user-defined):
```typescript
type AvailableCategory = {
  name: string           // "visual-engineering", "ultrabrain", etc.
  description: string    // From category config or default
}
```

---

## Skill Content Injection (Subagent Prompts)

When Sisyphus delegates via the `task` tool with `load_skills`, the skill content is resolved and injected into the subagent's system prompt. This is a separate mechanism from the Sisyphus prompt assembly described above.

The flow:

```
task(category="visual-engineering", load_skills=["frontend-ui-ux"], prompt="...")
  |
  v
resolveSkillContent(["frontend-ui-ux"])        [skill-resolver.ts]
  |  Reads SKILL.md, extracts markdown body
  v
buildSystemContent({                           [prompt-builder.ts]
  skillContent: "# Frontend UI/UX\nYou are a designer-turned-developer...",
  categoryPromptAppend: "Focus on visual excellence...",
  agentsContext: "Available agents: ..."
})
  |  Combines all system content, applies token limits
  v
buildSystemContentWithTokenLimit(...)          [token-limiter.ts]
  |  Truncates if needed: skills first, then category, then agents context
  v
sendSyncPrompt({                               [sync-prompt-sender.ts]
  system: systemContent,     // Skills + category + agents context
  parts: [taskPrompt],       // The actual task to perform
  agent: "sisyphus-junior",
  model: resolvedCategoryModel
})
```

The skill content becomes the `system` message for the subagent session. The task prompt (with TASK, EXPECTED OUTCOME, MUST DO, etc.) becomes the user message.

Token limits are enforced in this order (first to be truncated):
1. Skill content (trimmed if too long)
2. Category prompt append
3. Agents context

This ensures the actual task prompt is never truncated.

---

## The AgentPromptMetadata Type

This is the interface that connects agent definitions to prompt generation:

```typescript
interface AgentPromptMetadata {
  // Grouping in prompt sections
  category: "exploration" | "specialist" | "advisor" | "utility"
  
  // Cost tier for tool selection table
  cost: "FREE" | "CHEAP" | "EXPENSIVE"
  
  // When to delegate to this agent
  triggers: DelegationTrigger[]
  
  // Detailed usage guidance
  useWhen?: string[]
  avoidWhen?: string[]
  
  // Full markdown section for the prompt
  dedicatedSection?: string
  
  // Display name in prompt
  promptAlias?: string
  
  // Phase 0 key trigger
  keyTrigger?: string
}
```

To add a new agent to the system, you define its factory function AND its metadata. The metadata is what makes Sisyphus aware of the agent. Without metadata, the agent exists but Sisyphus doesn't know when to use it.

---

## Why This Pattern Is Powerful

1. **Self-documenting**: Sisyphus's prompt is always an accurate reflection of available capabilities. No stale references to disabled agents or missing skills.

2. **Zero-maintenance updates**: Add an agent, define its metadata, and the prompt updates everywhere: delegation table, tool selection, key triggers, dedicated sections.

3. **Model-adaptive**: The same dynamic content gets arranged into different prompt structures depending on the model family (Claude vs GPT). The content is decoupled from the template.

4. **User-extensible**: Custom agents, skills, and categories appear in the prompt automatically. The user doesn't need to edit any prompt files.

5. **Testable**: Section builders are pure functions. Given the same inputs, they produce the same output. You can unit test the delegation table builder independently of the rest of the system.

---

## Source File Map

| File | Purpose |
|------|---------|
| `src/agents/dynamic-agent-prompt-builder.ts` | All section builder functions |
| `src/agents/sisyphus.ts` | createSisyphusAgent, buildDynamicSisyphusPrompt |
| `src/agents/sisyphus/default.ts` | Default (Claude) prompt template |
| `src/agents/sisyphus/gpt-5-4.ts` | GPT-5.4 prompt template |
| `src/agents/builtin-agents/sisyphus-agent.ts` | maybeCreateSisyphusConfig entry point |
| `src/agents/env-context.ts` | createEnvContext() |
| `src/agents/builtin-agents/environment-context.ts` | applyEnvironmentContext() |
| `src/agents/custom-agent-summaries.ts` | OpenCode-native agent discovery |
| `src/plugin/skill-context.ts` | createSkillContext, skill discovery |
| `src/tools/delegate-task/prompt-builder.ts` | buildSystemContent for subagents |
| `src/tools/delegate-task/skill-resolver.ts` | resolveSkillContent |
| `src/tools/delegate-task/token-limiter.ts` | Token limit enforcement |

---

## Key Takeaways

1. **Prompts are assembled, not written.** Sisyphus's system prompt is composed from ~12 builder functions, each producing a section based on runtime state.

2. **Agent metadata drives everything.** The `AgentPromptMetadata` type connects agent definitions to prompt sections. Add an agent with metadata, and the prompt updates automatically.

3. **Two model-specific templates** (Claude and GPT-5.4) use the same builder functions but arrange content differently to match each model's reasoning style.

4. **Skill injection for subagents** is a separate pipeline: resolveSkillContent -> buildSystemContent -> token limiting -> session creation.

5. **This pattern is fully extractable.** You don't need the oh-my-opencode plugin to use dynamic prompt assembly. Define metadata per agent, write section builders, compose at runtime. Works in any LLM framework.

---

Next: [05 - The Hook System](./05-hooks.md)
