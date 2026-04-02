# Patterns for Your Own Projects

This guide extracts reusable patterns from oh-my-opencode that you can apply to your own multi-agent projects. These patterns help build systems that are more resilient, efficient, and easier to maintain, even if you are not using this specific plugin.

## 1. Pattern: Dynamic Prompt Assembly

### The Problem
Static prompts are brittle. They cannot adapt to available tools, the specific agents in the current session, or the project context. If you hardcode a tool list in a prompt and then add a new tool, you have to update the prompt manually.

### The Solution
Build prompts from composable sections at runtime. Instead of one giant string, treat your prompt as a collection of modules that are toggled on or off based on the current state.

### How oh-my-opencode Does It
The plugin uses an `AgentPromptMetadata` type that defines metadata for each agent, including:
- **Category**: What kind of work it does.
- **Cost**: Relative resource usage.
- **Triggers**: Keywords or conditions that activate the agent.
- **UseWhen/AvoidWhen**: Specific scenarios for delegation.

These metadata objects drive section builders like `buildDelegationTable()` and `buildToolSelectionTable()`. When an agent is initialized, the orchestrator gathers the relevant metadata and concatenates these strings into a cohesive system prompt.

### Extractable Pattern
Define metadata per agent or component. Write standalone functions (section builders) for each area of your prompt (e.g., identity, tool instructions, delegation rules). Compose the final prompt at creation time by calling these builders with the current runtime context.

### Your Version
Even without a complex plugin system, you can use a simple template-based approach in any language:

```typescript
const promptSections = [
  buildIdentity(agentType),
  buildToolContext(availableTools),
  buildProjectRules(currentPath),
  buildOutputFormat(taskComplexity)
];

const systemPrompt = promptSections.filter(Boolean).join('\n\n');
```

## 2. Pattern: Intent Classification Before Action

### The Problem
LLMs often default to "start coding" for every request. If a user asks a question about the architecture, the agent might start creating files instead of just explaining the design.

### The Solution
Classify intent first, then route to the right workflow. By forcing the agent to pause and categorize the request, you prevent "hallucinated actions."

### How oh-my-opencode Does It
The plugin uses a `keyword-detector` hook and Sisyphus's "Intent Gate" (Phase 0). Before performing any tools or complex logic, the agent must verbalize its understanding of the user's intent.

### Extractable Pattern
Before your agent acts, have it verbalize its classification in its internal reasoning or as a mandatory first step. Use a format like: "I detect [type] intent. My approach: [action]."

### Your Version
Add an intent classification step to your agent's system prompt:

"Before taking any action, you must explicitly state the intent of the user request. Choose from: EXPLORATION, IMPLEMENTATION, REFACTOR, or SUPPORT. Then, explain your planned approach based on that intent."

## 3. Pattern: Constrained Agents via Tool Restrictions

### The Problem
Agents with access to every tool tend to drift from their designated roles. An "Oracle" agent meant for code review might start fixing bugs if it has access to the `edit` tool, even if it wasn't asked to.

### The Solution
Enforce roles through tool deny-lists, not just instructions. Role-playing is not enough for reliable behavior; physical restrictions are required.

### How oh-my-opencode Does It
The system defines tool restrictions per agent. For example:
- **Oracle**: Cannot use `write` or `edit`.
- **Explore**: Cannot use `edit`.
- **Sisyphus-Junior**: Cannot use delegation tools.

These restrictions are enforced at the tool execution level. If an agent tries to call a restricted tool, the system intercepts and blocks it.

### Extractable Pattern
For each agent role, define what it CANNOT do. Enforce this at the tool registry level, not just in the prompt.

### Your Version
When creating OpenAI or Anthropic function-calling agents, only register the specific subset of functions that the agent should have access to.

```typescript
const agentTools = allTools.filter(tool => !denylist.includes(tool.name));
const response = await client.messages.create({
  tools: agentTools,
  // ...
});
```

## 4. Pattern: Tiered Model Routing

### The Problem
Using expensive models for every task (like simple file reading or grepping) wastes money. Using cheap models for everything (like complex architecture planning) leads to poor results.

### The Solution
Route tasks by complexity and domain to the appropriate model tier.

### How oh-my-opencode Does It
The plugin defines `Categories` such as `quick`, `ultrabrain`, and `visual-engineering`. Each category maps to an optimal model. A "quick" task might use a smaller, faster model, while an "ultrabrain" task uses the most powerful model available.

### Extractable Pattern
Define task categories (e.g., Simple, Medium, Complex). Assign specific models to each category. Let your orchestrator or user choose the category for the task.

### Your Version
Even with a single provider, route between tiers:

- **GPT-4o-mini**: Searching, reading files, simple greps.
- **GPT-4o**: Standard coding tasks, implementing functions.
- **o1-preview**: Complex refactoring, architecture planning, debugging difficult errors.

## 5. Pattern: Background Agent Execution

### The Problem
Sequential execution is slow. If you need to search for code patterns, check PR comments, and run a security scan, doing them one by one wastes the user's time.

### The Solution
Fire multiple agents simultaneously and collect their results when they are ready.

