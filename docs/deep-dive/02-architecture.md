# Plugin Architecture

How oh-my-opencode boots up, what it creates, and how the pieces connect.

---

## The Big Picture

oh-my-opencode is an OpenCode plugin. OpenCode is the AI coding tool (a fork/evolution of Claude Code). Plugins extend OpenCode by hooking into its lifecycle at defined points.

When OpenCode loads, it calls the plugin's entry function. That function returns a `PluginInterface` object with handlers for 8 hook points. Everything the plugin does flows through those 8 handlers.

```
OpenCode starts
  |
  v
Loads oh-my-opencode plugin (src/index.ts)
  |
  v
OhMyOpenCodePlugin(ctx) runs 5 initialization steps
  |
  v
Returns PluginInterface with 8 hook handlers
  |
  v
OpenCode calls those handlers during its lifecycle
```

---

## The 5-Step Initialization Pipeline

The entry point is `OhMyOpenCodePlugin` in `src/index.ts`. It runs 5 steps in order, each producing data that the next step consumes:

```
Step 1: loadPluginConfig(ctx)
    |
    v  pluginConfig
Step 2: createManagers({ ctx, pluginConfig })
    |
    v  managers (tmux, background, skillMcp, configHandler)
Step 3: createTools({ ctx, pluginConfig, managers })
    |
    v  filteredTools, mergedSkills, availableSkills, availableCategories
Step 4: createHooks({ ctx, pluginConfig, managers, mergedSkills, availableSkills })
    |
    v  hooks (48 hook instances)
Step 5: createPluginInterface({ ctx, pluginConfig, managers, hooks, tools })
    |
    v  PluginInterface (returned to OpenCode)
```

Let's walk through each step.

### Step 1: Load Configuration

**File**: `src/plugin-config.ts`

Reads and merges configuration from two levels:

```
User config:    ~/.config/opencode/oh-my-opencode.jsonc
Project config: .opencode/oh-my-opencode.jsonc
```

The merge strategy differs by field type:
- `agents`, `categories`: Deep merged (project overrides user per-field)
- `disabled_*` arrays: Set union (both levels are combined)
- Everything else: Project overrides user entirely

After merging, the config is validated with Zod schemas (`src/config/schema/`). Invalid fields get defaults. Legacy keys are auto-migrated (old agent names, old hook names, old model versions).

**Output**: `pluginConfig` - a fully validated configuration object.

### Step 2: Create Managers

**File**: `src/create-managers.ts`

Creates the long-lived service objects that multiple systems need:

| Manager | Purpose |
|---------|---------|
| `TmuxSessionManager` | Manages tmux panes for visual multi-agent display |
| `BackgroundManager` | Runs background agent tasks with per-provider concurrency limits |
| `SkillMcpManager` | Manages MCP servers embedded in skills (start, stop, lifecycle) |
| `ConfigHandler` | Handles dynamic config reloading |

These managers are passed to both tools and hooks. For example, the `task` tool needs `BackgroundManager` to launch background agents, and the `background-notification` hook needs it to detect completions.

### Step 3: Create Tools

**Files**: `src/create-tools.ts` and `src/plugin/tool-registry.ts`

This step builds all 26 tools the plugin provides to agents.

```
createTools()
  |
  +--> createSkillContext()          // Skill loading and resolution
  +--> createAvailableCategories()   // Category definitions for task delegation
  +--> createToolRegistry()          // Assembles all tools
         |
         +--> Built-in tools (read, write, edit, bash, etc.)
         +--> Search tools (grep, glob, ast-grep)
         +--> LSP tools (diagnostics, rename, references, symbols)
         +--> Session tools (list, read, search, info)
         +--> Background tools (output, cancel)
         +--> Delegation tools (task, call_omo_agent)
         +--> Skill tools (skill, skill_mcp)
         +--> Visual tools (look_at)
         |
         +--> normalizeToolArgSchemas()   // Standardize JSON schemas
         +--> filterDisabledTools()       // Remove disabled tools
         +--> trimToolsToCap()            // Enforce max tool count
```

Each tool is a `ToolDefinition` with a name, description, JSON schema for parameters, and an `execute` function. The `createToolRegistry` function in `src/plugin/tool-registry.ts` composes all of them into a single `ToolsRecord` object.

