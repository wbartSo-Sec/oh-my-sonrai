# Extending oh-my-opencode

Extending the platform allows you to tailor the AI's behavior, add new specialized capabilities, and integrate with external systems. You can extend oh-my-opencode through two primary paths: OpenCode-native extensions and plugin-level modifications.

## 1. Overview: Two Paths to Extend

Most users should stick to the OpenCode-native path. It allows you to add features without modifying the oh-my-opencode source code or maintaining a fork.

### OpenCode-native (Recommended)
This method uses configuration files and markdown templates to define new behaviors. It is portable, survives plugin updates, and requires no TypeScript knowledge. You can define agents, skills, commands, and categories by placing files in specific directories in your project or home folder.

### Plugin-level (Contributors)
This path involves modifying the oh-my-opencode repository itself. Use this if you want to contribute a feature to the core project, add a complex tool that requires custom logic, or implement a low-level lifecycle hook. It requires working with the TypeScript source and following the established factory patterns.

---

## 2. Creating Custom Agents (OpenCode-native)

Custom agents allow you to define specialized personas with unique system prompts, model preferences, and permission settings. Sisyphus automatically discovers these agents and adds them to its delegation table.

### Markdown File Method
Place .md files in these locations:
- Global: `~/.config/opencode/agents/`
- Per-project: `.opencode/agents/`

Example: `security-auditor.md`

```markdown
---
description: Expert security auditor for TypeScript and Go
mode: subagent
model: claude-3-5-sonnet-latest
temperature: 0.2
permissions:
  allow_file_read: true
  allow_bash: true
  allow_network: false
---

You are a senior security researcher. Your goal is to find vulnerabilities in the provided code.
Focus on:
1. SQL injection in database queries.
2. Improper handling of secrets.
3. Race conditions in concurrent Go code.
```

### JSON Config Method
You can also define agents directly in your `oh-my-opencode.jsonc` configuration file within the `agents` section.

```jsonc
{
  "agents": {
    "documentation-expert": {
      "description": "Writes clear, concise technical documentation",
      "model": "claude-3-5-sonnet-latest",
      "system_prompt": "You are a technical writer specializing in developer documentation...",
      "mode": "subagent"
    }
  }
}
```

Key point: The plugin parses these custom definitions and generates summaries for the primary agent. This allows Sisyphus to know exactly when to delegate a task to your custom security-auditor or documentation-expert.

---

## 3. Creating Custom Skills

Skills provide a way to inject complex instructions, context, and even embedded MCP servers into an agent's session. They are perfect for teaching an agent how to use a specific internal library or follow a team's coding standard.

### Skill File Locations
The plugin searches for skills in the following priority order:
1. `.opencode/skills/` (Project-specific)
2. `~/.config/opencode/skills/` (User-specific)
3. `.claude/skills/` (Claude Code compatibility)
4. `.agents/skills/`
5. `~/.agents/skills/`

### SKILL.md Format
Each skill must live in its own directory with a `SKILL.md` file.

Example: `.opencode/skills/jira-integration/SKILL.md`

```markdown
---
name: jira-integration
description: Manage Jira tickets and sprints
mcp:
  name: jira-server
  command: bun
  args: ["run", "src/mcp/jira/index.ts"]
  env:
    JIRA_API_TOKEN: "${JIRA_TOKEN}"
---

# Jira Integration Skill

Use this skill when the user asks to create, update, or search for Jira tickets.
Always verify the ticket ID before making changes.
```

### Loading Skills
Agents load skills via the `load_skills` parameter in the `task()` tool call. When Sisyphus delegates a task, it specifies which skills the subagent should have access to.

---

## 4. Creating Custom Commands

Commands are shortcuts for common tasks, invoked with a forward slash (e.g., `/deploy`). They are essentially pre-defined prompts that can take arguments.

### Command File Locations
Place .md files in:
- Project: `.opencode/command/`
- User: `~/.config/opencode/command/`
- Compat: `.claude/commands/` or `~/.config/opencode/commands/`

Example: `.opencode/command/review-pr.md`

```markdown
# /review-pr [pr_number]

Analyze the changes in the specified pull request.
Check for:
1. Adherence to our style guide.
2. Missing unit tests.
3. Performance regressions.

Use the `gh pr diff [pr_number]` command to see the changes.
```

---

## 5. Creating Custom Categories

Categories group settings for specific types of work, such as "writing", "refactoring", or "debugging". They allow you to apply consistent model choices and prompt snippets across different tasks.

### Adding Categories
Define categories in your `oh-my-opencode.jsonc` file.

```jsonc
{
  "categories": {
    "korean-writer": {
      "model": "gpt-4o",
      "temperature": 0.7,
      "prompt_append": "Please provide all responses in professional Korean."
    }
  }
}
```

You can also override built-in categories like `visual-engineering` or `ultrabrain` to change their default models or behavior.

---

## 6. Disabling Built-in Features

Sometimes a built-in feature might conflict with your workflow or security requirements. You can disable any component using the `disabled_*` arrays in your configuration.

### Examples of Disabling Features
```jsonc
{
  "disabled_agents": ["hephaestus"], // Remove the coder agent
  "disabled_hooks": ["git-guard"], // Stop the plugin from checking git status before tools
  "disabled_tools": ["bash"], // Prevent the agent from running shell commands
  "disabled_skills": ["github-triage"] // Remove a specific built-in skill
}
```

This is useful in locked-down environments or when you want to replace a built-in agent with your own custom version.

---

## 7. Plugin-Level Extension (For Contributors)

If you are developing for the oh-my-opencode repository itself, follow these structural patterns.

### Adding a New Agent
1. Create a factory file in `src/agents/builtin-agents/`.
2. Follow the `createXXXAgent` pattern, implementing the `Agent` interface.
3. Register the new agent in `src/agents/builtin-agents/index.ts` within the `agentSources` object.

### Adding a New Hook
1. Create a directory in `src/hooks/{hook-name}/`.
2. Implement the hook logic (e.g., `onToolExecuteBefore`).
3. Register the hook in the relevant creation file (e.g., `src/plugin/hooks/create-session-hooks.ts`).

### Adding a New Tool
1. Create a directory in `src/tools/{tool-name}/`.
2. Define the tool's schema, logic, and output formatting.
3. Register the tool in `src/plugin/tool-registry.ts`.

### Adding a New Built-in Skill
1. Implement the `BuiltinSkill` interface in `src/features/builtin-skills/skills/`.
2. Skills added here are bundled with the plugin and available globally without external files.

### Engineering Conventions
- Use kebab-case for all filenames and directories.
- Every module must have an `index.ts` barrel export.
- Use the factory pattern (`createXXX`) for instantiation.
- Keep business logic out of `index.ts` files; they should only handle exports.

---

## 8. Key Takeaways

- Most extensions should be OpenCode-native to ensure portability and ease of maintenance.
- Custom agents and skills are the most powerful ways to specialize the AI for your specific domain.
- Sisyphus automatically integrates custom agents into its delegation logic.
- The `oh-my-opencode.jsonc` file is your central hub for overriding defaults and disabling unwanted features.
- Plugin-level changes are reserved for core features and follow strict factory and barrel export patterns.

Next: [08 - Patterns for Your Own Projects](./08-patterns.md)
