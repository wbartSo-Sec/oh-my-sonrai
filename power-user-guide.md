# Oh My OpenCode — Power User Guide

## The One Thing: `ultrawork`

Type `ultrawork` (or `ulw`). That's it. Every agent activates. Every optimization kicks in. The agent doesn't stop until the job is done. If someone only remembers one thing, it's this.

---

## The Agent Team (11 Specialists)

This isn't one AI doing everything. It's a coordinated team:

| Agent | Role | Model Sweet Spot |
|---|---|---|
| **Sisyphus** | Orchestrator. Plans, delegates, verifies. | Opus / Kimi K2.5 / GLM-5 |
| **Hephaestus** | Deep autonomous worker. Give goals, not recipes. | GPT-5.3 Codex |
| **Prometheus** | Strategic planner. Interviews you before coding starts. | Opus / Kimi K2.5 |
| **Oracle** | Architecture consultant. Read-only, expensive, brilliant. | High-reasoning model |
| **Librarian** | External docs + OSS reference search | Fast model |
| **Explore** | Fast codebase grep (contextual search) | Fast model |
| **Metis** | Pre-planning consultant (finds ambiguities in requests) | - |
| **Momus** | Plan reviewer (catches gaps before execution) | - |
| **Atlas** | Master orchestrator for background/continuation sessions | - |
| **Multimodal Looker** | PDF/image/diagram analysis | - |
| **Sisyphus Junior** | Delegated subtask worker (inherits from category) | Category-dependent |

The key insight: Sisyphus doesn't pick a **model** when delegating. It picks a **category** (`visual-engineering`, `ultrabrain`, `deep`, `quick`), and the harness routes to the optimal model automatically. You configure models once, then forget about it.

---

## "Never Stops" Mechanisms (The Boulder)

Three independent systems ensure work actually completes:

1. **Todo Continuation Enforcer** — If the agent goes idle with incomplete todos, a 2-second countdown starts, then it gets yanked back to work. Exponential backoff on failures (30s base, doubles each failure, 5-minute pause after 5 consecutive failures).

2. **Ralph Loop** (`/ralph-loop` or `/ulw-loop`) — Self-referential development loop. Agent works, checks if done, loops. Only stops when it emits `<promise>DONE</promise>` or hits max iterations (default 100). State persists to disk, so it survives crashes.

3. **Atlas Hook** — Master orchestrator for background/boulder sessions. Decision gates check session type, abort signals, failure counts, background tasks, agent match, cooldown. Prevents infinite loops while ensuring completion.

---

## Hash-Anchored Edits (Hashline)

The single biggest reliability improvement. Every line the agent reads gets a content hash:

```
11#VK| function hello() {
22#XJ|   return "world";
33#MB| }
```

Edits reference these hashes. If the file changed since the last read, the hash won't match and the edit is **rejected before corruption**. This took one model from **6.7% to 68.3%** success rate just by changing the edit tool. 26 files power this system (`src/tools/hashline-edit/`).

---

## Comment Checker (Anti-AI-Slop)

Every `Write`, `Edit`, and `apply_patch` call is intercepted. A dedicated CLI tool scans the output for AI-generated comment patterns and strips them. The code reads like a senior engineer wrote it, not `// This function handles the authentication logic for the user`. Configurable with `custom_prompt`.

---

## Background Agents (True Parallelism)

Fire 5+ specialist agents simultaneously. Each runs independently with its own context. Results arrive via system notifications. This is how Sisyphus explores codebases:

```
task(subagent_type="explore", run_in_background=true, ...)  // fires immediately
task(subagent_type="explore", run_in_background=true, ...)  // parallel
task(subagent_type="librarian", run_in_background=true, ...) // also parallel
// Continue working. System notifies when each completes.
```

5 concurrent tasks per model/provider (configurable).

---

## Built-in MCPs (Always On)

Three remote MCPs ship with the plugin, no setup:

| MCP | What | Why |
|---|---|---|
| **Exa/Tavily** (websearch) | Real-time web search | Current docs, Stack Overflow, news |
| **Context7** | Official library documentation | Up-to-date API docs with code examples |
| **Grep.app** | GitHub code search across 1M+ repos | Real-world usage patterns |

Plus **Skill-Embedded MCPs** — skills can bring their own MCP servers that spin up on-demand and tear down when done. Zero context bloat.

