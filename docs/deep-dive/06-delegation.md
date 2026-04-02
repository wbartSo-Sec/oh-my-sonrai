# Task Delegation

How the category system routes tasks to optimal models, how skills inject specialized knowledge, how background agents execute in parallel, and how the `task` tool orchestrates it all.

---

## The Core Idea

When Sisyphus needs work done, it doesn't pick a model. It picks a **category** that describes the type of work. The system resolves that category to the right model, loads the right skills, and dispatches to a worker agent.

```
Sisyphus decides: "This is a frontend task"
  |
  v
task(category="visual-engineering", load_skills=["frontend-ui-ux"], prompt="...")
  |
  v
Category "visual-engineering" resolves to: Gemini 3.1 Pro (high)
Skills resolve to: frontend-ui-ux SKILL.md content
  |
  v
Sisyphus-Junior spawns with:
  - Model: Gemini 3.1 Pro
  - System: frontend-ui-ux skill content + category context
  - Prompt: The actual task
```

This decoupling is intentional. The orchestrator thinks in terms of work types, not model names. The system handles model selection.

---

## The Category System

### Built-in Categories

| Category | Default Model | Use Case |
|----------|--------------|----------|
| `visual-engineering` | Gemini 3.1 Pro (high) | Frontend, UI/UX, design, styling, animation |
| `ultrabrain` | GPT-5.4 (xhigh) | Hard logic, complex architecture, deep reasoning |
| `deep` | GPT-5.3 Codex (medium) | Autonomous research + implementation |
| `artistry` | Gemini 3.1 Pro (high) | Creative, unconventional approaches |
| `quick` | GPT-5.4 Mini | Single-file changes, typos, trivial tasks |
| `unspecified-low` | Claude Sonnet 4.6 | General tasks, low effort |
| `unspecified-high` | Claude Opus 4.6 (max) | General tasks, high effort |
| `writing` | Gemini 3 Flash | Documentation, prose, technical writing |

### Custom Categories

Users can define custom categories or override built-in ones:

```jsonc
{
  "categories": {
    "korean-writer": {
      "model": "google/gemini-3-flash",
      "temperature": 0.5,
      "description": "Korean technical writing",
      "prompt_append": "You are a Korean technical writer."
    },
    "visual-engineering": {
      "model": "openai/gpt-5.4",
      "temperature": 0.8
    }
  }
}
```

Custom categories appear in Sisyphus's prompt automatically (via the dynamic prompt assembly covered in [doc 04](./04-prompt-engineering.md)).

### Category Configuration Options

| Option | Type | Description |
|--------|------|-------------|
| `model` | string | AI model to use |
| `variant` | string | Model variant (max, high, medium, xhigh) |
| `temperature` | number | Sampling temperature |
| `top_p` | number | Nucleus sampling |
| `maxTokens` | number | Max response tokens |
| `thinking` | object | Anthropic extended thinking config |
| `reasoningEffort` | string | OpenAI reasoning effort |
| `prompt_append` | string | Appended to system prompt when this category is used |
| `description` | string | Shown in Sisyphus's category table |
| `fallback_models` | array | Fallback chain for this category |
| `is_unstable_agent` | boolean | Force background mode for monitoring |

---

## The `task` Tool

The `task` tool is the primary delegation mechanism. It's defined in `src/tools/delegate-task/` and is the most complex tool in the plugin.

### Parameters

```typescript
task({
  category: "visual-engineering",    // OR subagent_type: "explore"
  load_skills: ["frontend-ui-ux"],   // Skills to inject (REQUIRED, use [] for none)
  prompt: "...",                     // The task description
  description: "Fix sidebar layout", // Short label
  run_in_background: true,           // Async (true) or sync (false)
  session_id: "ses_abc123",          // Continue existing session (optional)
})
```

Two dispatch modes:
- **`category`**: Spawns Sisyphus-Junior with the category's model. For implementation work.
- **`subagent_type`**: Routes to a named agent (explore, librarian, oracle, etc.). For specialized work.

You must provide either `category` or `subagent_type`, not both.

### The Execution Pipeline