### How oh-my-opencode Does It
The `BackgroundManager` handles per-provider concurrency limits and tracks async task IDs. When an agent is spawned in the background, it receives a task ID. The system notifies the main agent when the background task is complete.

### Extractable Pattern
Launch independent work in parallel. Assign each background task a unique ID. Use a notification or polling mechanism to collect results before the main workflow continues.

### Your Version
Use async/await patterns with a simple task queue. In a Node.js script, you can run multiple LLM calls in parallel easily:

```typescript
const [searchResult, reviewResult] = await Promise.all([
  runSearchAgent(query),
  runReviewAgent(files)
]);
```

## 6. Pattern: Fallback Chains for Resilience

### The Problem
A single model or provider going down can break your entire system. Rate limits or transient API errors are common in LLM development.

### The Solution
Define ordered fallback chains for every agent. If the primary model fails, the system automatically tries the next one in the chain.

### How oh-my-opencode Does It
The system uses a 4-step model resolution: `override` -> `category` -> `fallback chain` -> `system default`. It performs fuzzy matching against available providers to find a working model.

### Extractable Pattern
For each agent, define a primary model and a list of N fallback models. Wrap your API calls in a retry loop that moves to the next model in the list on failure.

### Your Version
Create a simple wrapper function for your LLM calls:

```typescript
async function callWithFallback(prompt, modelChain) {
  for (const model of modelChain) {
    try {
      return await callLLM(prompt, model);
    } catch (e) {
      console.warn(`Model ${model} failed, trying next...`);
    }
  }
  throw new Error("All models in chain failed.");
}
```

## 7. Pattern: Skill Injection (Composable Context)

### The Problem
Giving every agent every tool and every piece of instruction wastes context window space and increases costs. It also makes it more likely the agent will get confused.

### The Solution
Load domain-specific knowledge and tools only when they are needed.

### How oh-my-opencode Does It
Skills are defined in `SKILL.md` files. They can include both text instructions and embedded MCP (Model Context Protocol) servers. When an agent is delegated a task, it can be passed a list of skills via `load_skills=[]`. The content of these skills is prepended as system content only for that specific subagent.

### Extractable Pattern
Define reusable instruction sets (snippets). Load them dynamically based on the task type.

### Your Version
Maintain a library of Markdown snippets. Prepend the relevant ones to your system prompt based on the detected intent or task.

```typescript
const systemPrompt = `
  ${baseInstructions}
  ${isTestingTask ? testingSkillSnippet : ''}
  ${isDatabaseTask ? dbSkillSnippet : ''}
`;
```

## 8. Pattern: The Hook/Middleware Pattern for Agent Behavior

### The Problem
Modifying agent behavior (adding quality checks, truncating long outputs, or handling errors) by changing the core logic makes the code difficult to maintain and test.

### The Solution
Use a middleware or hook system that intercepts events during the agent's lifecycle.

### How oh-my-opencode Does It
The plugin implements 48 hooks across 6 event types (e.g., `before-tool`, `after-tool`, `on-message`). Hooks are composed in tiers and dispatched in a deterministic order. This allows features like "file guards" (preventing edits to sensitive files) to be implemented as standalone modules.

### Extractable Pattern
Define lifecycle events for your agent. Register handlers for these events. Dispatch them in a consistent order during execution.

### Your Version
Wrap your tool execution logic in a simple hook runner:

```typescript
async function executeTool(toolName, args) {
  await runHooks('before-tool', { toolName, args });
  const result = await actualToolExecution(toolName, args);
  const finalResult = await runHooks('after-tool', { toolName, result });
  return finalResult;
}
```

## 9. Key Takeaways

| Pattern | Core Idea | Minimum Viable Version |
|---------|-----------|------------------------|
| Dynamic Prompt Assembly | Build prompts at runtime from modules | String concatenation of templates |
| Intent Classification | Categorize before acting | Ask the model to state its intent first |
| Constrained Agents | Use tool deny-lists to enforce roles | Register only specific functions per agent |
| Tiered Model Routing | Use the right model for the job | Map task types to different API models |
| Background Execution | Run independent tasks in parallel | Use `Promise.all` for multiple LLM calls |
| Fallback Chains | Primary + N fallback models | Loop through a list of models on failure |
| Skill Injection | Load context on-demand | Prepend instruction snippets based on task |
| Hook/Middleware | Intercept events to modify behavior | Wrapper functions around tool calls |

## Further Reading

This concludes the deep-dive series. For more information on how these patterns are implemented in oh-my-opencode, refer to the other docs in this series:

- [01-architecture.md](01-architecture.md): The high-level system design.
- [02-agents.md](02-agents.md): How individual agents are constructed.
- [03-tools.md](03-tools.md): The tool registry and execution engine.
- [04-hooks.md](04-hooks.md): Detailed look at the hook system.
- [05-skills.md](05-skills.md): How skills and MCPs work together.
- [06-config.md](06-config.md): The multi-level configuration system.
- [07-background-tasks.md](07-background-tasks.md): Orchestrating parallel agent execution.
