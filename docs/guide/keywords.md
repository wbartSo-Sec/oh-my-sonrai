# Keyword Modes

oh-my-opencode detects keywords in your messages and injects mode-specific system directives that reshape the agent's behavior for that turn. Keywords inside code blocks are ignored to prevent accidental activation.

---

## The 3 Modes

| Mode | Trigger Words | Effect |
|------|--------------|--------|
| **Ultrawork** | `ultrawork`, `ulw` | Maximum effort orchestration. Explore exhaustively, plan formally, delegate aggressively, verify everything. |
| **Search** | `search`, `find`, `locate`, `grep`, `where is`, `show me`, + many more | Maximize search parallelism. Fire multiple explore/librarian agents, use direct search tools, never stop at first result. |
| **Analyze** | `analyze`, `investigate`, `debug`, `why is`, `how does`, `review`, `understand`, + many more | Deep analysis. Gather context before acting, consult specialists (Oracle/Artistry) for hard problems, synthesize before proceeding. |

Multiple modes can activate simultaneously. Asking "find all Redis usage and explain why" triggers both Search (`find`) and Analyze (`explain`, `why`).

---

## Ultrawork

**Triggers**: Only the exact words `ultrawork` or `ulw` (case-insensitive). Intentionally narrow so it never fires accidentally.

**What it does**:
- Forces the agent to announce "ULTRAWORK MODE ENABLED!"
- Requires 100% certainty before implementation (explore codebase, resolve all ambiguity, consult specialists)
- Mandates Plan agent invocation for any non-trivial task
- Enforces zero-tolerance for partial work, scope reduction, or assumptions
- Requires manual QA verification with evidence before declaring done
- Demands full agent utilization (explore, librarian, oracle, artistry, category+skills delegation)

**Model-specific variants**: The injected directive adapts to the active model:
- **Claude** (default): Natural tool-like agent usage, parallel execution emphasis
- **GPT-5.4**: Adapted for GPT's reasoning style
- **Gemini**: Adapted for Gemini's style
- **Planner agents** (Prometheus): Ultrawork is filtered out entirely. Planners don't receive it.

**When to use it**: Complex multi-step tasks where you want the agent to plan thoroughly, delegate to specialists, and not cut corners. Think of it as the "no shortcuts" switch.

---

## Search

**Triggers** (case-insensitive):

| Language | Keywords |
|----------|----------|
| English | `search`, `find`, `locate`, `lookup`, `look up`, `explore`, `discover`, `scan`, `grep`, `query`, `browse`, `detect`, `trace`, `seek`, `track`, `pinpoint`, `hunt`, `where is`, `show me`, `list all` |
| Korean | 검색, 찾아, 탐색, 조회, 스캔, 서치, 뒤져, 찾기, 어디, 추적, 탐지, 찾아봐, 찾아내, 보여줘, 목록 |
| Japanese | 検索, 探して, 見つけて, サーチ, 探索, スキャン, どこ, 発見, 捜索, 見つけ出す, 一覧 |
| Chinese | 搜索, 查找, 寻找, 查询, 检索, 定位, 扫描, 发现, 在哪里, 找出来, 列出 |
| Vietnamese | tìm kiếm, tra cứu, định vị, quét, phát hiện, truy tìm, tìm ra, ở đâu, liệt kê |

**What gets injected**:
```
[search-mode]
MAXIMIZE SEARCH EFFORT. Launch multiple background agents IN PARALLEL:
- explore agents (codebase patterns, file structures, ast-grep)
- librarian agents (remote repos, official docs, GitHub examples)
Plus direct tools: Grep, ripgrep (rg), ast-grep (sg)
NEVER stop at first result - be exhaustive.
```

**When to use it**: When you want thorough search results, not just the first match. The agent fires multiple explore/librarian agents in parallel and uses direct search tools to cast a wide net.

---

## Analyze

**Triggers** (case-insensitive):

| Language | Keywords |
|----------|----------|
| English | `analyze`, `analyse`, `investigate`, `examine`, `research`, `study`, `deep-dive`, `inspect`, `audit`, `evaluate`, `assess`, `review`, `diagnose`, `scrutinize`, `dissect`, `debug`, `comprehend`, `interpret`, `breakdown`, `understand`, `why is`, `how does`, `how to` |
| Korean | 분석, 조사, 파악, 연구, 검토, 진단, 이해, 설명, 원인, 이유, 뜯어봐, 따져봐, 평가, 해석, 디버깅, 디버그, 어떻게, 왜, 살펴 |
| Japanese | 分析, 調査, 解析, 検討, 研究, 診断, 理解, 説明, 検証, 精査, 究明, デバッグ, なぜ, どう, 仕組み |
| Chinese | 调查, 检查, 剖析, 深入, 诊断, 解释, 调试, 为什么, 原理, 搞清楚, 弄明白 |
| Vietnamese | phân tích, điều tra, nghiên cứu, kiểm tra, xem xét, chẩn đoán, giải thích, tìm hiểu, gỡ lỗi, tại sao |

**What gets injected**:
```
[analyze-mode]
ANALYSIS MODE. Gather context before diving deep:
CONTEXT GATHERING (parallel):
- 1-2 explore agents (codebase patterns, implementations)
- 1-2 librarian agents (if external library involved)
- Direct tools: Grep, AST-grep, LSP for targeted searches
IF COMPLEX - DO NOT STRUGGLE ALONE. Consult specialists:
- Oracle: Conventional problems (architecture, debugging, complex logic)
- Artistry: Non-conventional problems (different approach needed)
SYNTHESIZE findings before proceeding.
```

**When to use it**: Activates automatically on most investigative questions. Words like "understand", "how does", "why is", "review", and "debug" all trigger it. Most of the time you don't need to think about this one; it activates naturally when you're asking analytical questions.

---

## How Detection Works

1. Your message text is extracted from the conversation
2. Code blocks (`` ``` `` and `` ` ``) are stripped so keywords in code don't trigger modes
3. System directive messages are skipped (prevents infinite loops)
4. Each keyword pattern is tested against the cleaned text
5. For each match, the corresponding mode message is injected as a system directive into the conversation
6. The agent processes the directive alongside your message, shaping its behavior for that turn

The detection runs in the `keyword-detector` hook (`src/hooks/keyword-detector/`), which operates on the `chat.message` event.

---

## Practical Notes

- **Analyze is the broadest trigger.** It fires on common words like "understand", "review", "how does". This is intentional: most investigative questions benefit from the analysis boost.
- **Ultrawork is the narrowest.** Only the explicit words `ultrawork`/`ulw` activate it. It's a deliberate opt-in.
- **Modes are per-message.** They apply to the turn where the keyword appears, not the entire session.
- **Co-triggering is normal.** "Find all auth code and explain the pattern" triggers both Search and Analyze. The agent gets both directives.
- **Code is safe.** Pasting code that contains `search` or `debug` won't activate modes. Only your natural language text is scanned.
