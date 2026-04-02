# The Hook System

Hooks are the central nervous system of oh-my-opencode. They provide a standardized mechanism to intercept, modify, and augment the communication between the AI model and the local environment. By decoupling logic into discrete, event-driven units, the system can enforce safety guards, inject context-specific rules, and recover from common tool errors without bloating the core plugin logic.

## OpenCode Event Types

The plugin intercepts six primary event types from the OpenCode host. Each hook can implement one or more of these interfaces to react to different stages of the execution lifecycle.

| Event Type | OpenCode Hook | Description |
|------------|---------------|-------------|
| PreToolUse | `tool.execute.before` | Triggered before a tool is executed. Used for safety guards and context injection. |
| PostToolUse | `tool.execute.after` | Triggered after a tool finishes. Used for output cleaning, error recovery, and state updates. |
| Message | `chat.message` | Triggered when a user sends a message. Used for keyword detection and mode switching. |
| Event | `event` | Lifecycle events like session creation, deletion, or errors. Used for cleanup. |
| Transform | `experimental.chat.messages.transform` | Modifies the full message history before it is sent to the model. Used for prompt assembly. |
| Params | `chat.params` | Adjusts model parameters like temperature or effort levels. |

## 3-Tier Architecture

The hook system is organized into three tiers based on their functional domain and dependency requirements. This separation ensures that core system stability does not depend on higher-level features like task delegation or skill loading.

1. **Core (39 hooks)**: Essential logic for safety, truncation, and basic environment management. These hooks handle file guards, output limits, and standard tool interactions.
2. **Continuation (7 hooks)**: Logic dedicated to maintaining long-running tasks across multiple model turns. This includes the task status monitor and resume information hooks.
3. **Skill (2 hooks)**: Hooks that facilitate the loading and execution of specific skills, ensuring that skill-defined MCPs and tools are correctly integrated into the session.

## Hook Factory Pattern

Each hook is defined using a factory function. This pattern allows for dependency injection (such as configuration or shared state) and keeps hook logic isolated and testable. A factory returns an object where keys match the OpenCode event types.

```typescript
// Pseudocode example of a hook factory
export function createSafetyGuardHook(deps: HookDependencies) {
  return {
    "tool.execute.before": async (input: ToolInput, context: HookContext) => {
      if (input.toolName === "bash" && input.arguments.includes("rm -rf /")) {
        throw new Error("Destructive command blocked by SafetyGuard");
      }
    },
    "tool.execute.after": async (output: ToolOutput) => {
      // Logic to run after tool execution
    }
  };
}
```

## The Gating System

To maintain performance and allow user customization, every hook passes through a gating system. The `isHookEnabled(hookName)` function checks the user and project configuration to determine if a hook should be active. 

The `safeCreateHook(hookName, factory, options)` utility wraps the construction process. If a hook is disabled, it returns `null`. If the factory throws an error during initialization (e.g., due to missing dependencies), the error is caught, logged, and the hook is skipped. This prevents a single faulty hook from crashing the entire plugin.

## Hook Dispatching

The plugin handlers in `src/plugin/*.ts` manage the execution of these hooks. Hooks are dispatched in a deterministic order to ensure predictable behavior (e.g., a file guard must run before a tool modifies a file).

Execution uses optional chaining to handle disabled (null) hooks gracefully:

```typescript
// Deterministic dispatch order in tool.execute.before handler
await hooks.writeExistingFileGuard?.["tool.execute.before"]?.(input, context);
await hooks.questionLabelTruncator?.["tool.execute.before"]?.(input, context);
await hooks.commentChecker?.["tool.execute.before"]?.(input, context);
```

The `?.` chain ensures that if `hooks.commentChecker` is null (disabled), the call is skipped without error.

## Concrete Example: Comment Checker

The Comment Checker hook enforces team standards by detecting and flagging AI-generated comments in code edits.

*   **Factory**: `createCommentCheckerHooks(config)` initializes the hook with specific forbidden patterns.
*   **On tool.execute.before**: It registers a pending call for the current tool execution, correlating the specific file being edited with the upcoming output.
*   **On tool.execute.after**: It scans the tool output (e.g., from a file write or edit) for common AI markers like "As an AI language model..." or redundant "I've updated the file" comments. If detected, it can modify the output to strip these comments or warn the user.

## Concrete Example: Keyword Detector

The Keyword Detector hook enables mode-switching based on the content of user messages.

*   **Factory**: `createKeywordDetectorHook(ctx)` receives the plugin context to access shared state.
*   **On chat.message**: It extracts the text from the incoming user message. It searches for specific activation keywords such as `ultrawork`, `search`, or `analyze`. 
*   **Action**: When a keyword is found, the hook injects mode-specific system instructions into the user message parts. For example, triggering `ultrawork` might append instructions that prioritize speed and atomic commits for that specific turn.

## Hook Execution Order

The order of execution is critical for maintaining system invariants. Below are the real-world sequences for the primary event types.

