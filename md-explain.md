# SOUL.md 和 Project Context 说明

## SOUL.md

SOUL.md 是 OpenClaw agent 的**人格/声音层**，它被注入到每次 LLM 调用的系统提示词中。

### 组织结构

SOUL.md 模板包含以下几个部分：

```
# SOUL.md - Who You Are

## Core Truths        # 核心原则：行事风格、态度
## Boundaries         # 边界：隐私、外部行动限制
## Vibe               # 氛围：简洁vs详细、不媚俗
## Continuity         # 连续性：session间如何持久化记忆
```

### 设计原则

**应该包含**：tone、opinions、brevity、humor、boundaries、bluntness

**不应该变成**：
- 人生故事
- changelog
- 安全策略堆砌
- 无行为影响的大段氛围文字

### 在系统提示词中的位置

从 `system-prompt.md` 文档可知，Workspace bootstrap 文件按此顺序注入：

```
AGENTS.md → SOUL.md → TOOLS.md → IDENTITY.md → USER.md → HEARTBEAT.md → BOOTSTRAP.md → MEMORY.md
```

**Project Context** 层之上（prompt cache boundary 以上），所以是每次 turn 都稳定注入的。

### 大小限制

- 单文件最大：`agents.defaults.bootstrapMaxChars`（默认 12000）
- 总体最大：`agents.defaults.bootstrapTotalMaxChars`（默认 60000）

### 核心目的

> "SOUL.md is where your agent's voice lives." — 让人不像 generic assistant sludge，通过 concise + sharp 的指令让它有真正的性格，而不是 corporate hedging language。

---

## Project Context

**Project Context** 是 OpenClaw 系统提示词中的一个区块，位于 **prompt cache boundary 之上**（稳定可缓存），用于注入 workspace 的身份文件和配置。

### 注入的文件（按顺序）

```
AGENTS.md       → 工作区规则、工作流程
SOUL.md         → agent 的人格/声音/风格
TOOLS.md        → 工具配置（本地笔记，如相机名、SSH详情、语音偏好）
IDENTITY.md     → agent 身份（名字、角色设定）
USER.md         → 用户信息
HEARTBEAT.md    → 心跳任务清单（如果启用心跳）
BOOTSTRAP.md    → 仅全新 workspace 首次启动
MEMORY.md       → 长期记忆摘要（如果存在）
```

### 关键特性

| 特性 | 说明 |
|------|------|
| **每次 turn 都注入** | 除非特定文件有 gate 拦截（如 `HEARTBEAT.md` 可独立控制） |
| **在 cache boundary 之上** | 稳定内容，prompt cache 可复用 |
| **Sub-agent 不同** | 仅注入 `AGENTS.md` 和 `TOOLS.md`，保持轻量 |
| **大小限制** | 单文件默认 12K，总量默认 60K，超出自动截断 |

### 截断行为

- 文件超过 `bootstrapMaxChars` → 截断但磁盘完整保留
- 截断后模型只看到缩短版本，可通过 `memory_search` / `memory_get` 按需读取完整内容
- `MEMORY.md` 建议保持精简摘要，详细日记应放在 `memory/*.md`

### 内部 hook

可通过 `agent:bootstrap` hook 拦截并替换注入内容，例如切换不同 persona。

### 一句话总结

Project Context = workspace 的"身份档案"，每次对话都作为稳定上下文注入，让 agent 知道自己是谁、什么风格、可以做什么。

---

## Prompt 调用模型前的组织结构

根据 `src/agents/system-prompt.ts`，OpenClaw 在调用模型前将 prompt 分为 **两大块**：

### 1. 稳定前缀（Stable Prefix）— **cache boundary 以上**

按顺序包含以下 sections：

```
## Tooling              # 可用工具列表 + 使用指导
## Tool Call Style       # 何时 narration、何时静默
## Execution Bias        # 执行偏差：act in-turn、continue until done
## Safety               # 安全护栏：no power-seeking、obey stop/pause
## OpenClaw Control     # 配置/重启用 gateway tool、禁止发明 CLI
## Skills               # 可用技能列表（按需加载 SKILL.md）
## Memory               # Memory Recall 指导
## OpenClaw Self-Update  # 如何安全修改配置
## Model Aliases        # 模型别名
## Workspace            # 工作目录路径
## Documentation        # 本地文档/源码位置
## Sandbox              # 沙箱环境信息（如启用）
## User Identity        # 用户身份信息
## Current Date & Time  # 时区（cache-stable）
## Workspace Files      # 注入文件列表提示
## Assistant Output    # 输出格式指令
# Project Context       # 注入的 workspace 文件（AGENTS.md、SOUL.md 等）
## Reasoning Format     # 推理格式提示（如启用）
```

**关键**：`SYSTEM_PROMPT_CACHE_BOUNDARY` 标记之前的全部内容都在 cache boundary 以上，可被 prompt cache 复用。