---

## Built-in Skills

| Skill | What it does |
|---|---|
| **Playwright** | Full browser automation — test, screenshot, scrape, interact |
| **Dev Browser** | Persistent browser state for multi-step web workflows |
| **Git Master** | Atomic commits, rebase surgery, blame, bisect, history search |
| **Frontend UI/UX** | Designer-turned-developer approach to UI — works even without mockups |

Users can add their own: `.opencode/skills/*/SKILL.md` (project) or `~/.config/opencode/skills/*/SKILL.md` (user). Skills inherit tool permissions and MCP servers.

---

## Slash Commands

| Command | What |
|---|---|
| `/init-deep` | Auto-generates hierarchical `AGENTS.md` files throughout your project for context efficiency |
| `/ralph-loop` | Start self-referential dev loop until completion |
| `/ulw-loop` | Same but with ultrawork mode |
| `/start-work` | Invoke Prometheus planner — interview mode before execution |
| `/refactor` | Intelligent refactoring with LSP, AST-grep, architecture analysis, TDD verification |
| `/handoff` | Create detailed context summary for continuing in a new session |
| `/stop-continuation` | Emergency brake for all continuation mechanisms |

---

## Recovery & Resilience (46 Hooks)

The hook system is where the real magic hides:

- **Session Recovery** — Auto-recovers from crashes
- **Runtime Fallback** — API provider errors? Auto-switches to another model
- **Model Fallback** — Provider-level fallback chains (Claude > OpenAI > Gemini > Copilot > etc.)
- **Edit Error Recovery** — Failed file edits get retried automatically
- **Delegate Task Retry** — Failed task delegations get retried
- **JSON Error Recovery** — Malformed JSON in tool calls gets auto-corrected
- **Context Window Monitor** + **Preemptive Compaction** — Triggers compaction before you hit the hard limit
- **Anthropic Context Window Limit Recovery** — Multi-strategy recovery (truncation, compaction, summarization)
- **Compaction Todo Preserver** — Todos survive compaction events
- **GPT Permission Continuation** — Auto-handles GPT's permission-tail interruptions

---

## Developer Experience Hooks

- **Think Mode** — Dynamic thinking budget adjustment per model
- **Write Existing File Guard** — Forces `Read` before `Write` on existing files (prevents blind overwrites)
- **Question Label Truncator** — Auto-truncates overly long question labels
- **Directory AGENTS.md Injector** — Auto-injects relevant `AGENTS.md` when reading directories
- **Rules Injector** — Conditional rules from AGENTS.md/config applied per-context
- **Keyword Detector** — Detects `ultrawork`, `search`, `analyze`, `prove-yourself` and injects mode-specific prompts

---

## IDE-Grade Tooling (26 Tools)

Not just text manipulation. Real IDE tools:

- **LSP**: `lsp_rename`, `lsp_goto_definition`, `lsp_find_references`, `lsp_diagnostics`, `lsp_symbols`, `lsp_prepare_rename`
- **AST-Grep**: Pattern-aware search and rewrite across 25 languages
- **Tmux**: Full interactive terminal — REPLs, debuggers, TUI apps, all live
- **Session Manager**: List, read, search, analyze past sessions
- **Look At**: Analyze PDFs, images, diagrams with vision models

---

## Multi-Level Config

```
Project (.opencode/oh-my-opencode.jsonc)
  -> User (~/.config/opencode/oh-my-opencode.jsonc)
    -> Defaults
```

JSONC with comments. Zod v4 validated. Deep-merge for agents/categories, set-union for `disabled_*` arrays. Auto-migration of legacy keys. You can override model, temperature, prompt, and permissions per agent.

---

## The TL;DR for Showing Someone

1. **Install, type `ultrawork`, watch it go** — the 30-second demo
2. **Show background agents firing in parallel** — "it's a team, not one AI"
3. **Show the comment checker catching slop** — "code reads like a senior wrote it"
4. **Show Ralph Loop on a real task** — "it doesn't stop until it's done"
5. **Show `/init-deep`** — "automatic project context for every agent"
6. **Show a model failure then automatic fallback** — "it doesn't crash, it adapts"
7. **Show hashline edit rejection on a stale file** — "it can't corrupt your code"

The plugin's thesis: stop configuring, stop babysitting, stop worrying about which model to use. The harness handles it. You describe what you want. It ships.
