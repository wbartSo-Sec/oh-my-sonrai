# Context Management

How oh-my-opencode controls what goes into the agent's context window, keeps it from overflowing, and recovers when it does. This covers every injection, truncation, and recovery mechanism in the system.

---

## The Context Window Problem

An LLM has a fixed context window. Everything the agent "sees" must fit: the system prompt, conversation history, tool results, injected rules, AGENTS.md files, and more. oh-my-opencode manages this context through three layers:

1. **Injection**: Getting the right information in (AGENTS.md, rules, README.md, environment context)
2. **Truncation**: Keeping large outputs from consuming too much space (tool output limits, dynamic truncation)
3. **Recovery**: Handling overflow when it happens (compaction, pruning, multi-strategy recovery)

```
Context Window Budget
+--------------------------------------------------+
| System prompt (agent instructions)                |
| Injected context (AGENTS.md, rules, README)       |
| Conversation history                              |
| Tool results (potentially huge)                   |
| Agent's response space                            |
+--------------------------------------------------+
        ^                              ^
        |                              |
   Injection hooks              Truncation hooks
   add context here             shrink content here
```

---

## Layer 1: Context Injection

### How AGENTS.md Gets Injected

AGENTS.md files provide directory-level context to agents. There are two mechanisms, depending on your OpenCode version:

#### Mechanism A: Native OpenCode Injection (OpenCode >= 1.1.37)

OpenCode 1.1.37+ has built-in AGENTS.md support. It loads the project root AGENTS.md automatically and discovers subdirectory AGENTS.md files as the agent explores the codebase. When this native support is detected, oh-my-opencode's `directoryAgentsInjector` hook is **auto-disabled** to avoid duplicate injection.

The version check happens in `create-tool-guard-hooks.ts`:
```
getOpenCodeVersion() -> isOpenCodeVersionAtLeast("1.1.37")
  -> If true: directoryAgentsInjector = null (auto-disabled)
  -> If false: create the hook
```

#### Mechanism B: Plugin Injection (OpenCode < 1.1.37)

For older OpenCode versions, the `directoryAgentsInjector` hook handles it:

**When it runs**: After every `read` tool call (tool.execute.after).

**How it finds AGENTS.md files**:

```
Agent reads: src/components/Button.tsx
  |
  v
findAgentsMdUp({ startDir: "src/components/", rootDir: ctx.directory })
  |
  Walk upward:
    src/components/AGENTS.md  -> found? push
    src/AGENTS.md             -> found? push
    <rootDir>/AGENTS.md       -> SKIP (OpenCode loads this one natively)
  |
  v
  Return found paths in top-down order (src/AGENTS.md first, then src/components/AGENTS.md)
```

Key behaviors:
- **Root AGENTS.md is always skipped** by the plugin because OpenCode's own `system.ts` already loads it. This prevents duplication.
- **Walks upward** from the file's directory to `ctx.directory` (the plugin root). Never climbs past `ctx.directory`.
- **Injection order is top-down**: broader context (closer to root) is injected first, then more specific subdirectory context.

**How content is injected**: Appended to the read tool's output string:

```
[original read output for Button.tsx]

[Directory Context: /project/src/AGENTS.md]
# src Module Guide
This directory contains React components...

[Directory Context: /project/src/components/AGENTS.md]
# Component Conventions
All components use Tailwind CSS...
```

**Deduplication**: Per-session. Once an AGENTS.md from a directory has been injected, it won't be injected again for that session. The injected directories are tracked in a `Set<string>` persisted to disk as JSON. Cleared on `session.deleted` and `session.compacted` events (so re-injection happens after compaction).

**Truncation**: Content is passed through `dynamicTruncator` before injection. If the AGENTS.md is too large for the remaining context budget, it's truncated with a notice:
```
[Note: Content was truncated to save context window space. For full context, please read the file directly: src/AGENTS.md]
```

---

### Your Question: Parent Directory of Multiple Repos

> If the home directory is not a repo, but a parent directory of multiple repos, how does the AGENTS.md files get read or injected?

**Short answer**: AGENTS.md files inside child repos are discovered and injected. The parent directory's AGENTS.md (if any) is skipped if it's the root directory. The walker never leaves `ctx.directory`.