### 2. 动态上下文（Dynamic Context）— **cache boundary 以下**

```
# Dynamic Project Context   # 动态文件（非稳定注入）
## Silent Replies           # 无话可说时的响应格式
## Webchat Canvas           # Canvas 特定指导
## Messaging                # 消息 channel 特定指导
## Voice                    # 语音/TTS 提示
## Group Chat Context       # 群聊上下文
## Reactions               # 反应指导
## Heartbeats              # 心跳任务
## Runtime                 # 运行时信息（host/os/node/model等）
```

### 核心设计原则

| 设计 | 说明 |
|------|------|
| **Stable Prefix** | 稳定内容放 cache boundary 以上，复用 prompt cache |
| **Dynamic Context** | 每次 turn 变化的放 boundary 以下 |
| **Prompt Modes** | `full`（默认）、`minimal`（子 agent）、`none`（仅身份行） |
| **Provider Override** | provider 可替换 `interaction_style`、`tool_call_style`、`execution_bias` |
| **Provider Suffix** | stable prefix above cache boundary、dynamic suffix below |

### 一句话总结

Prompt = **稳定前缀（Tooling/Safety/Skills/Workspace/Project Context）** + **动态上下文（Messaging/Group Chat/Runtime）**，中间以 `SYSTEM_PROMPT_CACHE_BOUNDARY` 分割，boundary 以上可缓存，以下每次变化。

---

## Prompt 长度计算与 Context Window 管理

根据 `src/agents/bootstrap-budget.ts`、`src/agents/context-window-guard.ts`、`src/agents/pi-embedded-helpers/bootstrap.ts`，OpenClaw 的 prompt 长度管理分为以下层次：

### 1. Bootstrap 文件 — 按字符（chars）截断

```
DEFAULT_BOOTSTRAP_MAX_CHARS = 12,000      # 单文件最大字符数
DEFAULT_BOOTSTRAP_TOTAL_MAX_CHARS = 60,000 # 所有 bootstrap 文件总计最大字符数
```

**截断策略**：
- 头部 75% + 尾部 25%，中间用 marker 标记截断
- marker 格式：`[...truncated, read ${fileName} for full content...]`
- 如果剩余 budget 不足，最小保留 64 字符

```typescript
// bootstrap.ts trimBootstrapContent()
const BOOTSTRAP_HEAD_RATIO = 0.75;
const BOOTSTRAP_TAIL_RATIO = 0.25;
```

### 2. Token 估算 — 使用 `estimateTokens()`

实际 token 数通过 `@earendil-works/pi-coding-agent` 的 `estimateTokens()` 计算：

```typescript
// compaction.ts
const SAFETY_MARGIN = 1.2; // 20% buffer 补偿 estimateTokens() 的不准确性

// 计算 effective max tokens
const effectiveMax = Math.floor(maxTokens / SAFETY_MARGIN);
```

**估算不准确的原因**：`chars/4` heuristic 对多字节字符、特殊 token、代码等会低估。

### 3. Context Window 保护 — 硬性限制

```typescript
// context-window-guard.ts
CONTEXT_WINDOW_HARD_MIN_TOKENS = 4,000   # 最小 context window
CONTEXT_WINDOW_WARN_BELOW_TOKENS = 8,000  # 警告阈值
CONTEXT_WINDOW_HARD_MIN_RATIO = 0.1      # 最少 10% 的 context window
CONTEXT_WINDOW_WARN_BELOW_RATIO = 0.2     # 低于 20% 警告
```

**Token 来源优先级**：
1. `modelsConfig` 中的 `contextTokens` / `contextWindow`
2. 模型本身的 `modelContextTokens` / `modelContextWindow`
3. 默认值 `8,000`

### 4. 完整流程

```
Bootstrap 文件内容（原始 chars）
    ↓
按 maxChars 截断（head 75% + tail 25%）
    ↓
检查 totalMaxChars 总限制
    ↓
注入 Project Context 后
    ↓
调用 estimateTokens() 估算总 token 数
    ↓
对比 contextWindow：
    - tokens < hardMinTokens (4k) → BLOCK
    - tokens < warnBelowTokens (8k) → WARN
    - 不足 20% buffer → 触发压缩（compaction）
```

### 5. 配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agents.defaults.bootstrapMaxChars` | 12,000 | 单文件最大字符数 |
| `agents.defaults.bootstrapTotalMaxChars` | 60,000 | bootstrap 文件总计最大字符数 |
| `agents.defaults.contextTokens` | - | 强制限制的 context token 上限 |

### 一句话总结

Bootstrap 阶段用字符数快速截断，实际发送给模型的 token 数通过 `estimateTokens()` 估算后与模型的 context window 比较，最终决定是否需要 compaction。