```
task() called by agent
  |
  v
Validate arguments
  - run_in_background is required
  - load_skills is required (can be [])
  - Either category or subagent_type must be set
  |
  v
resolveSkillContent(load_skills)         [skill-resolver.ts]
  - Load SKILL.md content for each requested skill
  - Returns joined skill text + individual contents
  |
  v
resolveParentContext()                   [parent-context-resolver.ts]
  - Get the parent session's model (for model inheritance)
  |
  v
Branch: category or subagent_type?
  |
  +--> Category path:
  |      resolveCategoryExecution()      [category-resolver.ts]
  |        - mergeCategories() (built-in + user)
  |        - resolveModelForDelegateTask() (fuzzy match available models)
  |        - Build fallback chain
  |        - Returns: agent, model, prompt_append, fallback info
  |      |
  |      v
  |      buildSystemContent()             [prompt-builder.ts]
  |        - Combine: skill content + category prompt_append + agents context
  |        - Apply token limits via buildSystemContentWithTokenLimit()
  |      |
  |      v
  |      Dispatch to executor
  |
  +--> Subagent path:
         resolveSubagentExecution()       [subagent-resolver.ts]
           - Map subagent_type to agent name
           - Block direct "sisyphus-junior" invocation
           - Returns: agent name, model config
         |
         v
         Dispatch to executor
```

### Category Resolution in Detail

`resolveCategoryExecution()` in `src/tools/delegate-task/category-resolver.ts`:

1. **Merge categories**: Combine built-in defaults with user config
2. **Resolve category config**: Find the category definition
3. **Resolve model**: Call `resolveModelForDelegateTask()` which:
   - Checks if user explicitly set a model for this category
   - Fuzzy-matches the category's default model against connected providers
   - Falls back through the category's fallback chain
   - On cold cache (first run), returns `{ skipped: true }` to use system default
4. **Build fallback chain**: Ordered list of fallback models for runtime retry
5. **Return**: The agent to use (always Sisyphus-Junior for categories), the resolved model, prompt append text, and fallback chain

### Model Selection Logic

`resolveModelForDelegateTask()` in `src/tools/delegate-task/model-selection.ts`:

```
1. User explicitly set model for this category?
   YES -> Use it exactly
   |
2. Available models cache is empty? (cold cache / first run)
   YES -> Return { skipped: true }, defer to system default
   |
3. Fuzzy match category default model against connected providers
   MATCH -> Use matched model
   |
4. Try user-configured fallback_models in order
   MATCH -> Use first match
   |
5. Try built-in fallback chain in order
   MATCH -> Use first match
   |
6. No match -> Return undefined, system default will be used
```

The fuzzy matching is important. Provider names in the fallback chain use pipe-separated alternatives: `openai|github-copilot|opencode/gpt-5.4`. This means "try this model with any of these providers."

---

## The Skill System

### What Skills Are

A skill is a SKILL.md file that contains:
1. **YAML frontmatter**: Name, description, optional embedded MCP server config
2. **Markdown body**: Instructions injected into the agent's system prompt

```markdown
---
name: my-skill
description: Specialized knowledge for X
mcp:
  my-server:
    command: npx
    args: ["-y", "my-mcp-server"]
---

# My Skill Instructions

You are an expert in X. When working on Y, follow these rules:
1. Always check Z before modifying...
```

### How Skills Are Loaded

When `load_skills=["frontend-ui-ux"]` is passed to the `task` tool:

1. `resolveSkillContent()` calls `resolveMultipleSkillsAsync()` from the skill loader
2. The loader searches for the skill in priority order:
   - `.opencode/skills/frontend-ui-ux/SKILL.md`
   - `~/.config/opencode/skills/frontend-ui-ux/SKILL.md`
   - `.claude/skills/frontend-ui-ux/SKILL.md`
   - `.agents/skills/frontend-ui-ux/SKILL.md`
   - `~/.agents/skills/frontend-ui-ux/SKILL.md`
   - Built-in skills (shipped with the plugin)
3. The markdown body is extracted and returned as skill content
4. If the skill has an MCP section, `SkillMcpManager` starts the MCP server

### Skill-Embedded MCPs

Skills can carry their own MCP servers. This is a key innovation: instead of loading all MCP tools into every agent's context (which wastes tokens), tools are loaded only when the relevant skill is activated.

```yaml
mcp:
  playwright:
    command: npx
    args: ["@playwright/mcp@latest"]
```