### tool.execute.before
1.  **writeExistingFileGuard**: Prevents accidental overwrites of critical files.
2.  **questionLabelTruncator**: Cleans up internal metadata from labels.
3.  **claudeCodeHooks**: Integrates default Claude Code behaviors.
4.  **nonInteractiveEnv**: Ensures tools don't hang waiting for user input.
5.  **commentChecker**: Prepares for code quality verification.
6.  **directoryAgentsInjector**: Provides context about local agent availability.
7.  **directoryReadmeInjector**: Injects README content for directory awareness.
8.  **rulesInjector**: Applies repository-specific linting and style rules.
9.  **tasksTodowriteDisabler**: Manages task tool availability.
10. **prometheusMdOnly**: Restricts planning tools to markdown output.
11. **sisyphusJuniorNotepad**: Synchronizes the executor's internal state.
12. **atlasHook**: Final context check before execution.

### tool.execute.after
1.  **toolOutputTruncator**: Prevents context window overflow from large tool results.
2.  **claudeCodeHooks**: Standard post-execution cleanup.
3.  **preemptiveCompaction**: Triggers memory management if the context is tight.
4.  **contextWindowMonitor**: Reports token usage and remaining budget.
5.  **commentChecker**: Scans and cleans tool output for AI-isms.
6.  **directoryAgentsInjector**: Updates directory state after changes.
7.  **rulesInjector**: Verifies that tool output adheres to rules.
8.  **emptyTaskResponseDetector**: Catches tools that return no data unexpectedly.
9.  **agentUsageReminder**: Suggests delegating tasks if the model is struggling.
10. **categorySkillReminder**: Reminds the model of available specialized skills.
11. **interactiveBashSession**: Manages state for persistent tmux sessions.
12. **editErrorRecovery**: Attempts to fix common `edit` tool failures.
13. **delegateTaskRetry**: Automatically retries failed subagent calls.
14. **atlasHook**: Final state synchronization.
15. **taskResumeInfo**: Persists progress data for session resumes.
16. **hashlineReadEnhancer**: Formats file read output for better parsing.
17. **jsonErrorRecovery**: Fixes malformed JSON returned by structured tools.

### chat.message
1.  **modelFallback**: Switches providers if the primary model is unavailable.
2.  **stopContinuationGuard**: Intercepts "stop" commands to kill background loops.
3.  **backgroundNotification**: Alerts the user when background tasks complete.
4.  **runtimeFallback**: Final safety check for the execution environment.
5.  **keywordDetector**: Analyzes message for mode-activation keywords.
6.  **thinkMode**: Activates internal reasoning blocks if requested.
7.  **claudeCodeHooks**: Default message handling logic.
8.  **autoSlashCommand**: Resolves shorthand commands (e.g., `/fix`).
9.  **noSisyphusGpt**: Enforces model restrictions for specific agents.
10. **noHephaestusNonGpt**: Ensures builder agents use appropriate models.
11. **startWork**: Triggers the initial task plan for new sessions.

## Hook Categories by Purpose

| Category | Description | Key Hooks |
|----------|-------------|-----------|
| Context/Injection | Adding repo-specific data to prompts | rulesInjector, directoryReadmeInjector |
| Productivity/Control | Managing how the AI works | tasksTodowriteDisabler, nonInteractiveEnv |
| Quality/Safety | Enforcing standards and safety | writeExistingFileGuard, commentChecker |
| Recovery/Stability | Fixing errors and retrying | editErrorRecovery, jsonErrorRecovery |
| Truncation | Managing context length | toolOutputTruncator, questionLabelTruncator |
| Notifications | Communicating with the user | backgroundNotification, agentUsageReminder |
| Task Management | Tracking progress | sisyphusJuniorNotepad, emptyTaskResponseDetector |
| Continuation | Multi-turn task persistence | taskResumeInfo, stopContinuationGuard |
| Integration | Claude Code compatibility | claudeCodeHooks |
| Specialized | Domain-specific logic | atlasHook, prometheusMdOnly |

## Disabling Hooks

Users can disable any hook through the `disabled_hooks` array in the configuration file. This is useful for troubleshooting or when a specific guard interferes with a specialized workflow.

```json
{
  "disabled_hooks": [
    "commentChecker",
    "directoryAgentsInjector"
  ]
}
```

When a hook is disabled, the factory is never called, and its slot in the hook registry remains `null`. The deterministic execution order is preserved, but the disabled hook's logic is simply skipped during the dispatch phase.

## Key Takeaways

*   The hook system decouples complex logic from the core plugin interface using a factory pattern.
*   Deterministic execution order ensures that safety guards and context injection happen in a predictable sequence.
*   The gating system provides stability by catching initialization errors and allowing granular control via configuration.
*   Hooks transform raw tool outputs and user messages into structured, context-aware communication.

## Source File Map

| Component | Path |
|-----------|------|
| Hook Entry Point | `src/plugin/hooks/create-hooks.ts` |
| Factory Definitions | `src/hooks/` |
| Gating Logic | `src/plugin/hooks/gate-hooks.ts` |
| Execution Order | `src/plugin/hooks/create-*-hooks.ts` |
| Type Definitions | `src/plugin/hooks/types.ts` |

Next: [06 - Task Delegation](./06-delegation.md)
