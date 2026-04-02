# 学习指南 07 — 项目级记忆与 CLAUDE.md 系统

除了由 AI 自动整理的 `DreamTask` 记忆外，Claude Code 还支持一套由人工维护或项目定义的层级化指令系统，统称为 **CLAUDE.md 系统**。这让 AI 能够遵守特定的编码规范、项目架构和团队约定。

本章节将分析 `src/utils/claudemd.ts` 揭示的层级化记忆加载机制。

---

## 1. 四层记忆架构 (The Four-Layer Architecture)

Claude Code 会按特定顺序加载指令文件。加载顺序越靠后，优先级越高（即覆盖前面的指令）。

### 源码原文 (`src/utils/claudemd.ts`)
> ```typescript
> /**
>  * 1. Managed memory (eg. /etc/claude-code/CLAUDE.md) - 全局管理策略
>  * 2. User memory (~/.claude/CLAUDE.md) - 用户全局私有指令
>  * 3. Project memory (CLAUDE.md, .claude/CLAUDE.md) - 项目代码库共享指令
>  * 4. Local memory (CLAUDE.local.md) - 用户针对该项目的私有指令
>  */
> ```

### 技术解析
*   **层级覆盖**：如果你在项目根目录的 `CLAUDE.md` 写了“使用 Tab 缩进”，但在 `CLAUDE.local.md` 中写了“使用空格缩进”，AI 会以 `local` 版本为准。
*   **发现机制 (Upward Traversal)**：系统会从当前目录不断向父目录回溯，直到根目录。越靠近当前操作目录的文件，优先级越高。这使得单体仓库（Monorepo）中的子项目可以拥有独立的规范。

---

## 2. 规则过滤与条件加载 (Conditional Rules)

并非所有指令都会在每一轮对话中加载。Claude Code 支持基于文件路径的“按需加载”。

### 源码原文 (`src/utils/claudemd.ts`)
> ```typescript
> function parseFrontmatterPaths(rawContent: string) {
>   const { frontmatter, content } = parseFrontmatter(rawContent)
>   // 提取 frontmatter 中的 paths 字段，作为过滤用的 Glob 模式
>   if (!frontmatter.paths) return { content }
>   // ...
> }
> ```

### 教学解析
*   **按需加载**：通过在 `.claude/rules/*.md` 文件的 Frontmatter 中定义 `paths: "src/api/**"`，该规则仅在 AI 操作 API 相关文件时才会被注入上下文。
*   **Token 节省**：这种机制防止了大型项目因指令文件过多而迅速填满上下文窗口。

---

## 3. 指令包含指令：@include 指令

为了方便复用，CLAUDE.md 支持类似编程语言的 `include` 功能。

### 源码原文 (`src/utils/claudemd.ts`)
> ```typescript
> // 语法示例: @path, @./relative/path, @~/home/path
> function extractIncludePathsFromTokens(tokens, basePath) {
>   const includeRegex = /(?:^|\s)@((?:[^\s\\]|\\ )+)/g
>   // ... 递归加载被引用的文件
> }
> ```

### 教学解析
*   **模块化指令**：你可以将“测试规范”写在 `tests.md` 中，然后在主 `CLAUDE.md` 中通过 `@./tests.md` 引用。这极大地提高了指令的可维护性。

---

## 4. 自动预处理：剥离 HTML 注释与截断

为了确保注入的指令干净且不超限，系统会进行预处理。

### 源码原文 (`src/utils/claudemd.ts`)
> ```typescript
> export function stripHtmlComments(content: string) {
>   // 剥离 <!-- ... --> 格式的注释
> }
>
> // 限制索引文件 (MEMORY.md) 大小
> export const MAX_ENTRYPOINT_BYTES = 25_000
> ```

### 教学解析
*   **作者注释**：你可以在 `CLAUDE.md` 中使用 HTML 注释记录“为什么这么写”，AI 在读取时会自动忽略这些注释，从而节省 Token 并防止干扰。
*   **硬性截断**：如果 `MEMORY.md` 索引文件超过 25KB，系统会截断并发出警告，强制开发者保持索引的精简。

---

## 迁移到 Gemini 工作流的实战建议

1.  **模仿 .claude 结构**：在你的项目根目录创建一个 `.gemini/` 文件夹。
2.  **创建 GEMINI.md**：存放项目的全局规范（技术栈、代码风格、避坑指南）。
3.  **编写条件规则**：创建子文件夹 `.gemini/rules/`。对于特定模块，编写如 `database-rules.md`，并在开头注明 `[适用范围: src/db/**]`。
4.  **手动注入上下文**：在与 Gemini 对话开始时，如果你正在处理数据库，手动输入：“请遵循以下数据库规范：[粘贴 database-rules.md 内容]”。

---

## 思考与讨论

1.  **为什么需要 CLAUDE.local.md？**（提示：用于存放个人的 API Key 配置、本地开发环境特有的路径，或者不想被 Git 提交到团队仓库的个人偏好）。
2.  **向上溯源（Upward Traversal）的优势**：在 Monorepo 架构中，根目录的 `CLAUDE.md` 定义通用规范，子目录的 `CLAUDE.md` 定义特定服务的规范，这种设计解决了什么问题？