When the `playwright` skill is loaded:
1. The MCP server is started by `SkillMcpManager`
2. The server's tools become available to the agent
3. When the task completes, the MCP server is cleaned up

The agent only sees the MCP tools when it needs them.

### How Skill Content Becomes System Prompt

```
resolveSkillContent() returns raw SKILL.md body text
  |
  v
buildSystemContent() combines:
  1. Skill content (prepended)
  2. Category prompt_append
  3. Agents context (available agents for delegation awareness)
  |
  v
buildSystemContentWithTokenLimit() enforces size limits:
  - First truncates skill content if over limit
  - Then truncates category append
  - Then truncates agents context
  - Task prompt is never truncated
  |
  v
sendSyncPrompt() sends to subagent session:
  system: [combined system content]     // Skills + category + context
  parts:  [task prompt]                 // The actual work to do
```

---

## Background Execution

### How It Works

When `run_in_background=true`:

```
task(run_in_background=true, ...)
  |
  v
executeBackgroundTask()                [background-task.ts]
  |
  +--> BackgroundManager.launch(taskConfig)
  |      - Check concurrency limits
  |      - Create OpenCode session
  |      - Send prompt
  |      - Return task_id immediately
  |
  v
Return to calling agent: "Background Task ID: bg_abc123"
  |
  [Agent continues other work]
  |
  [System notification on completion]
  |
  v
Agent calls: background_output(task_id="bg_abc123")
  |
  v
Result returned
```

### When `run_in_background=false`:

```
task(run_in_background=false, ...)
  |
  v
executeSyncTask()                      [sync-task.ts]
  |
  +--> sync-session-creator: Create session
  +--> sync-prompt-sender: Send prompt with system content
  +--> sync-session-poller: Poll until session idle
  +--> sync-result-fetcher: Extract result
  |
  v
Result returned directly to calling agent
```

### Concurrency Management

The `BackgroundManager` enforces concurrency limits at two levels:

```jsonc
{
  "background_task": {
    "providerConcurrency": {
      "anthropic": 3,    // Max 3 concurrent Anthropic tasks
      "openai": 5,       // Max 5 concurrent OpenAI tasks
      "opencode": 10     // Max 10 concurrent OpenCode Zen tasks
    },
    "modelConcurrency": {
      "anthropic/claude-opus-4-6": 2  // Max 2 concurrent Opus tasks
    }
  }
}
```

Priority: `modelConcurrency` > `providerConcurrency` > `defaultConcurrency`

This prevents overwhelming a single provider with too many concurrent requests, which would cause rate limiting.

### Unstable Agent Handling

Categories marked with `is_unstable_agent: true` (auto-enabled for Gemini models) get special treatment:

- Forced to run in background mode even if `run_in_background=false` was requested
- Monitored by the `unstable-agent-babysitter` hook
- Recovery mechanisms if the agent crashes or stalls

---

## Session Continuity

The `session_id` parameter enables multi-turn conversations with the same subagent:

```typescript
// First delegation
result = task(category="deep", prompt="Implement auth", run_in_background=false)
// session_id returned in result

// Follow-up (preserves full context)
task(session_id="ses_abc123", prompt="Fix: Type error on line 42")
```

**Why this matters**:
- The subagent has full conversation context preserved
- No repeated file reads, exploration, or setup
- Saves 70%+ tokens on follow-ups
- The subagent knows what it already tried

Continuation works for both sync and background modes:
- `executeBackgroundContinuation()` for async follow-ups
- `executeSyncContinuation()` for sync follow-ups

---

## Sisyphus-Junior: The Worker Agent

When a category-based task is dispatched, it's executed by Sisyphus-Junior. This is a special agent that:

1. **Cannot delegate**: Its `task` and `call_omo_agent` tools are denied. This prevents infinite delegation chains.
2. **Uses the category's model**: Its model is set by the resolved category, not its own default.
3. **Cannot be invoked directly**: `subagent_type="sisyphus-junior"` is blocked by `subagent-resolver.ts`. You must use `category` to spawn a Junior instance.
4. **Has focused context**: Only gets the skill content and task prompt, not Sisyphus's full orchestration prompt.

