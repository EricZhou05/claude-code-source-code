# 学习指南 06 — 后台任务与 Dream 记忆整理 (DreamTask)

在 Claude Code 中，AI 不仅仅是在对话中学习，它还有一个专门的“后台整理”机制，称为 **DreamTask**。这个功能由 Feature Flag `tengu_onyx_plover` 控制，模拟了人类睡眠时巩固记忆的过程。

本章节将分析 `src/tasks/DreamTask/` 和 `src/services/autoDream/` 揭示的 AI 记忆持久化方案。

---

## 1. 什么是 DreamTask？

DreamTask 是一个在后台运行的“影子代理（Forked Agent）”。它在用户不直接与 AI 交互时（或在后台）静默运行，负责阅读过去的会话记录，并将其提炼为长期的、结构化的“记忆文件”。

### 源码原文 (`src/services/autoDream/autoDream.ts`)
> ```typescript
> // Gate order (cheapest first):
> //   1. Time: hours since lastConsolidatedAt >= minHours (24h)
> //   2. Sessions: transcript count >= minSessions (5 sessions)
> //   3. Lock: no other process mid-consolidation
> ```

### 技术解析
*   **触发三要素**：
    1.  **时间门禁**：距离上次整理必须超过 24 小时（`minHours: 24`）。
    2.  **会话门禁**：累积的新会话必须达到 5 个（`minSessions: 5`）。
    3.  **分布式锁**：通过 `tryAcquireConsolidationLock` 确保同一时间只有一个整理任务在运行，防止记忆文件被写乱。

---

## 2. 记忆整理的四个阶段 (The Four Phases)

Dream 代理不是漫无目的地阅读，它遵循一套严格的算法流程。

### 源码原文 (`src/services/autoDream/consolidationPrompt.ts`)
> ```markdown
> ## Phase 1 — Orient (定位)
> - `ls` 记忆目录，阅读索引文件 `ENTRYPOINT_NAME`。
>
> ## Phase 2 — Gather recent signal (搜集信号)
> - 查找新信息。来源优先级：日志文件 > 发现矛盾的事实 > 会话转录本 (JSONL)。
>
> ## Phase 3 — Consolidate (合并巩固)
> - 将新信号合并到现有的主题文件中，而不是创建重复文件。
> - 将相对日期（“昨天”）转换为绝对日期。
>
> ## Phase 4 — Prune and index (剪枝与索引)
> - 更新索引文件，确保其保持在 25KB 以内。删除过时或被取代的记忆指针。
> ```

### 教学解析
*   **反碎片化**：Phase 3 强调“合并”而非“新建”，防止 AI 记忆变得零碎。
*   **时效性修复**：将“昨天”改为具体日期，是保证长期记忆可用性的关键细节。
*   **索引压缩**：Phase 4 强制限制索引文件大小，这其实是为了节省 AI 在下次启动时读取背景信息的 Token 开销。

---

## 3. UI 表现与透明度

虽然是后台任务，但 Claude Code 通过 `DreamTask.ts` 将其可视化，让用户知道 AI 正在“进化”。

### 源码原文 (`src/tasks/DreamTask/DreamTask.ts`)
> ```typescript
> export type DreamTaskState = TaskStateBase & {
>   phase: DreamPhase // 'starting' | 'updating'
>   filesTouched: string[] // 被修改的记忆文件
>   turns: DreamTurn[] // 整理过程中的思考片段
> }
> ```

### 技术解析
*   **进度反馈**：用户可以在终端状态栏看到 AI 正在“Dreaming”。
*   **结果通知**：当整理完成，系统会发送一个类似 `Improved 3 memories` 的系统消息，告知用户哪些记忆被优化了。

---

## 4. 核心工具限制 (Tool Constraints)

为了安全起见，后台整理任务的权限被严格限制。

### 源码原文 (`src/services/autoDream/autoDream.ts`)
> `Bash is restricted to read-only commands (ls, find, grep, cat...). Anything that writes, redirects to a file, or modifies state will be denied.`

### 教学解析
*   **最小权限原则**：Dream 任务只能读取代码和转录本，只能修改特定的 `memory/` 目录。它不能运行编译命令，也不能修改用户的源代码。这种权限隔离保证了后台任务的安全性。

---

## 迁移到 Gemini 工作流的实战建议

如果你想为 Gemini 构建类似的记忆系统：

1.  **建立“记忆文件夹”**：创建一个 `memories/` 目录，存放按主题分类的 Markdown 文件。
2.  **定期自省**：每隔一段时间，将最近的 5-10 条对话记录喂给 Gemini，指令为：“请阅读这些对话，提取出关于项目架构、开发偏好或已解决难题的新知识，并更新到 `memories/` 目录中的相应文件。”
3.  **维护索引**：始终保持一个 `index.md`，记录每个记忆文件的简要内容。在每次新对话开始时，只给 Gemini 这个 `index.md`，让它按需调取详细记忆。

---

## 思考与讨论

1.  **为什么要区分“会话转录本”和“记忆文件”？**（提示：转录本包含大量废话和工具输出，记忆文件是经过模型提炼的结构化知识，Token 效率更高）。
2.  **绝对日期转换的重要性**：如果 AI 记住了“昨天修复了登录 Bug”，一个月后它如何理解这个“昨天”？这对长线项目管理有什么启示？