**Key output**: `filteredTools` (the final tool set), plus `mergedSkills`, `availableSkills`, and `availableCategories` which are needed by the hook and agent systems.

### Step 4: Create Hooks

**File**: `src/create-hooks.ts`

Hooks are the plugin's behavior modification layer. They intercept events (tool calls, messages, session lifecycle) and can modify inputs, outputs, or trigger side effects.

Hooks are composed in three tiers:

```
createHooks()
  |
  +--> createCoreHooks()              // 39 hooks
  |      +--> createSessionHooks()    //   23 session-scoped hooks
  |      +--> createToolGuardHooks()  //   12 tool pre/post hooks
  |      +--> createTransformHooks()  //    4 message transform hooks
  |
  +--> createContinuationHooks()      //  7 continuation hooks
  |      (todo enforcer, atlas, ralph loop, etc.)
  |
  +--> createSkillHooks()             //  2 skill-aware hooks
         (category skill reminder, auto slash command)
```

Each hook is conditionally created based on `isHookEnabled(hookName)` and wrapped in `safeCreateHook` which catches errors during construction. A hook that fails to initialize becomes `null` instead of crashing the plugin.

The three tiers are spread-merged into a single flat hooks object. This is the `CreatedHooks` type used by the plugin handlers.

**Output**: `hooks` object with up to 48 hook instances (each is either a handler object or `null`).

### Step 5: Create Plugin Interface

**File**: `src/plugin-interface.ts`

Wires everything together into the 8 handlers OpenCode expects:

```typescript
return {
  config:     configHandler,                    // Agent and MCP registration
  tool:       filteredTools,                    // 26 tools
  "chat.message":           chatMessageHandler, // Message interception
  "chat.params":            chatParamsHandler,  // API parameter adjustment
  "chat.headers":           chatHeadersHandler, // HTTP header injection
  event:                    eventHandler,        // Session lifecycle events
  "tool.execute.before":    toolExecBefore,     // Pre-tool hooks
  "tool.execute.after":     toolExecAfter,      // Post-tool hooks
  "experimental.chat.messages.transform": messagesTransform  // Context injection
}
```

This object is what OpenCode receives. From this point on, OpenCode calls these handlers at the appropriate lifecycle points.

---

## The 8 OpenCode Hook Handlers

These are the touchpoints between OpenCode and the plugin:

### `config`

Called once during plugin load. Registers:
- **Agents**: Built-in agents (Sisyphus, Hephaestus, Oracle, etc.) with their models, prompts, and tool restrictions
- **MCPs**: Built-in MCP servers (websearch, context7, grep_app)
- **Commands**: Slash commands (/init-deep, /ralph-loop, /start-work, etc.)

This is implemented as a 6-phase pipeline in `src/plugin-handlers/`:
1. Provider detection (which AI providers are available)
2. Plugin component creation
3. Agent creation (calls `createBuiltinAgents`)
4. Tool creation
5. MCP creation
6. Command creation

### `tool`

The tool registry from Step 3. OpenCode uses this to know what tools are available to agents.

### `chat.message`

**File**: `src/plugin/chat-message.ts`

Intercepts every message in the conversation. Hooks are called in sequence:

```
modelFallback -> stopContinuationGuard -> backgroundNotification ->
runtimeFallback -> keywordDetector -> thinkMode -> claudeCodeHooks ->
autoSlashCommand -> noSisyphusGpt -> noHephaestusNonGpt -> startWork
```

This is where the keyword detector activates modes (`ultrawork`, `search`), where model fallback happens on errors, and where slash commands are auto-executed.

### `chat.params`

**File**: `src/plugin/chat-params.ts`

Modifies API request parameters before they're sent to the AI provider. Currently used for:
- Anthropic effort level adjustment (dynamically setting reasoning effort)

### `chat.headers`

**File**: `src/plugin/chat-headers.ts`

Injects HTTP headers into API requests. Used for Copilot integration (`x-initiator` header).

### `event`

**File**: `src/plugin/event.ts`

Handles session lifecycle events:
- `session.created`: Version checks, toast notifications, model cache refresh
- `session.deleted`: Cleanup
- `session.idle`: Todo continuation, Ralph loop, session notifications
- `session.error`: Session recovery, context window limit recovery, runtime fallback

This is the most complex handler because it dispatches to many hooks based on event type.

### `tool.execute.before`

**File**: `src/plugin/tool-execute-before.ts`