The Junior agent has its own general fallback chain (`claude-sonnet-4-6` -> `kimi-k2.5` -> `gpt-5.4 medium` -> `minimax-m2.7` -> `big-pickle`) which is used if the category model fails at runtime.

---

## Putting It Together: A Complete Delegation Example

Sisyphus receives: "Add a responsive chart component to the dashboard"

```
1. Sisyphus classifies: visual/frontend work
   -> category = "visual-engineering"
   -> skills = ["frontend-ui-ux"]

2. task(category="visual-engineering", load_skills=["frontend-ui-ux"],
        prompt="TASK: Add responsive chart component to dashboard
                EXPECTED: Chart renders in src/components/Dashboard.tsx
                MUST DO: Use existing Chart library from package.json
                MUST NOT: Add new dependencies
                CONTEXT: Dashboard.tsx uses Tailwind CSS grid layout",
        run_in_background=false)

3. resolveSkillContent(["frontend-ui-ux"])
   -> Loads frontend-ui-ux SKILL.md body (~500 tokens of design instructions)

4. resolveCategoryExecution("visual-engineering")
   -> Model: google/gemini-3.1-pro (high)  [matched against connected providers]
   -> prompt_append: none
   -> fallback: glm-5 -> claude-opus-4-6 -> kimi-k2.5

5. buildSystemContent({
     skillContent: "# Frontend UI/UX\nYou are a designer-turned-developer...",
     categoryPromptAppend: "",
     agentsContext: "Available agents: explore (codebase grep), librarian (docs)..."
   })
   -> Token limit check: OK (within limits)

6. executeSyncTask()
   -> Create session with agent="sisyphus-junior", model="google/gemini-3.1-pro"
   -> Send: system=[skill content + agents context], parts=[task prompt]
   -> Poll until idle
   -> Fetch result

7. Result returned to Sisyphus
   -> Sisyphus verifies: lsp_diagnostics on changed files
   -> Reports completion to user
```

---

## Source File Map

| File | Purpose |
|------|---------|
| `src/tools/delegate-task/tools.ts` | Main `task` tool definition and execute flow |
| `src/tools/delegate-task/category-resolver.ts` | Category -> model resolution |
| `src/tools/delegate-task/model-selection.ts` | Model fuzzy matching and fallback |
| `src/tools/delegate-task/prompt-builder.ts` | System content composition |
| `src/tools/delegate-task/token-limiter.ts` | Token limit enforcement |
| `src/tools/delegate-task/skill-resolver.ts` | Skill content loading |
| `src/tools/delegate-task/executor.ts` | Execution routing (background/sync/unstable) |
| `src/tools/delegate-task/background-task.ts` | Background task launcher |
| `src/tools/delegate-task/sync-task.ts` | Sync execution orchestrator |
| `src/tools/delegate-task/sync-session-creator.ts` | Session creation |
| `src/tools/delegate-task/sync-prompt-sender.ts` | Prompt injection |
| `src/tools/delegate-task/sync-session-poller.ts` | Session polling |
| `src/tools/delegate-task/sync-result-fetcher.ts` | Result extraction |
| `src/tools/delegate-task/subagent-resolver.ts` | Subagent type resolution |
| `src/tools/delegate-task/parent-context-resolver.ts` | Parent session context |
| `src/features/opencode-skill-loader/skill-content.ts` | Skill SKILL.md resolution |

---

## Key Takeaways

1. **Categories decouple task intent from model selection.** The orchestrator says "this is frontend work." The system picks Gemini 3.1 Pro. If Gemini isn't available, it falls through to the next best option.

2. **Skills are on-demand context injection.** Instead of bloating every agent with every tool and instruction, skills load domain-specific knowledge only when needed.

3. **Background execution enables parallelism.** Fire multiple agents simultaneously with concurrency limits per provider. The system notifies on completion.

4. **Session continuity saves tokens.** Continuing an existing session preserves full context, avoiding repeated setup.

5. **Sisyphus-Junior is the execution boundary.** It can write code but can't delegate. This prevents infinite delegation chains while still allowing the orchestrator pattern to work.

6. **The `task` tool is the most complex tool in the plugin** because it bridges the gap between high-level orchestration decisions and low-level session management.

---

Next: [07 - Extending oh-my-opencode](./07-extending.md)
