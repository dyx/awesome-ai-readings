# Claude 如何记住你的项目

> - **Version:** 260324
> - **Source:** [Claude 如何记住你的项目](https://code.claude.com/docs/en/memory)
> - **Tags:** `Memory`

> 使用 CLAUDE.md 文件为 Claude 提供持久指令，并让 Claude 通过自动记忆功能自动积累学习内容。

每个 Claude Code 会话都从一个全新的上下文窗口开始。两种机制可以跨会话传递知识：

* **CLAUDE.md 文件**：你编写的指令，为 Claude 提供持久上下文
* **自动记忆**：Claude 根据你的更正和偏好自己编写的笔记

## CLAUDE.md 与自动记忆

Claude Code 有两个互补的记忆系统。两者都在每次对话开始时加载。Claude 将它们视为上下文，而不是强制配置。你的指令越具体和简洁，Claude 遵循它们的一致性就越高。

|          | CLAUDE.md 文件  | 自动记忆                   |
| :------- | :------------ | :--------------------- |
| **谁编写**  | 你             | Claude                 |
| **包含内容** | 指令和规则         | 学习和模式                  |
| **范围**   | 项目、用户或组织      | 每个工作树                  |
| **加载到**  | 每个会话          | 每个会话（前 200 行）          |
| **用于**   | 编码标准、工作流、项目架构 | 构建命令、调试见解、Claude 发现的偏好 |

当你想指导 Claude 的行为时，使用 CLAUDE.md 文件。自动记忆让 Claude 从你的更正中学习，无需手动操作。

subagents 也可以维护自己的自动记忆。有关详细信息，请参阅 [subagent 配置](/zh-CN/sub-agents#enable-persistent-memory)。

## CLAUDE.md 文件

CLAUDE.md 文件是 markdown 文件，为项目、你的个人工作流或整个组织为 Claude 提供持久指令。你用纯文本编写这些文件；Claude 在每个会话开始时读取它们。

### 选择 CLAUDE.md 文件的位置

CLAUDE.md 文件可以位于多个位置，每个位置有不同的范围。更具体的位置优先于更广泛的位置。

| 范围 | 位置 | 目的  | 用例示例 | 共享对象 |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ---------------- | ------------ |
| **托管策略** | • macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br />• Linux 和 WSL: `/etc/claude-code/CLAUDE.md`<br />• Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | 由 IT/DevOps 管理的组织范围指令 | 公司编码标准、安全策略、合规要求 | 组织中的所有用户     |
| **项目指令** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md`                                                                                                                                 | 项目的团队共享指令             | 项目架构、编码标准、常见工作流  | 通过源代码控制的团队成员 |
| **用户指令** | `~/.claude/CLAUDE.md`                                                                                                                                                 | 所有项目的个人偏好             | 代码样式偏好、个人工具快捷方式  | 仅你（所有项目）     |

工作目录上方目录层次结构中的 CLAUDE.md 文件在启动时完整加载。子目录中的 CLAUDE.md 文件在 Claude 读取这些目录中的文件时按需加载。有关完整的解析顺序，请参阅 [CLAUDE.md 文件如何加载](#how-claudemd-files-load)。

对于大型项目，你可以使用 [项目规则](#organize-rules-with-clauderules) 将指令分解为特定主题的文件。规则让你将指令范围限定到特定文件类型或子目录。

### 设置项目 CLAUDE.md

项目 CLAUDE.md 可以存储在 `./CLAUDE.md` 或 `./.claude/CLAUDE.md` 中。创建此文件并添加适用于在项目上工作的任何人的指令：构建和测试命令、编码标准、架构决策、命名约定和常见工作流。这些指令通过版本控制与你的团队共享，因此请关注项目级标准而不是个人偏好。

<Tip>
  运行 `/init` 自动生成起始 CLAUDE.md。Claude 分析你的代码库并创建一个包含构建命令、测试指令和它发现的项目约定的文件。如果 CLAUDE.md 已存在，`/init` 会建议改进而不是覆盖它。从那里进行细化，添加 Claude 不会自己发现的指令。
</Tip>

### 编写有效的指令

CLAUDE.md 文件在每个会话开始时加载到上下文窗口中，与你的对话一起消耗令牌。因为它们是上下文而不是强制配置，你编写指令的方式会影响 Claude 遵循它们的可靠性。具体、简洁、结构良好的指令效果最好。

**大小**：每个 CLAUDE.md 文件目标在 200 行以下。较长的文件消耗更多上下文并降低遵守度。如果你的指令变得很大，使用 [导入](#import-additional-files) 或 [`.claude/rules/`](#organize-rules-with-clauderules) 文件进行分割。

**结构**：使用 markdown 标题和项目符号来分组相关指令。Claude 扫描结构的方式与读者相同：有组织的部分比密集段落更容易遵循。

**具体性**：编写具体到足以验证的指令。例如：

* "使用 2 空格缩进"而不是"正确格式化代码"
* "在提交前运行 `npm test`"而不是"测试你的更改"
* "API 处理程序位于 `src/api/handlers/`"而不是"保持文件有组织"

**一致性**：如果两条规则相互矛盾，Claude 可能会任意选择一条。定期审查你的 CLAUDE.md 文件、子目录中的嵌套 CLAUDE.md 文件和 [`.claude/rules/`](#organize-rules-with-clauderules) 以删除过时或冲突的指令。在 monorepos 中，使用 [`claudeMdExcludes`](#exclude-specific-claudemd-files) 跳过与你的工作无关的其他团队的 CLAUDE.md 文件。

### 导入其他文件

CLAUDE.md 文件可以使用 `@path/to/import` 语法导入其他文件。导入的文件在启动时展开并加载到上下文中，与引用它们的 CLAUDE.md 一起。

允许相对路径和绝对路径。相对路径相对于包含导入的文件解析，而不是工作目录。导入的文件可以递归导入其他文件，最大深度为五跳。

要引入 README、package.json 和工作流指南，在你的 CLAUDE.md 中的任何地方使用 `@` 语法引用它们：

```text  theme={null}
有关项目概述，请参阅 @README，有关此项目的可用 npm 命令，请参阅 @package.json。

# 其他指令
- git 工作流 @docs/git-instructions.md
```

对于你不想签入的个人偏好，从你的主目录导入文件。导入位于共享 CLAUDE.md 中，但它指向的文件保留在你的机器上：

```text  theme={null}
# 个人偏好
- @~/.claude/my-project-instructions.md
```

<Warning>
  Claude Code 第一次在项目中遇到外部导入时，它会显示一个批准对话框，列出这些文件。如果你拒绝，导入保持禁用状态，对话框不会再出现。
</Warning>

有关组织指令的更结构化方法，请参阅 [`.claude/rules/`](#organize-rules-with-clauderules)。

### CLAUDE.md 文件如何加载

Claude Code 通过从当前工作目录向上遍历目录树来读取 CLAUDE.md 文件，检查沿途的每个目录。这意味着如果你在 `foo/bar/` 中运行 Claude Code，它会从 `foo/bar/CLAUDE.md` 和 `foo/CLAUDE.md` 加载指令。

Claude 还在当前工作目录下的子目录中发现 CLAUDE.md 文件。它们不是在启动时加载，而是在 Claude 读取这些子目录中的文件时包含。

如果你在一个大型 monorepo 中工作，其他团队的 CLAUDE.md 文件被拾取，使用 [`claudeMdExcludes`](#exclude-specific-claudemd-files) 跳过它们。

#### 从其他目录加载

`--add-dir` 标志使 Claude 可以访问主工作目录外的其他目录。默认情况下，不加载这些目录中的 CLAUDE.md 文件。

要也从其他目录加载 CLAUDE.md 文件，包括 `CLAUDE.md`、`.claude/CLAUDE.md` 和 `.claude/rules/*.md`，设置 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` 环境变量：

```bash  theme={null}
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir ../shared-config
```

### 使用 `.claude/rules/` 组织规则

对于较大的项目，你可以使用 `.claude/rules/` 目录将指令组织到多个文件中。这使指令保持模块化并更容易让团队维护。规则也可以 [范围限定到特定文件路径](#path-specific-rules)，因此它们仅在 Claude 处理匹配文件时加载到上下文中，减少噪音并节省上下文空间。

<Note>
  规则在每个会话或打开匹配文件时加载到上下文中。对于不需要始终在上下文中的特定任务指令，改用 [skills](/zh-CN/skills)，它仅在你调用它们或 Claude 确定它们与你的提示相关时加载。
</Note>

#### 设置规则

在你的项目的 `.claude/rules/` 目录中放置 markdown 文件。每个文件应涵盖一个主题，具有描述性文件名，如 `testing.md` 或 `api-design.md`。所有 `.md` 文件都被递归发现，因此你可以将规则组织到子目录中，如 `frontend/` 或 `backend/`：

```text  theme={null}
your-project/
├── .claude/
│   ├── CLAUDE.md           # 主项目指令
│   └── rules/
│       ├── code-style.md   # 代码样式指南
│       ├── testing.md      # 测试约定
│       └── security.md     # 安全要求
```

没有 [`paths` frontmatter](#path-specific-rules) 的规则在启动时加载，优先级与 `.claude/CLAUDE.md` 相同。

#### 特定路径的规则

规则可以使用带有 `paths` 字段的 YAML frontmatter 范围限定到特定文件。这些条件规则仅在 Claude 处理与指定模式匹配的文件时适用。

```markdown  theme={null}
---
paths:
  - "src/api/**/*.ts"
---

# API 开发规则

- 所有 API 端点必须包括输入验证
- 使用标准错误响应格式
- 包括 OpenAPI 文档注释
```

没有 `paths` 字段的规则无条件加载并适用于所有文件。路径范围规则在 Claude 读取与模式匹配的文件时触发，而不是在每次工具使用时。

在 `paths` 字段中使用 glob 模式按扩展名、目录或任何组合匹配文件：

| 模式                     | 匹配                     |
| ---------------------- | ---------------------- |
| `**/*.ts`              | 任何目录中的所有 TypeScript 文件 |
| `src/**/*`             | `src/` 目录下的所有文件        |
| `*.md`                 | 项目根目录中的 Markdown 文件    |
| `src/components/*.tsx` | 特定目录中的 React 组件        |

你可以指定多个模式并使用大括号扩展在一个模式中匹配多个扩展名：

```markdown  theme={null}
---
paths:
  - "src/**/*.{ts,tsx}"
  - "lib/**/*.ts"
  - "tests/**/*.test.ts"
---
```

#### 使用符号链接跨项目共享规则

`.claude/rules/` 目录支持符号链接，因此你可以维护一组共享规则并将它们链接到多个项目中。符号链接被解析并正常加载，循环符号链接被检测并优雅处理。

此示例链接共享目录和单个文件：

```bash  theme={null}
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

#### 用户级规则

`~/.claude/rules/` 中的个人规则适用于你机器上的每个项目。使用它们来处理不是项目特定的偏好：

```text  theme={null}
~/.claude/rules/
├── preferences.md    # 你的个人编码偏好
└── workflows.md      # 你的首选工作流
```

用户级规则在项目规则之前加载，给予项目规则更高的优先级。

### 为大型团队管理 CLAUDE.md

对于在团队中部署 Claude Code 的组织，你可以集中指令并控制加载哪些 CLAUDE.md 文件。

#### 部署组织范围的 CLAUDE.md

组织可以部署一个集中管理的 CLAUDE.md，适用于机器上的所有用户。此文件不能被个人设置排除。

<Steps>
  <Step title="在托管策略位置创建文件">
    * macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
    * Linux 和 WSL: `/etc/claude-code/CLAUDE.md`
    * Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`
  </Step>

  <Step title="使用你的配置管理系统部署">
    使用 MDM、Group Policy、Ansible 或类似工具在开发者机器上分发文件。有关其他组织范围配置选项，请参阅 [托管设置](/zh-CN/permissions#managed-settings)。
  </Step>
</Steps>

#### 排除特定的 CLAUDE.md 文件

在大型 monorepos 中，祖先 CLAUDE.md 文件可能包含与你的工作无关的指令。`claudeMdExcludes` 设置让你按路径或 glob 模式跳过特定文件。

此示例排除顶级 CLAUDE.md 和来自父文件夹的规则目录。将其添加到 `.claude/settings.local.json` 以使排除保持本地到你的机器：

```json  theme={null}
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

模式使用 glob 语法与绝对文件路径匹配。你可以在任何 [设置层](/zh-CN/settings#settings-files)：用户、项目、本地或托管策略配置 `claudeMdExcludes`。数组跨层合并。

托管策略 CLAUDE.md 文件不能被排除。这确保组织范围指令始终适用，无论个人设置如何。

## 自动记忆

自动记忆让 Claude 跨会话积累知识，无需你编写任何内容。Claude 在工作时为自己保存笔记：构建命令、调试见解、架构笔记、代码样式偏好和工作流习惯。Claude 不会每个会话都保存内容。它根据信息在未来对话中是否有用来决定什么值得记住。

<Note>
  自动记忆需要 Claude Code v2.1.59 或更高版本。使用 `claude --version` 检查你的版本。
</Note>

### 启用或禁用自动记忆

自动记忆默认开启。要切换它，在会话中打开 `/memory` 并使用自动记忆切换，或在你的项目设置中设置 `autoMemoryEnabled`：

```json  theme={null}
{
  "autoMemoryEnabled": false
}
```

要通过环境变量禁用自动记忆，设置 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。

### 存储位置

每个项目在 `~/.claude/projects/<project>/memory/` 获得自己的记忆目录。`<project>` 路径来自 git 存储库，因此同一存储库中的所有 worktrees 和子目录共享一个自动记忆目录。在 git 存储库外，改用项目根目录。

要将自动记忆存储在不同位置，在你的用户或本地设置中设置 `autoMemoryDirectory`：

```json  theme={null}
{
  "autoMemoryDirectory": "~/my-custom-memory-dir"
}
```

此设置从策略、本地和用户设置接受。它不从项目设置（`.claude/settings.json`）接受，以防止共享项目将自动记忆写入重定向到敏感位置。

目录包含一个 `MEMORY.md` 入口点和可选的主题文件：

```text  theme={null}
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 简洁索引，加载到每个会话
├── debugging.md       # 关于调试模式的详细笔记
├── api-conventions.md # API 设计决策
└── ...                # Claude 创建的任何其他主题文件
```

`MEMORY.md` 充当记忆目录的索引。Claude 在你的会话中读取和写入此目录中的文件，使用 `MEMORY.md` 跟踪存储的内容。

自动记忆是机器本地的。同一 git 存储库中的所有 worktrees 和子目录共享一个自动记忆目录。文件不在机器或云环境之间共享。

### 它如何工作

`MEMORY.md` 的前 200 行在每次对话开始时加载。第 200 行之外的内容在会话开始时不加载。Claude 通过将详细笔记移到单独的主题文件中来保持 `MEMORY.md` 简洁。

此 200 行限制仅适用于 `MEMORY.md`。CLAUDE.md 文件无论长度如何都完整加载，尽管较短的文件产生更好的遵守度。

主题文件如 `debugging.md` 或 `patterns.md` 在启动时不加载。Claude 在需要信息时使用其标准文件工具按需读取它们。

Claude 在你的会话中读取和写入记忆文件。当你在 Claude Code 界面中看到"Writing memory"或"Recalled memory"时，Claude 正在主动更新或读取 `~/.claude/projects/<project>/memory/`。

### 审计和编辑你的记忆

自动记忆文件是纯 markdown，你可以随时编辑或删除。运行 [`/memory`](#view-and-edit-with-memory) 从会话中浏览和打开记忆文件。

## 使用 `/memory` 查看和编辑

`/memory` 命令列出在你当前会话中加载的所有 CLAUDE.md 和规则文件，让你切换自动记忆开或关，并提供打开自动记忆文件夹的链接。选择任何文件在你的编辑器中打开它。

当你要求 Claude 记住某些内容时，如"总是使用 pnpm，而不是 npm"或"记住 API 测试需要本地 Redis 实例"，Claude 将其保存到自动记忆。要改为添加指令到 CLAUDE.md，直接要求 Claude，如"将其添加到 CLAUDE.md"，或通过 `/memory` 自己编辑文件。

## 故障排除记忆问题

这些是 CLAUDE.md 和自动记忆最常见的问题，以及调试步骤。

### Claude 不遵循我的 CLAUDE.md

CLAUDE.md 内容作为用户消息在系统提示之后传递，而不是系统提示本身的一部分。Claude 读取它并尝试遵循它，但没有严格遵守的保证，特别是对于模糊或冲突的指令。

要调试：

* 运行 `/memory` 验证你的 CLAUDE.md 文件被加载。如果文件未列出，Claude 看不到它。
* 检查相关 CLAUDE.md 是否在为你的会话加载的位置（参见 [选择 CLAUDE.md 文件的位置](#choose-where-to-put-claudemd-files)）。
* 使指令更具体。"使用 2 空格缩进"比"格式化代码很好"效果更好。
* 查找跨 CLAUDE.md 文件的冲突指令。如果两个文件为相同行为提供不同的指导，Claude 可能会任意选择一个。

对于你想要在系统提示级别的指令，使用 [`--append-system-prompt`](/zh-CN/cli-reference#system-prompt-flags)。这必须在每次调用时传递，因此它更适合脚本和自动化而不是交互式使用。

<Tip>
  使用 [`InstructionsLoaded` hook](/zh-CN/hooks#instructionsloaded) 记录确切加载了哪些指令文件、何时加载以及为什么。这对于调试特定路径规则或子目录中的延迟加载文件很有用。
</Tip>

### 我不知道自动记忆保存了什么

运行 `/memory` 并选择自动记忆文件夹来浏览 Claude 保存的内容。一切都是纯 markdown，你可以读取、编辑或删除。

### 我的 CLAUDE.md 太大了

超过 200 行的文件消耗更多上下文并可能降低遵守度。将详细内容移到使用 `@path` 导入引用的单独文件中（参见 [导入其他文件](#import-additional-files)），或将你的指令分割到 `.claude/rules/` 文件中。

### 在 `/compact` 后指令似乎丢失了

CLAUDE.md 完全在压缩中存活。在 `/compact` 之后，Claude 从磁盘重新读取你的 CLAUDE.md 并将其新鲜重新注入会话。如果指令在压缩后消失，它仅在对话中给出，未写入 CLAUDE.md。将其添加到 CLAUDE.md 以使其在会话中持久化。

有关大小、结构和具体性的指导，请参阅 [编写有效的指令](#write-effective-instructions)。