Runs before every tool call. Hooks execute in order:

```
writeExistingFileGuard -> questionLabelTruncator -> claudeCodeHooks ->
nonInteractiveEnv -> commentChecker -> directoryAgentsInjector ->
directoryReadmeInjector -> rulesInjector -> tasksTodowriteDisabler ->
prometheusMdOnly -> sisyphusJuniorNotepad -> atlasHook
```

Hooks can modify tool inputs (e.g., inject AGENTS.md content into a read call) or block tool execution entirely (e.g., prevent writes to files that haven't been read first).

### `tool.execute.after`

**File**: `src/plugin/tool-execute-after.ts`

Runs after every tool call. Hooks execute in order:

```
toolOutputTruncator -> claudeCodeHooks -> preemptiveCompaction ->
contextWindowMonitor -> commentChecker -> directoryAgentsInjector ->
directoryReadmeInjector -> rulesInjector -> emptyTaskResponseDetector ->
agentUsageReminder -> categorySkillReminder -> interactiveBashSession ->
editErrorRecovery -> delegateTaskRetry -> atlasHook -> taskResumeInfo ->
hashlineReadEnhancer -> jsonErrorRecovery
```

Hooks can modify tool outputs (e.g., truncate oversized grep results), add system messages (e.g., remind the agent about available skills), or trigger recovery flows (e.g., retry a failed delegation).

### `experimental.chat.messages.transform`

**File**: `src/plugin/messages-transform.ts`

Transforms the full message array before it's sent to the model. Used for:
- Context injection (AGENTS.md, rules, project context)
- Thinking block validation (ensures thinking blocks comply with API requirements)

---

## Data Flow Diagram

Here's how data flows through the system during a typical interaction:

```
User types a message
  |
  v
chat.message handler
  - keywordDetector checks for "ultrawork", "search", etc.
  - thinkMode detects thinking keywords
  |
  v
messages.transform handler
  - contextInjector adds AGENTS.md, rules
  - thinkingBlockValidator fixes block format
  |
  v
chat.params handler
  - anthropicEffort adjusts reasoning level
  |
  v
[Message sent to AI model]
  |
  v
Model responds with tool calls
  |
  v
For each tool call:
  |
  +--> tool.execute.before handler
  |      - writeExistingFileGuard blocks blind overwrites
  |      - rulesInjector adds matching rules
  |
  +--> Tool executes (e.g., read file, run bash, delegate task)
  |
  +--> tool.execute.after handler
         - toolOutputTruncator manages output size
         - commentChecker flags AI-generated comments
         - agentUsageReminder suggests better approaches
  |
  v
[Results returned to model]
  |
  v
Model continues or finishes
  |
  v
event handler (session.idle)
  - todoContinuationEnforcer checks for incomplete todos
  - ralphLoop decides whether to continue
```

---

## How Agents Are Registered

Agents are created in the `config` handler's agent creation phase. The flow:

```
createBuiltinAgents()
  |
  +--> For each agent (sisyphus, oracle, explore, ...):
  |      1. Check if disabled (disabled_agents config)
  |      2. Apply user overrides (model, temperature, prompt_append)
  |      3. Resolve model (4-step: override -> category -> fallback -> default)
  |      4. Call agent factory: createXXXAgent(resolvedModel) -> AgentConfig
  |
  +--> Discover OpenCode-native agents (from opencode.json)
  |      - Parse customAgentSummaries
  |      - Add to availableAgents (so Sisyphus knows about them)
  |
  +--> Build Sisyphus's dynamic prompt
  |      - Inject available agents, skills, categories into prompt
  |
  +--> Return Record<string, AgentConfig>
```

The critical detail: oh-my-opencode's agents and OpenCode's native agents coexist. Custom agents defined in `opencode.json` or `.opencode/agents/*.md` are discovered and added to the available agents list. Sisyphus can delegate to them.

---

## How Tools Are Invoked

When an agent uses the `task` tool to delegate work, the flow is:

```
Agent calls task(category="visual-engineering", prompt="...", load_skills=["frontend-ui-ux"])
  |
  v
createDelegateTask.execute()  (src/tools/delegate-task/tools.ts)
  |
  +--> resolveSkillContent(["frontend-ui-ux"])     // Load skill prompts
  +--> resolveParentContext()                       // Get parent model/context
  |
  v
resolveCategoryExecution()  (category-resolver.ts)
  |
  +--> mergeCategories(userCategories)             // Combine built-in + user categories
  +--> resolveModelForDelegateTask()               // Find best available model
  |      - Check user override -> category default -> fallback chain -> system default
  |      - Fuzzy match against connected provider models
  +--> Build fallback chain
  |
  v
buildSystemContent()  (prompt-builder.ts)
  |
  +--> Prepend skill content
  +--> Append category-specific prompt
  +--> Apply token limits
  |
  v
executeBackgroundTask() or executeSyncTask()  (executor.ts)
  |
  +--> Create new OpenCode session
  +--> Set agent to Sisyphus-Junior (configured for the category)
  +--> Send prompt
  +--> Return task_id (async) or result (sync)
```

---

## Key Design Decisions

### Why a Plugin, Not a Fork

oh-my-opencode is a plugin, not a fork of OpenCode. This means:

1. **Upgrades are free**: When OpenCode updates, the plugin works with the new version without merge conflicts.
2. **Clear boundaries**: The plugin can only do what the hook interface allows. This prevents fragile internal coupling.
3. **Composable**: Users can run other plugins alongside oh-my-opencode.

The tradeoff is that some features require creative workarounds (e.g., using `experimental.chat.messages.transform` for context injection instead of directly modifying the message pipeline).

### Why Dynamic Agent Registration

Agents aren't hardcoded in a config file. They're created dynamically based on:
- Which AI providers are actually connected
- Which models are available
- What the user has overridden in their config

This means the agent list adapts to each user's environment. If you don't have an OpenAI key, Oracle falls through its fallback chain to a model you do have.

### Why 48 Hooks

Each hook has a single, narrow responsibility. This follows the same philosophy as the agent system: specialization beats generalization. A `comment-checker` hook does one thing: detect AI-generated comments. A `tool-output-truncator` does one thing: manage output sizes.

The cost is complexity in the composition layer. But the benefit is that each hook can be independently enabled, disabled, tested, and understood.

### Why Three Hook Tiers

The three tiers serve different purposes:

| Tier | Purpose | Depends On |
|------|---------|------------|
| **Core** | Fundamental behavior (quality, safety, recovery) | Only plugin config |
| **Continuation** | Session persistence (todos, loops, atlas) | Core hooks (e.g., session recovery) |
| **Skill** | Skill-aware features (category reminders, auto commands) | Discovered skills |

They're separated because they have different dependency requirements. Continuation hooks need the session recovery hook from the core tier. Skill hooks need the discovered skills from the tool creation step.

---

## Source File Map

| File | Purpose |
|------|---------|
| `src/index.ts` | Plugin entry point, 5-step init |
| `src/plugin-config.ts` | Config loading, merging, validation |
| `src/create-managers.ts` | Manager object creation |
| `src/create-tools.ts` | Tool creation orchestration |
| `src/create-hooks.ts` | 3-tier hook composition |
| `src/plugin-interface.ts` | PluginInterface assembly |
| `src/plugin/tool-registry.ts` | Tool registration and filtering |
| `src/plugin/tool-execute-before.ts` | PreToolUse dispatch |
| `src/plugin/tool-execute-after.ts` | PostToolUse dispatch |
| `src/plugin/chat-message.ts` | Message interception |
| `src/plugin/chat-params.ts` | API parameter modification |
| `src/plugin/event.ts` | Session lifecycle dispatch |
| `src/plugin/messages-transform.ts` | Context transformation |
| `src/plugin-handlers/` | 6-phase config handler |
| `src/config/schema/` | Zod validation schemas |

---

## Key Takeaways

1. **The plugin is a pipeline**: Config -> Managers -> Tools -> Hooks -> Interface. Each step's output feeds the next.

2. **8 hook handlers are the entire surface area** between the plugin and OpenCode. Everything the plugin does must flow through one of these 8 handlers.

3. **Hooks are composed in 3 tiers** (Core, Continuation, Skill) and dispatched in deterministic order within each handler.

4. **Agent registration is dynamic**: The available agents adapt to your provider connections, config overrides, and even OpenCode-native agent definitions.

5. **Tool delegation is the key orchestration mechanism**: The `task` tool resolves categories to models, loads skills, builds prompts, and dispatches to executors (background or sync).

---

Next: [03 - The Agent System](./03-agents.md)