**Detailed behavior**:

```
ctx.directory = /home/user/projects/    (parent of multiple repos)

Agent reads: /home/user/projects/repoA/src/utils/helper.ts

findAgentsMdUp walks:
  /home/user/projects/repoA/src/utils/  -> check AGENTS.md
  /home/user/projects/repoA/src/        -> check AGENTS.md
  /home/user/projects/repoA/            -> check AGENTS.md  (child repo root)
  /home/user/projects/                  -> SKIP (this is ctx.directory / rootDir)
  [stop]

Result: Any AGENTS.md found inside repoA is injected.
        The parent directory's AGENTS.md is NOT injected (rootDir skip).
```

The guard `!parent.startsWith(rootDir)` prevents climbing outside the plugin's root directory. So if you read a file from repoA, you get repoA's AGENTS.md hierarchy. You don't get repoB's AGENTS.md files. And you don't get the parent's AGENTS.md (because that's the rootDir).

If OpenCode >= 1.1.37, this hook is disabled entirely and OpenCode handles AGENTS.md discovery natively.

---

### How README.md Gets Injected

The `directoryReadmeInjector` hook works identically to the AGENTS.md injector with two differences:

1. **Root README.md IS included** (not skipped). There's no native OpenCode README injection to conflict with.
2. **The hook is never auto-disabled** (no version check).

**When it runs**: After every `read` tool call.

**Injection format**:
```
[Project README: /project/README.md]
# My Project
This is a TypeScript library for...
```

Same deduplication and truncation behavior as the AGENTS.md injector.

---

### How Rules Get Injected

The `rulesInjector` hook is the most sophisticated injection mechanism. It discovers rule files from multiple sources, matches them against the current file path using glob patterns, deduplicates by content hash and real path, and injects matching rules into tool output.

**When it runs**: After `read`, `write`, `edit`, and `multiedit` tool calls (tool.execute.after).

**Rule file discovery**:

The hook searches for rule files in this order:

```
1. Walk upward from file's directory to project root, checking in each directory:
   - .github/instructions/*.instructions.md
   - .cursor/rules/*.md
   - .claude/rules/*.md
   - .sisyphus/rules/*.md

2. Single-file project rules at project root:
   - .github/copilot-instructions.md

3. User-level global rules:
   - ~/.sisyphus/rules/*.md
   - ~/.opencode/rules/*.md
   - ~/.claude/rules/*.md   (unless skipClaudeUserRules is set)
```

Allowed file extensions: `.md` and `.mdc`.

**Project root detection**: The hook uses `findProjectRoot()` which walks upward looking for project markers (`.git`, `package.json`, `Cargo.toml`, etc.). This is how it knows where the project boundary is.

**How rule matching works**:

Rule files can have YAML frontmatter that controls when they apply:

```markdown
---
description: TypeScript coding standards
globs: "src/**/*.ts"
alwaysApply: false
---

# TypeScript Standards
Always use strict mode...
```

The matching logic in `shouldApplyRule()`:

```
1. Does the rule have alwaysApply: true?
   YES -> Apply it. Always.

2. Does the rule have globs, paths, or applyTo in frontmatter?
   YES -> Compute file path relative to project root
       -> Use picomatch to test each glob pattern
       -> If any pattern matches -> Apply with reason "glob: <pattern>"

3. No frontmatter or no matching globs?
   -> Don't apply this rule.

Exception: Single-file project rules (e.g., .github/copilot-instructions.md)
   -> Always applied regardless of frontmatter.
```

The frontmatter parser supports multiple formats:
- `globs: "src/**/*.ts"` (single string)
- `globs: ["src/**/*.ts", "lib/**/*.ts"]` (array)
- `paths: "src/**"` (Claude Code alias for globs)
- `applyTo: "**/*.go"` (Copilot alias for globs)

**Deduplication** (two levels):

1. **By real path** (symlink-aware): Prevents injecting the same filesystem object twice per session.
2. **By content hash**: Prevents injecting identical content from different files per session.

Both levels are tracked in a per-session persistent cache, same pattern as the directory injectors.

**Injection format**:

```
[original tool output]

[Rule: .sisyphus/rules/typescript-standards.md]
[Match: glob: src/**/*.ts]
# TypeScript Standards
Always use strict mode...
```

**Proximity sorting**: Rules are sorted by "distance" (how many directory levels separate the rule file from the target file). Closer rules are injected first. Global user rules have distance 9999 (injected last).

---

### The Context Collector and Messages Transform

The directory injectors and rules injector don't just append to tool output. They also register content with the **ContextCollector** (`src/features/context-injector/collector.ts`), which is the centralized system for injecting context into the message array.

The flow:

```
Hooks discover context (AGENTS.md, README, rules)
  |
  v
Register with ContextCollector (keyed by source:id, with priority)
  |
  v
experimental.chat.messages.transform fires
  |
  v
contextInjectorMessagesTransform:
  1. Get pending contexts from collector
  2. Sort by priority + registration order
  3. Merge content with "\n\n---\n\n" separators
  4. Create a synthetic text part (synthetic: true)
  5. Insert into the last user message's parts array
     (before the real user text, so model sees it as context)
  |
  v
Model receives: [system prompt] + [user message with synthetic context parts] + [history]
```

The synthetic parts are marked `synthetic: true` so they're hidden from the UI but included in the model input. This is how AGENTS.md content appears in the agent's context without cluttering the conversation display.

---

### Environment Context

A small but always-present injection. `applyEnvironmentContext()` appends to every agent's system prompt:

```xml
<omo-env>
  Timezone: America/Halifax
  Locale: en-US
</omo-env>
```

This gives agents awareness of the user's timezone and locale. Disable with `experimental.disable_omo_env: true`.

---

## Layer 2: Context Truncation

### Tool Output Truncator

**File**: `src/hooks/tool-output-truncator.ts`

Runs on `tool.execute.after` for whitelisted tools. When a tool produces output larger than the budget, it's truncated.

**Which tools are truncated**:
- Default whitelist: `read`, `grep`, `glob`, `bash`, `webfetch`, and other tools that commonly produce large output
- With `experimental.truncate_all_tool_outputs: true`: ALL tools are truncated

**How truncation works**:
1. The `dynamicTruncator` checks current context usage via session message tokens
2. Computes remaining context budget
3. Applies tool-specific max token limits (e.g., `webfetch` has a lower limit)
4. If output exceeds budget, truncates and replaces `output.output`

The truncator is smart about context pressure: on a 200K token model with 60% used, it allows larger outputs than on a model with 90% used.

### Context Window Monitor

**File**: `src/hooks/context-window-monitor.ts`

Tracks token usage per session. When usage crosses 70%, injects a system reminder into tool outputs:

```
[SYSTEM DIRECTIVE: OH-MY-OPENCODE - CONTEXT WINDOW MONITOR]
You are using a 200,000-token context window.
You still have context remaining - do NOT rush or skip tasks.
Complete your work thoroughly and methodically.
[Context Status: 70.0% used (140,000/200,000 tokens), 30.0% remaining]
```

This nudges the agent to be context-aware without panicking. The reminder only fires once per session to avoid noise.

### Preemptive Compaction

**File**: `src/hooks/preemptive-compaction.ts`

Monitors context usage and triggers compaction (conversation summarization) proactively before hitting hard limits:

```
Token usage approaches model-specific threshold
  |
  v
Trigger session.compact (OpenCode's built-in compaction)
  |
  v
Context is summarized, freeing space
  |
  v
If compaction leads to degraded behavior (no-text tails, failures):
  degradation-monitor triggers recovery or backoff
```

### Dynamic Context Pruning (Experimental)

When `experimental.dynamic_context_pruning.enabled: true`, the system can prune old tool outputs to reclaim context:

**Strategies**:
| Strategy | What It Does |
|----------|-------------|
| `deduplication` | Removes duplicate tool calls (same tool, same args, same result) |
| `supersede_writes` | Prunes write/edit inputs when the file was later read (the read supersedes the write) |
| `purge_errors` | Removes errored tool inputs after N turns |

**Safety guards**:
- `turn_protection.turns: 3` - Recent turns are never pruned
- `protected_tools` - Tools like `task`, `todowrite`, `lsp_rename` are never pruned

---

## Layer 3: Context Recovery

### Anthropic Context Window Limit Recovery

**Directory**: `src/hooks/anthropic-context-window-limit-recovery/`

When a provider returns a context-limit error (the message array is too large), this hook orchestrates multi-step recovery:

```
Session error: context_length_exceeded
  |
  v
Strategy 1: Deduplication pruning
  - Remove duplicate tool outputs from stored parts
  - Retry request
  |
  v  (if still fails)
Strategy 2: Tool output truncation
  - Find and truncate the largest stored tool results
  - Target a specific token reduction ratio
  - Retry request
  |
  v  (if still fails)
Strategy 3: Aggressive truncation (if experimental.aggressive_truncation enabled)
  - Sweeping truncation across all stored tool outputs
  - Last resort before giving up
  - Retry request
```

The recovery operates on stored tool parts (filesystem or via the OpenCode SDK) and can modify historical message content to free space.

### Compaction Context Injector

**Directory**: `src/hooks/compaction-context-injector/`

When compaction happens (conversation is summarized to free context), the agent's identity can get lost. The summary might not preserve which agent is active, what model is being used, or which tools are available.

This hook solves that:

```
Before compaction:
  capture(sessionID) -> checkpoint agent config (agent name, model, tools)

During compaction:
  inject(sessionID) -> return COMPACTION_CONTEXT_PROMPT (re-states key context)

After compaction:
  event(session.compacted) -> run recovery:
    1. Load checkpoint
    2. Validate current agent/model matches checkpoint
    3. If mismatched: send noReply prompt to restore agent config
    4. Monitor assistant messages for no-text tails (degradation signal)
```

---

## How Everything Fits Together

Here's the full lifecycle showing when each mechanism fires:

```
User sends message
  |
  v
[chat.message hooks]
  - keywordDetector: registers mode-specific context with ContextCollector
  |
  v
[experimental.chat.messages.transform]
  - contextInjectorMessagesTransform: injects synthetic parts
    (AGENTS.md, README, rules from ContextCollector)
  - thinkingBlockValidator: fixes thinking block format
  |
  v
[Message sent to model]
  |
  v
Model responds with tool calls
  |
  v
For each tool call:
  |
  +--> [tool.execute.before]
  |      - rulesInjector: no-op here (active in after)
  |      - directoryAgentsInjector: no-op here
  |
  +--> Tool executes
  |
  +--> [tool.execute.after]  (in order)
         1. toolOutputTruncator: truncate large outputs
         2. contextWindowMonitor: add 70% usage warning if needed
         3. rulesInjector: find & inject matching rules
         4. directoryAgentsInjector: find & inject AGENTS.md
         5. directoryReadmeInjector: find & inject README.md
  |
  v
[Results returned to model, model continues]
  |
  v
[session.idle event]
  - preemptiveCompaction: check if compaction needed
  |
  v
[If context limit error]
  - anthropicContextWindowLimitRecovery: multi-step recovery
  |
  v
[If compaction triggered]
  - compactionContextInjector: preserve and restore agent config
```

---

## Configuration Reference

### Disabling Specific Mechanisms

```jsonc
{
  // Disable specific injection hooks
  "disabled_hooks": [
    "rules-injector",              // No rule file injection
    "directory-agents-injector",    // No AGENTS.md injection (auto-disabled on OC 1.1.37+)
    "directory-readme-injector",    // No README.md injection
    "context-window-monitor",       // No 70% usage warning
    "preemptive-compaction",        // No proactive compaction
    "tool-output-truncator"         // No automatic truncation (risky)
  ],

  // Experimental features
  "experimental": {
    "truncate_all_tool_outputs": false,  // Truncate ALL tools, not just whitelist
    "aggressive_truncation": false,      // Allow aggressive recovery truncation
    "disable_omo_env": false,            // Remove <omo-env> block from prompts
    "dynamic_context_pruning": {
      "enabled": false,
      "turn_protection": { "enabled": true, "turns": 3 },
      "strategies": {
        "deduplication": { "enabled": true },
        "supersede_writes": { "enabled": true, "aggressive": false },
        "purge_errors": { "enabled": true, "turns": 5 }
      }
    }
  }
}
```

### Rule File Locations

| Scope | Location | Applied When |
|-------|----------|-------------|
| **Project rules** (per-directory) | `.sisyphus/rules/*.md` | Glob matches target file |
| **Project rules** (per-directory) | `.cursor/rules/*.md` | Glob matches target file |
| **Project rules** (per-directory) | `.claude/rules/*.md` | Glob matches target file |
| **Project rules** (per-directory) | `.github/instructions/*.instructions.md` | Glob matches target file |
| **Project-wide** | `.github/copilot-instructions.md` | Always (single-file rule) |
| **User-global** | `~/.sisyphus/rules/*.md` | Glob matches target file |
| **User-global** | `~/.opencode/rules/*.md` | Glob matches target file |
| **User-global** | `~/.claude/rules/*.md` | Glob matches target file |

### Rule Frontmatter Reference

```yaml
---
description: What this rule does (informational)
globs: "src/**/*.ts"         # Glob pattern (picomatch syntax)
# OR
globs:                       # Multiple patterns
  - "src/**/*.ts"
  - "lib/**/*.tsx"
# OR
paths: "src/**"              # Claude Code alias for globs
# OR
applyTo: "**/*.go"           # Copilot alias for globs
alwaysApply: true            # Apply regardless of file path
---
```

---

## Source File Map

| File | Purpose |
|------|---------|
| **Injection** | |
| `src/hooks/directory-agents-injector/` | AGENTS.md discovery and injection |
| `src/hooks/directory-readme-injector/` | README.md discovery and injection |
| `src/hooks/rules-injector/` | Rule file discovery, matching, injection |
| `src/features/context-injector/` | ContextCollector + messages transform |
| `src/agents/env-context.ts` | Environment context generation |
| **Truncation** | |
| `src/hooks/tool-output-truncator.ts` | Post-tool output truncation |
| `src/shared/dynamic-truncator.ts` | Context-aware truncation utilities |
| `src/hooks/context-window-monitor.ts` | Usage tracking and warnings |
| `src/hooks/preemptive-compaction.ts` | Proactive compaction trigger |
| **Recovery** | |
| `src/hooks/anthropic-context-window-limit-recovery/` | Multi-strategy overflow recovery |
| `src/hooks/compaction-context-injector/` | Agent config preservation across compaction |
| **Shared** | |
| `src/shared/session-injected-paths.ts` | Per-session injection dedup storage |
| `src/shared/opencode-version.ts` | OpenCode version detection for auto-disable |

---

## Key Takeaways

1. **Context injection is event-driven.** AGENTS.md, README.md, and rules are injected after `read` tool calls, not upfront. This means context is added on-demand as the agent explores the codebase.

2. **Rules use glob matching** against relative file paths. A rule in `.sisyphus/rules/typescript.md` with `globs: "src/**/*.ts"` only injects when the agent reads/writes a matching TypeScript file.

3. **Deduplication prevents noise.** Both real-path dedup (symlink-aware) and content-hash dedup ensure the same content doesn't appear twice in a session.

4. **Root AGENTS.md is handled by OpenCode**, not the plugin. The plugin intentionally skips it to avoid duplication. Subdirectory AGENTS.md files are the plugin's responsibility (or native OpenCode on >= 1.1.37).

5. **The parent directory question**: When `ctx.directory` is a parent of multiple repos, the walker finds AGENTS.md inside whichever child repo the agent is currently reading from. It never crosses into sibling repos and skips the parent directory's own AGENTS.md.

6. **Three layers of defense** against context overflow: proactive truncation (tool output limits), proactive compaction (preemptive trigger), and reactive recovery (multi-strategy on error). Each layer catches what the previous one missed.

7. **All injection mechanisms support dynamic truncation** based on remaining context budget. Large AGENTS.md files or rules are automatically trimmed with a notice pointing to the full file.

---

Previous: [08 - Patterns for Your Own Projects](./08-patterns.md)
