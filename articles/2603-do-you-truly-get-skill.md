# 兄弟！你真的懂 Skill 吗？

> - **Published:** 2026-03-17
> - **Source:** [兄弟！你真的懂 Skill 吗？](https://mp.weixin.qq.com/s?__biz=MzI2NDU4OTExOQ==&mid=2247694816&idx=1&sn=5d796cf823344314aa4dd99163ceef5c&poc_token=HHQDummj70ZO_NuTpa5rVUXJZz5wB6fmSovmIVKv)
> - **Tags:** `Skill`

## 01 前言：Skill 到底怎么"跑起来"的？

如果你关注过 Claude Code、Cursor 或 CodeBuddy 这类 AI 编程助手，大概率听说过"Skill"这个概念——给 AI 一个技能包，它就能处理 PDF、做 PPT、写前端。

但一个更根本的问题是：**这些技能包到底是怎么驱动执行的？**

是 function calling？是 API 注册？还是某种尚未被广泛讨论的机制？

Anthropic 开源了 16 个官方 Skill。本文的工作是：逐个分析它们的目录结构、SKILL.md 内容和执行方式，同时结合对整个 Skill 框架源码（9 个核心文件，约 2000+ 行代码）的逐文件拆解，回答一个问题：**Skill 系统的执行模式到底有几种？**

本文覆盖了以下分析工作：

- **框架层：** 逐文件分析 `_types.py` （数据模型）→ `_repository.py`（技能仓库）→ `_tools.py`（6 个核心工具函数）→ `_toolset.py` + `_dynamic_toolset.py`（Token 优化）→ `_run_tool.py`（沙箱执行）→ `_skill_processor.py`（请求注入器），梳理了从 SKILL.md 声明到 LLM 真正发起 function_call 并执行的完整 7 步链路
- **应用层：** 16 个 Skill 的目录结构和 SKILL.md 内容逐个分析，归类执行方式
- **归纳层：** 从框架代码的执行路径出发，看每种 Skill 触发了框架的哪些组件，提炼出 5 种模式

## 02 先看全景：16 个 Skill 一览

| Skill | 有 scripts/ ？ | 有参考文档？ | 核心执行方式 |
| --- | --- | --- | --- |
| **pdf** | ✅ | ✅ | 脚本执行 + 参考文档 |
| **pptx** | ✅ | ✅ | 脚本执行 + 参考文档 |
| **xlsx** | ✅ | ❌ | 脚本执行 |
| **docx** | ✅ | ✅ | 脚本执行 |
| **webapp-testing** | ✅ | ✅ | 脚本执行 |
| **frontend-design** | ❌ | ❌ | 纯 Prompt 注入 |
| **brand-guidelines** | ❌ | ❌ | 纯 Prompt 注入 |
| **algorithmic-art** | ❌ | ❌ | 纯 Prompt 注入 |
| **doc-coauthoring** | ❌ | ❌ | 纯 Prompt 注入 |
| **internal-comms** | ❌ | ❌ | 纯 Prompt 注入 |
| **web-artifacts-builder** | ❌ | ❌ | 纯 Prompt 注入 |
| **canvas-design** | ❌ | ❌ | 纯 Prompt + 资源 |
| **theme-factory** | ❌ | ❌ | 纯 Prompt + 资源 |
| **slack-gif-creator** | ❌ (有 core/) | ❌ | 库调用型 |
| **mcp-builder** | ✅ | ✅ | 参考文档 + 编排 |
| **skill-creator** | ✅ | ✅ | 编排型（含子 Agent） |

值得注意的是：16 个 Skill 中，没有一个使用了 Tools: 声明来注册 function calling。

全部都是通过 SKILL.md 的文本内容 + skill_run 沙箱命令来驱动执行的。

这说明什么？Anthropic 自己在实践中告诉我们：与其给 AI 注册新函数，不如教它怎么写代码。

### 2.1 先说一下背景知识：框架怎么理解一个 Skill

在深入五种模式之前，我先用最简洁的方式交代一下 Skill 框架的核心链路，否则后面的分析会看不懂。

我拆了整个框架源码，核心组件就这么几个：

```css
SKILL.md  →  FsSkillRepository（扫描、解析）
          →  Skill 对象（name, description, body, tools, resources）
          →  SkillToolSet（6 个管理工具 + skill_run）
          →  DynamicSkillToolSet（按需加载业务工具）
          →  SkillsRequestProcessor（注入 system prompt）
```

LLM 使用 Skill 的完整流程是这样的：

```python
1. LLM 调用 skill_list()  → 看到所有技能的 name + description（~30 Token/个）
2. LLM 调用 skill_load("pdf") → 触发 state_delta 写入，SKILL.md body 注入 system prompt
3. LLM 调用 skill_select_docs(docs=["forms.md"]) → 按需加载详细文档
4. LLM 调用 skill_run(command="python3 scripts/xxx.py") → 在沙箱中执行命令
```

关键设计：**所有 skill_xxx 工具只修改 state_delta（一种临时状态），真正的内容注入在 SkillsRequestProcessor 中完成**——这是一种 CQRS（命令查询分离）架构。写入方和读取方通过 state_delta 解耦，各自独立演化。

另一个关键概念是 `skill_run` 的沙箱执行。它不是在 Agent 进程里跑 Python 代码，而是：

```bash
1. 创建隔离工作空间 /tmp/ws_xxx/
2. 将技能目录复制到工作空间（增量哈希优化，不重复拷贝）
3. 自动注入环境变量：$WORK_DIR、$OUTPUT_DIR、$SKILLS_DIR
4. bash -lc "python3 scripts/xxx.py" 在隔离目录中执行
5. 收集 stdout + 指定的输出文件，返回给 LLM
```

工作空间布局长这样：

```bash
/tmp/ws_session123/
├── skills/pdf/           ← 技能目录（只读保护）
│   ├── SKILL.md
│   ├── scripts/
│   ├── out/  → ../../out     ← 符号链接
│   └── work/ → ../../work    ← 符号链接
├── out/                  ← $OUTPUT_DIR
├── work/                 ← $WORK_DIR
└── runs/                 ← 执行记录
```

技能文件只读保护 + 输出目录隔离 + 可选 Docker 容器执行 = **生产级的安全沙箱。**

### 2.2 深入 skill_run：整个系统最核心的 200 行代码
  
`skill_run` 是五种模式中四种都依赖的核心执行引擎——整个 Skill 系统的"手脚"。以下结合源码展开分析。

它在框架里长什么样

`skill_run` 不是一个普通的函数，它是一个完整的 Tool 类——`SkillRunTool`，继承自框架的 `BaseTool`。LLM 看到的 function calling Schema 是这样的：

```python
class SkillRunTool(BaseTool):
    def _get_declaration(self) -> FunctionDeclaration:
        return FunctionDeclaration(
            name="skill_run",
            description="Run a command inside a skill workspace. "
                        "Stages the entire skill directory and runs a single command.",
            parameters=Schema.model_validate(SkillRunInput.model_json_schema()),
            response=Schema.model_validate(SkillRunOutput.model_json_schema()),
        )
```

LLM 调用时传入的参数，对应一个精心设计的输入模型：

```python
class SkillRunInput(BaseModel):
    skill: str                           # 必需：技能名称（"pdf"、"pptx"...）
    command: str                         # 必需：要执行的 shell 命令
    cwd: str = ""                        # 可选：工作目录（相对于技能根目录）
    env: dict[str, str] = {}             # 可选：自定义环境变量
    output_files: list[str] = []         # 可选：输出文件的 glob 模式
    timeout: int = 0                     # 可选：超时时间（秒）
    inputs: list[WorkspaceInputSpec] = [] # 可选：声明式输入映射
    save_as_artifacts: bool = False      # 可选：是否持久化输出文件
```

这个参数模型的设计相当老练。几个点值得注意：

1. `output_files` 支持 glob 模式——你可以写 `["$OUTPUT_DIR/*.png"]`，框架会自动收集所有匹配的文件返回给 LLM。这比"你必须提前知道输出文件叫什么"灵活太多了。
2. `inputs` 是声明式输入映射——不用 LLM 自己拼 `cp` 命令把用户文件复制到工作空间，直接声明"我需要用到这个文件"，框架帮你搬。
3. `save_as_artifacts`——输出文件默认是临时的（对话结束就没了），设为 true 可以持久化。这个选项给了 LLM 选择权：分析报告要持久化，中间结果不需要。

核心执行流程：6 步，一步不多

`_run_async_impl` 是整个 Skill 系统最核心的方法。当 LLM 发出 `skill_run(...)` 调用后，执行逻辑如下：

```python
async def _run_async_impl(self, *, tool_context, args):
    inputs = SkillRunInput.model_validate(args)

    # ═══ 第1步：定位技能目录 ═══
    repository = self._get_repository(tool_context)
    skill_root = repository.path(inputs.skill)
    # → "/path/to/skills/pdf"

    # ═══ 第2步：创建/获取隔离工作空间 ═══
    workspace_runtime = repository.workspace_runtime
    manager = workspace_runtime.manager(tool_context)
    ws = await manager.create_workspace(
        tool_context, session_id, WorkspacePolicy()
    )
    # → WorkspaceInfo(id="ws-abc", path="/tmp/ws_session123_1706000000")

    # ═══ 第3步：暂存技能目录到工作空间（增量优化）═══
    await self._stage_skill(tool_context, ws, skill_root, inputs.skill)
    # 做了 5 件事：
    #   a. compute_dir_digest(skill_root)  → 计算目录哈希
    #   b. 如果哈希没变 → return（直接跳过！）
    #   c. fs.stage_directory → 复制技能文件到工作空间
    #   d. _link_workspace_dirs → 创建符号链接
    #   e. _read_only_except_symlinks → 设置只读保护

    # ═══ 第4步：处理声明式输入文件 ═══
    if inputs.inputs:
        fs = workspace_runtime.fs(tool_context)
        await fs.stage_inputs(tool_context, ws, inputs.inputs)

    # ═══ 第5步：🔥 真正执行命令 ═══
    cwd = self._resolve_cwd(inputs.cwd, inputs.skill)
    # cwd 默认是 "skills/<skill_name>"
    result = await self._run_program(tool_context, ws, cwd, inputs)
    # 内部调用：bash -lc "python3 scripts/analyze.py ..."
    # 在 /tmp/ws_xxx/skills/pdf/ 目录下执行
    # 自动注入 $WORK_DIR、$OUTPUT_DIR 等环境变量

    # ═══ 第6步：收集输出文件 ═══
    files, manifest = await self._prepare_outputs(
        tool_context, ws, inputs
    )

    return SkillRunOutput(
        stdout=result.stdout,
        stderr=result.stderr,
        exit_code=result.exit_code,
        timed_out=result.timed_out,
        duration_ms=int(result.duration),
        output_files=files,
    ).model_dump()
```

  

六步，每一步职责清晰，没有冗余。

`_stage_skill`**：增量暂存的精妙设计**

整个 `_run_tool.py` 中，`_stage_skill` 的增量暂存逻辑最值得分析。一次对话中，LLM 可能连续调用 3-4 次 `skill_run`（安装依赖 → 分析数据 → 生成图表 → 导出报告）。如果每次都完整复制技能目录，性能开销不可忽视。源码如下：

```python

async def _stage_skill(self, ctx, ws, skill_root, skill_name):
    # 计算技能目录的哈希（基于所有文件内容）
    digest = compute_dir_digest(skill_root)

    # 加载工作空间元数据
    metadata = load_metadata(ws.path)

    # 🔥 如果这个技能已暂存且哈希相同 → 直接跳过
    if (metadata.skills.get(skill_name) and 
        metadata.skills[skill_name].digest == digest):
        return  # 零开销！

    # 否则执行完整暂存
    fs = workspace_runtime.fs(ctx)
    dst = Path("skills") / skill_name
    await fs.stage_directory(
        ctx, ws, skill_root, dst.as_posix(), WorkspaceStageOptions()
    )

    # 创建便捷符号链接：out/ → ../../out, work/ → ../../work
    await self._link_workspace_dirs(ctx, ws, skill_name)

    # 设置只读保护（防止脚本意外修改技能源文件）
    await self._read_only_except_symlinks(ctx, ws, dst.as_posix())

    # 更新元数据（记录哈希，下次检查用）
    metadata.skills[skill_name] = SkillMetadata(
        name=skill_name, digest=digest, 
        mounted=True, staged_at=time.time()
    )
    save_metadata(ws.path, metadata)
```

为什么这段代码设计得好？ 三个原因：

第一，增量哈希让"重复调用零开销"。 `compute_dir_digest` 遍历技能目录的所有文件，计算一个综合哈希。如果没有文件变化，哈希不变，直接 return。对于 pdf 这样有 8 个脚本的技能，第 2-4 次 `skill_run` 完全跳过目录拷贝，节省的 IO 很可观。

第二，符号链接让脚本"无感知"地访问共享目录。 脚本代码里写 `open("out/report.txt")` 和写 `open("$OUTPUT_DIR/report.txt")` 效果一样——因为 `out/` 就是指向 `$OUTPUT_DIR` 的符号链接。脚本不需要知道自己在沙箱里运行，它只看到一个正常的目录结构。这是"效果等价而非形式等价"原则的典型实践——脚本以为自己在一个正常项目里，实际上在一个精心隔离的沙箱里。

第三，只读保护防止意外篡改。 `_read_only_except_symlinks` 把技能目录下所有文件设为只读——除了符号链接指向的 `out/` 和 `work/`。这意味着脚本可以自由写输出文件，但不能修改技能自身的代码。如果一个有 bug 的脚本试图 `open("SKILL.md", "w")`，会直接报 `PermissionError`。这个防御层在生产环境中非常重要。

自动注入的环境变量

`_run_program` 执行命令前，会自动注入一组环境变量：

| 变量 | 指向 | 让脚本可以... |
| --- | --- | --- |
| `$WORKSPACE_DIR` | `/tmp/ws_session123_...` | 访问工作空间根目录 |
| `$SKILLS_DIR` | `$WORKSPACE_DIR/skills` | 引用其他技能的文件 |
| `$WORK_DIR` | `$WORKSPACE_DIR/work` | 存放中间文件 |
| `$OUTPUT_DIR` | `$WORKSPACE_DIR/out` | 存放最终输出 |
| `$RUN_DIR` | `$WORKSPACE_DIR/runs/run_xxx` | 访问运行记录 |
| `$SKILL_NAME` | `pdf` | 知道自己是哪个技能 |

这套环境变量的设计让脚本完全不需要硬编码路径。 比如 pdf 技能的脚本里写的是 `os.environ["OUTPUT_DIR"]` 而不是 `/tmp/ws_xxx/out/`——这意味着同一个脚本在本地测试和 Docker 容器里都能跑，零修改。

两种运行时：本地 vs 容器

`WorkspaceRuntime` 是一个抽象层，有两个实现：

```python
# 开发/测试环境：本地执行
workspace_runtime = LocalWorkspaceRuntime()
# → 在 /tmp/ws_xxx/ 中用 bash -lc 执行
# → 技能文件只读保护
# → 共享宿主机的 Python、pip 等工具

# 生产环境：Docker 容器隔离
workspace_runtime = ContainerWorkspaceRuntime(
    image="python:3.11-slim"
)
# → 在独立 Docker 容器中执行
# → 技能目录只读挂载
# → 默认禁用网络
# → 完全隔离的文件系统
```

**为什么需要两种？** 因为开发时你不想每次测试都等 Docker 启动，本地模式秒开。但生产环境必须用容器——因为 `skill_run` 的 command 参数是 LLM 构造的，理论上 LLM 可能生成危险命令（rm -rf / 之类的）。容器隔离确保即使脚本失控也不影响宿主机。

这也是为什么框架用 `shell_quote()` 做命令安全处理——`_utils.py` 里的这个函数会转义特殊字符，防止命令注入。这是沙箱安全的第一道防线，容器隔离是第二道。

`skill_run` **为什么是"万能兜底"**

回过头看，`skill_run` 的威力在于它的极简抽象：

```
输入：一个技能名 + 一条 shell 命令
输出：stdout + stderr + exit_code + 输出文件
```

这就是全部接口。 不管你的技能是 Python 写的、Node.js 写的、Go 编译的二进制、还是一个 Shell 脚本——只要能用 `bash -lc` 启动，`skill_run` 就能执行。

这比 function calling 灵活得多：

- function calling 要求你为每个操作定义 **精确的 JSON Schema**（参数名、类型、描述）
- `skill_run` 只需要你在 SKILL.md 里用自然语言描述"这个脚本怎么用"

**前者是 API 设计，后者是写文档。** 对于大多数技能作者来说，写文档的门槛远低于设计 API Schema。而且 LLM 理解自然语言文档的能力，远强于理解结构化 Schema——这是 Anthropic 选择"教 LLM 写代码"而非"注册 function calling"的根本原因。

关键问题：模型到底是怎么知道要调 skill_run 的？

这可能是整个 Skill 系统最值得深究的问题——LLM 从来没读过框架源码，它怎么知道"我应该调 skill_run"，参数该填什么，command 该写什么？

答案是：**三层信息注入形成了一个完整的闭环。**

**第一层：工具 Schema 注入（LLM 看到 `skill_run` 这个工具的存在）**

当 Agent 启动时，`SkillToolSet` 会把 `skill_run` 作为一个固有工具注册进 LLM 的可用工具列表。LLM 每轮请求时，都会在 tools 参数里看到 `skill_run` 的 FunctionDeclaration：

```json
{
  "name": "skill_run",
  "description": "Run a command inside a skill workspace. Stages the entire skill directory and runs a single command.",
  "parameters": {
    "skill": {"type": "string", "description": "技能名称"},
    "command": {"type": "string", "description": "要执行的 shell 命令"},
    "output_files": {"type": "array", "description": "输出文件的 glob 模式"},
    ...
  }
}
```

这一步只告诉 LLM **"你有 `skill_run` 这个工具可以用"**。但 LLM 还不知道该用它来干什么、command 该填什么。

**第二层：System Prompt 中的行为指引（告诉 LLM "什么时候该用 `skill_run`"）**

还记得前面 `_inject_overview` 的源码吗？它除了注入技能列表，还注入了一段操作指引：

```python
instruction = f"""Available skills:{skill_instructions}Tooling and workspace guidance:- Skills run inside an isolated workspace...- Prefer $SKILLS_DIR, $WORK_DIR, $OUTPUT_DIR... over hard-coded paths- If a skill is not loaded, call skill_load- When body and needed docs/tools are present, call skill_run or use tools directly"""
```

注意最后一句：`When body and needed docs/tools are present, call skill_run or use tools directly`。这句话直接告诉 LLM——"当你已经加载了技能的 body（SKILL.md 内容），就可以调用 `skill_run` 了"。

这一步建立了 **"加载技能 → 阅读说明 → 用 `skill_run` 执行"** 的行为链条。但 LLM 还不知道 command 参数具体该写什么。

**第三层：SKILL.md body 注入（告诉 LLM "command 具体怎么填"）**

当 LLM 调用 `skill_load("pdf")` 之后，`SkillsRequestProcessor` 会把 SKILL.md 的 body 文本完整注入到 system prompt 中。而 SKILL.md 的 body 里写的恰恰就是**脚本使用说明：**

```bash
## 可用脚本

- `scripts/extract_form_structure.py` - 提取 PDF 表单结构
  用法：python3 scripts/extract_form_structure.py <input_pdf>

- `scripts/fill_fillable_fields.py` - 填充 PDF 表单字段
  用法：python3 scripts/fill_fillable_fields.py <input_pdf> <data_json>
  输出到 $OUTPUT_DIR/filled_output.pdf

### 依赖安装
首次使用时运行：pip install -r scripts/requirements.txt
```

LLM 读到这段文字后，就知道了：

- 有哪些脚本可用
- 每个脚本的命令行格式是什么
- 参数怎么传、输出到哪里

**三层合在一起形成完整闭环：**
```bash
第1层  工具 Schema    →  LLM 知道：有 skill_run 这个工具
第2层  System Prompt  →  LLM 知道：加载技能后用 skill_run 执行
第3层  SKILL.md body  →  LLM 知道：command 参数该写 "python3 scripts/xxx.py ..."
```

所以当用户说"帮我填写这个 PDF 表单"时，LLM 的推理链条是：

```bash
1. 看 system prompt 里的技能列表 → 发现 "pdf" 技能匹配
2. 调 skill_load("pdf") → SKILL.md body 注入 system prompt
3. 阅读 body → 发现 scripts/extract_form_structure.py 可以分析表单结构
4. 拼装 command："python3 scripts/extract_form_structure.py input.pdf"
5. 调 skill_run(skill="pdf", command="python3 scripts/extract_form_structure.py input.pdf")
6. 读返回的 stdout → 知道表单有哪些字段
7. 再调 skill_run 执行填充脚本
```

**本质上，SKILL.md 的 body 就是"给 LLM 看的使用手册"**。 框架只负责两件事：在正确的时机把这份手册塞进 system prompt，以及提供 `skill_run` 这个万能执行器。至于 LLM 读完手册后怎么组装命令、以什么顺序调用——全靠 LLM 自己的理解能力。

这就是为什么 **SKILL.md 的 body 质量**是整个系统的关键：如果使用说明写得不清楚（比如没写参数格式、没写依赖安装步骤），LLM 就会像一个拿到烂文档的工程师一样——要么瞎猜，要么报错。而如果写得好（像 Anthropic 的 pdf 技能那样，每个脚本都有完整的用法示例），LLM 就能准确执行。

好，有了这个基础，我们来看五种模式。

## 03 模式一：纯 Prompt 注入型

代表：`frontend-design`、`brand-guidelines`、`algorithmic-art`

一句话概括：Skill = 一段精心编写的 system prompt。

### 3.1 结构

```
frontend-design/
└── SKILL.md          ← 唯一的内容文件
```

没有 scripts、没有 tools、没有任何外部依赖。整个 Skill 就是一个 Markdown 文件。

### 3.2 SKILL.md 里写了什么

```markdown
# 设计思考
在编码之前，先理解上下文并致力于一个 BOLD 美学方向：
- **目的**: 此界面解决什么问题？谁在使用它？
- **基调**: 选择一个极端：粗犷极简、复古未来主义、奢华精致...
- **差异化**: 什么让这个令人难忘？

## 前端美学指南
- 选择美观独特的字体，避免 Arial/Inter...
- 使用 CSS 变量保持一致性...
- **永远不要** 使用通用 AI 生成的美学...
```

### 3.3 执行流程

```
用户: "帮我做一个科技感的落地页"
  │
  ▼ LLM 匹配到 frontend-design，加载 SKILL.md
  ▼ SKILL.md body 注入 system message
  ▼ LLM 根据注入的美学指南，直接写 HTML/CSS/React
  ▼ 通过 write_to_file 输出代码
  │
用户得到一个有设计感的落地页
```

### 3.4 关键洞察

**没有任何外部执行。** Skill 的全部价值在于三件事：

1. 提供领域专业知识（设计原则、色彩理论）
2. 约束 LLM 行为（"永远不要用通用 AI 美学"）
3. 引导思维流程（"编码前先理解上下文"）

这其实是 Prompt Engineering 的最佳实践——只不过包装成了可复用的技能包。

我的理解：这种模式看着最简单，但写好 SKILL.md 的难度不低。需要把一个资深前端设计师的审美品味、设计方法论、反面案例全浓缩进一份 Markdown 里，而且要用 LLM 能理解并遵循的方式表达。本质上，是在把"隐性知识"编码成"显式规则"——这就是为什么 Anthropic 把 `frontend-design` 的 SKILL.md 写到了 4.4KB，而不是草草几行。

在框架中的映射：

- `_parse_tools_from_body()` → 返回 `[]`（没有 Tools 声明）
- `DynamicSkillToolSet.get_tools()` → 不返回额外工具
- `SkillsRequestProcessor` → 仅注入 body 文本到 system prompt
- 无 `skill_run` 调用
- LLM 使用 Agent 自带的基础工具（`write_to_file`、`read_file` 等）完成任务

## 04 模式二：脚本执行型

代表：`pdf`、`pptx`、`xlsx`、`webapp-testing`

一句话概括：SKILL.md 当教程，scripts/ 当工具箱。

### 4.1 结构

```
pdf/
├── SKILL.md              ← 使用手册 + 代码示例
├── forms.md              ← 表单填写指南
├── reference.md          ← 高级参考
└── scripts/
    ├── extract_form_structure.py
    ├── fill_fillable_fields.py
    ├── convert_pdf_to_images.py
    └── ... (共 8 个脚本)
```

### 4.2 核心设计

SKILL.md 是一份 **Python PDF 操作的完整教程**——用什么库、怎么用、示例代码。但这些代码示例不是给机器跑的，是 **教 LLM 自己怎么写代码** 的。

而 `scripts/` 下的脚本才是给 `skill_run` 执行的预制工具。

### 4.3 执行流程

```
用户: "帮我填写这个 PDF 表单"
  │
  ▼ LLM 加载 SKILL.md，了解 PDF 操作全景
  ▼ 发现需要填表 → 按需加载 forms.md
  ▼ 按照文档指引，调用预制脚本：
  │
  │   skill_run("python3 scripts/extract_form_structure.py input.pdf")
  │   → 返回: "Found 12 fields..."
  │
  │   skill_run("python3 scripts/fill_fillable_fields.py input.pdf data.json")
  │   → 返回: 填好的 PDF
  │
用户得到填好的 PDF 表单
```

### 4.4 两种代码的区别

|  | SKILL.md 中的代码示例 | scripts/ 预制脚本 |
| --- | --- | --- |
| **用途** | 教 LLM 怎么写代码 | 直接执行的黑盒工具 |
| **谁执行** | LLM 自己写 + Agent 基础工具 | `skill_run`  在沙箱中执行 |
| **适合** | 简单一次性操作 | 复杂、需要验证的工作流 |

这里的精妙之处在于：**SKILL.md body 既是文档也是 prompt**。LLM 读了这份"教程"之后，简单任务自己写代码搞定，复杂任务知道该调哪个预制脚本。

**一个值得注意的源码细节：** `skill_run` 执行预制脚本时，框架会先调 _stage_skill 把技能目录复制到沙箱，并且用 compute_dir_digest() **做增量哈希**——如果技能文件没变就直接跳过。这意味着同一次对话中多次调用 `skill_run`（先安装依赖、再分析表单、再填写），只有第一次会做目录拷贝，后续几乎零开销。

**另外值得注意的是 Token 效率。**模式二用的是 `skill_run`，它在 LLM 的每轮请求中只占一个工具 Schema（约 20 Token），而不是为每个脚本都注册一个 function calling（每个约 200-500 Token）。8 个脚本如果都注册为 function calling，每轮请求就多出 1600-4000 Token——10 轮对话下来是 1.6 万到 4 万 Token 的额外消耗。用 `skill_run` 统一执行，LLM 通过阅读 SKILL.md 来判断该调哪个脚本，经济得多。

## 05 模式三：库调用型

代表：`slack-gif-creator`

一句话概括：Skill 自带 Python 库，LLM 现场写代码 import 它。

### 5.1 结构

```
slack-gif-creator/
├── SKILL.md              ← API 文档 + 使用示例
├── requirements.txt      ← 依赖声明
└── core/                 ← Python 库（不是 scripts/！）
    ├── gif_builder.py    ← GIFBuilder 类
    ├── validators.py     ← validate_gif, is_slack_ready
    ├── easing.py         ← 缓动函数
    └── frame_composer.py ← 帧生成辅助
```

注意 `core/` 而非 `scripts/`——这不是一堆独立脚本，而是一个 Python 库。


### 5.2 执行流程

```python
用户: "做一个心跳动画的 Slack emoji GIF"
  │
  ▼ LLM 加载 SKILL.md，理解 API
  ▼ LLM 自己编写一个完整脚本：
  │
  │   from core.gif_builder import GIFBuilder
  │   from PIL import Image, ImageDraw
  │   import math
  │
  │   builder = GIFBuilder(width=128, height=128, fps=10)
  │   for i in range(20):
  │       scale = 0.8 + 0.4 * abs(math.sin(i/19 * math.pi * 2))
  │       frame = Image.new('RGB', (128, 128), (255, 240, 240))
  │       # ... 绘制心形 + 缩放 ...
  │       builder.add_frame(frame)
  │   builder.save('heartbeat.gif', optimize_for_emoji=True)
  │
  ▼ skill_run 执行这个 LLM 写的脚本
  │   （core/ 已被复制到工作空间，可直接 import）
  │
用户得到优化过的 Slack emoji GIF
```

### 5.3 与脚本执行型的关键区别

|  | 脚本执行型（pdf） | 库调用型（slack-gif-creator） |
| --- | --- | --- |
| 文件结构 | `scripts/xxx.py`  （独立可执行） | `core/xxx.py`  （Python 模块） |
| 调用方式 | `skill_run("python3 scripts/xxx.py")` | LLM 自己写脚本 `import core.xxx` |
| LLM 角色 | 调用者（跑预制脚本） | 开发者（组合库函数写新代码） |

**SKILL.md body 在这里本质上是 API 文档**——教 LLM 怎么使用 `GIFBuilder`、`validate_gif` 这些函数。LLM 不是执行预制脚本，而是 **现场编写自定义脚本**，组合 Skill 提供的库函数来完成任务。

**这种模式值得重点关注**，它把 Skill 系统和 LLM 代码能力的结合推到了极致。模式二中 LLM 是"调用者"，模式三中 LLM 是"开发者"——它不只是按说明书操作，而是理解了库的 API 后，自己设计算法、组合函数、生成新代码。这类似于给一个工程师一个写好的 SDK：文档写清楚了，他就能用这个 SDK 写出各种应用。

**框架层面的支撑：**`_stage_skill` 会把整个 slack-gif-creator/ 目录（包括 core/）复制到工作空间。因为工作目录是 `/tmp/ws_xxx/skills/slack-gif-creator/`，LLM 写的脚本在这个目录下执行时，`from core.gif_builder import GIFBuilder` 能直接成功——Python 的 import 路径天然包含当前目录。这个设计不需要改 `sys.path`，不需要 `pip install`，目录结构本身就是 `import path`。

## 06 模式四：参考文档渐进加载型

代表：`pptx`、`mcp-builder`

一句话概括：SKILL.md 是路由表，详细文档按需加载。

### 6.1 结构

```
pptx/
├── SKILL.md              ← 路由表（Quick Reference）
├── editing.md            ← 编辑工作流（6.9KB）
├── pptxgenjs.md          ← 从零创建（12.8KB）
└── scripts/
    ├── thumbnail.py
    └── office/
        ├── unpack.py
        └── soffice.py
```

### 6.2 SKILL.md 核心设计——路由表

```md
## Quick Reference

| Task | Guide |
|------|-------|
| Read/analyze content | `python -m markdown presentation.pptx` |
| Edit or create from template | Read [editing.md](editing.md) |
| Create from scratch | Read [pptxgenjs.md](pptxgenjs.md) |
```

SKILL.md 本身不包含详细操作步骤——它只告诉 LLM **"你要做的事情，应该去读哪份文档"**。

### 6.3 三层信息模型

这是整个 Skill 系统最精妙的设计之一：

```objectivec
第1层 description（~50词）
  "Presentation creation, editing, and analysis..."
  → 始终在可用技能列表中，LLM 判断是否需要加载
第2层 SKILL.md body（~200行）
  Quick Reference 路由表 + 设计指南 + QA 流程
  → 加载后注入 system message，LLM 知道大方向
第3层 editing.md / pptxgenjs.md（按需）
  详细操作步骤 + 代码示例
  → LLM 通过 skill_select_docs 按需加载
```

### 6.4 Token 节省效果

```objectivec
一次性全部加载：
  SKILL.md + editing.md + pptxgenjs.md ≈ 28.7KB ≈ ~7000 tokens

渐进加载（编辑任务）：
  description → ~50 tokens
  SKILL.md body → ~2000 tokens
  editing.md → ~1700 tokens
  总计 ≈ 3750 tokens（节省 ~46%）
```

**这就是"漏斗式处理"在 Token 管理上的应用：** 先用轻量信息粗筛（需要哪个 Skill？），再用中等信息定方向（该读哪份文档？），最后用详细信息执行（具体怎么操作？）。

这种模式体现了一个重要的设计原则：**"先粗筛再精选"**。面对海量数据/复杂问题，不要一步到位，应采用分层策略。Anthropic 的三层信息模型是这个思想在 Token 经济学上的落地：

| 注入层级 | 触发条件 | 典型 Token 消耗 |
| --- | --- | --- |
| L0: 概览 | 始终注入 | ~30 Token/技能 |
| L1: SKILL.md body | `skill_load`  后 | ~500-2000 Token |
| L2: 详细文档 | `skill_select_docs`  后 | ~1000-5000 Token |

如果你有 20 个技能，L0 的总成本是 ~600 Token——这就是"观望"的代价。一旦确定需要 pptx 技能，L1 注入 ~2000 Token 给出方向。最后按需加载 L2，整个过程的 Token 效率远高于"一口气把所有文档塞进去"。

从框架代码来看，`_skill_processor.py` 的 `_inject_overview` 方法是始终执行的（L0），而 `_build_docs_text` 只在 `_get_docs_selection` 返回非空列表时才执行（L2）。写入方 `skill_select_docs` 和读取方 `_build_docs_text` 之间通过 `state_delta["temp:skill:docs:pptx"]` 这个临时状态解耦——又是 CQRS。

## 07 模式五：编排型

代表：`skill-creator`

一句话概括：SKILL.md 不是工具说明，而是一个完整的多阶段工作流编排方案。

### 7.1 结构

```css
skill-creator/
├── SKILL.md              ← 32KB 的超详细编排指南
├── agents/               ← 子 Agent 指令
│   ├── grader.md         ← 评分 Agent
│   ├── comparator.md     ← A/B 对比 Agent
│   └── analyzer.md       ← 分析 Agent
├── scripts/              ← 自动化脚本
│   ├── aggregate_benchmark.py
│   ├── run_loop.py
│   ├── run_eval.py
│   └── package_skill.py
├── eval-viewer/          ← 评估结果查看器
└── references/           ← 参考文档
```

### 7.2 这不是工具，是流水线

SKILL.md 定义了一条完整的多阶段流水线：

```
Capture Intent → Interview → Write SKILL.md → Run Tests
→ Evaluate → Improve → Repeat → Package
```

执行时 LLM 会：

1. 与用户对话确定技能范围（纯对话，无工具）
2. 写出新 Skill 的 SKILL.md（用 write_to_file）
3. 并行 spawn 子 Agent 做 A/B 测试（with-skill vs baseline）
4. 执行评估脚本 聚合测试数据、生成可视化报告
5. 收集用户反馈 → 修改 → 回到第 3 步循环迭代
6. 打包输出 最终的.skill 文件

它同时用到了：

- 纯 Prompt 指令（工作流定义、决策逻辑）
- `skill_run` 执行（脚本工具链）
- `skill_select_docs`（按需加载子 Agent 指令）
- 外部交互（浏览器查看器、用户反馈循环）

这种模式的本质：SKILL.md 不是在教 LLM 怎么用一个工具，而是**在 编排 LLM 执行一个复杂的多步骤项目。**

**说实话，这个 Skill 的复杂度远超其余 15 个。** 一个 32KB 的 Markdown 文件，定义了一条完整的 CI/CD 式流水线——有测试、有评估、有 A/B 对比、有迭代循环。它不是给 LLM 用的工具说明，它是给 LLM 下的"项目管理指令"。

这让我意识到 SKILL.md 的上限远比我之前想象的高：它不只是"技能描述文件"，它可以是一个完整的 SOP（标准操作程序）。你把公司内部一个复杂的业务流程写成 SKILL.md 的格式——定义好每个阶段做什么、判断条件是什么、失败了怎么回退——LLM 就能按这个 SOP 自主执行。

`skill-creator` 同时也是五种模式的"全家桶"，它在不同阶段用了不同的执行方式：

- 阶段 1-2：纯 Prompt（对话 + 写文件）
- 阶段 3：`skill_select_docs` 加载子 Agent 指令（渐进加载）
- 阶段 4-5：`skill_run` 执行评估脚本（脚本执行）
- 全程：32KB 编排指南控制流程（编排）

## 08 五种模式的统一视角

```
                    Skill 执行模式光谱
  ◄── 轻量 ────────────────────────────────── 重量 ──►
  纯 Prompt      参考文档       库调用       脚本执行      编排
  注入型         渐进加载型      型           型            型
  ──────────    ──────────    ──────────   ──────────   ──────────
  frontend-     pptx          slack-gif-   pdf          skill-
  design        mcp-builder   creator      xlsx         creator
  ──────────    ──────────    ──────────   ──────────   ──────────
  仅注入 body    注入 +         注入 +       注入 +        注入 +
  到 system      按需加载       LLM 写代码   预制脚本      多步骤
  message        docs          import 库    skill_run    工作流
```

### 8.1 所有模式共用的底座

| 框架机制 | 模式一 | 模式二 | 模式三 | 模式四 | 模式五 |
| --- | --- | --- | --- | --- | --- |
| `skill_load`  → body 注入 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `skill_select_docs` | ❌ | ✅ | ❌ | ✅ | ✅ |
| `skill_run`  (预制脚本) | ❌ | ✅ | ❌ | ✅ | ✅ |
| `skill_run`  (LLM 写的脚本) | ❌ | ❌ | ✅ | ❌ | ❌ |
| `Tools:`  声明 (function calling) | ❌ | ❌ | ❌ | ❌ | ❌ |
| 子 Agent 编排 | ❌ | ❌ | ❌ | ❌ | ✅ |

## 09 最终洞察：为什么官方 Skill 都不用 function calling？

回到最初的问题——"怎么让 Skill 的能力被模型真正调用"。

Anthropic 自己的 16 个 Skill 给出了一个出人意料的答案：

**不需要注册 function calling。把 SKILL.md 写好就行了。**

这和直觉完全相反。作为后端工程师，第一反应是：要让 LLM 执行一个能力，那就注册一个 function calling，定义好输入输出 Schema，类型安全、结构化、可验证。

但 Anthropic 显然想通了一件事：**function calling 是给确定性操作设计的，而大多数 Skill 面对的是不确定性任务。**

原因有三：

1. **LLM 本身就是最好的代码执行器。** 对于 PDF、PPT、Excel 这类任务，LLM 看了代码示例后自己就能写出正确的代码。注册 function calling 反而限制了灵活性——你只能调预定义的函数，不能灵活组合。

想想看：pdf 技能有 8 个脚本，如果注册为 8 个 function calling，LLM 只能在这 8 个中选。但如果 SKILL.md 教会了 LLM 怎么用 `pypdf` 和 `reportlab`，它可以写出第 9 种、第 10 种操作——这些操作可能是 Skill 作者压根没想到的。

2. **`skill_run` 是万能兜底。** 任何语言、任何命令，只要能在 shell 里跑，skill_run 就能执行。不需要为每个操作都定义工具 Schema。

从 Token 经济学的角度看，一个 `skill_run` 的 Schema 约 20 Token，而 8 个独立 function calling 的 Schema 约 1600-4000 Token。在 10 轮对话中，这差了** 1.6 万到 4 万 Token。**

3. **`Tools:` 的价值场景很窄。** 只有当你需要调用 **不能通过写代码/shell 命令完成的操作** 时（如调用内部 API、访问数据库、触发外部服务），才需要注册 function calling。而 Anthropic 的官方 Skill 都可以通过代码+脚本完成。

框架确实实现了完整的 `Tools:` 声明 → `DynamicSkillToolSet` → function calling 链路（源码中可以拆出完整的 7 步）。但 Anthropic 自己的 16 个 Skill 一个都没用——**框架预留了能力，但实践中发现不需要。**这本身就是一个重要的工程信号。

### 9.1 真正的公式  

```
Skill 的执行力 = SKILL.md body 质量 × (Agent 基础工具 + skill_run 沙箱能力)
```

**SKILL.md 的 body 文本是"灵魂"**——它决定了 LLM 能否理解任务、选对方法、写对代码。

**`skill_run` 是"手脚"**——它提供了在隔离沙箱中执行任意命令的能力。

**`Tools:` 声明是"可选配件"**——框架支持，但大多数场景不需要。

## 10 给 Skill 开发者的实操建议

根据这五种模式，选择适合你场景的方式：

| 你的场景 | 推荐模式 | 你需要做的 |
| --- | --- | --- |
| 教 LLM 遵循某种规范/风格 | 纯 Prompt 注入 | 写好 SKILL.md，全靠文本质量 |
| 需要 LLM 操作特定文件格式 | 脚本执行型 | 写预制脚本放 `scripts/` ，SKILL.md 写使用说明 |
| 需要 LLM 灵活组合 API | 库调用型 | 写 Python 库放 `core/` ，SKILL.md 当 API 文档 |
| 知识量大，不同任务需要不同文档 | 渐进加载型 | SKILL.md 做路由表，详细文档独立存放 |
| 需要 LLM 执行复杂多步骤工作流 | 编排型 | SKILL.md 定义流水线 + 脚本工具链 + 子 Agent |

万变不离其宗：写好 SKILL.md 是一切的基础。

## 11 附：框架完整架构速览

如果你对框架源码感兴趣，这是我拆解后画的架构全景图：

```
用户对话
   │
   ▼
┌──────────────────────────────────────────────────┐
│  LlmAgent                                         │
│  ┌──────────────────┐  ┌───────────────────────┐  │
│  │ SkillToolSet      │  │ DynamicSkillToolSet    │  │
│  │ (管理类工具)       │  │ (按需加载业务工具)      │  │
│  │ - skill_load      │  │ - 根据 state_delta     │  │
│  │ - skill_list      │  │   动态返回工具实例      │  │
│  │ - skill_run       │  │ - 四级查找降级         │  │
│  │ - skill_select_*  │  │   缓存→字典→集合→注册表  │  │
│  └────────┬──────────┘  └────────┬──────────────┘  │
│           │                      │                  │
│           ▼                      ▼                  │
│  ┌──────────────────────────────────────────────┐  │
│  │  state_delta（临时状态，不持久化）              │  │
│  │  temp:skill:loaded:xxx = "1"                  │  │
│  │  temp:skill:tools:xxx = '["tool1"]' 或 '*'    │  │
│  │  temp:skill:docs:xxx = '["API.md"]'           │  │
│  └─────────────────────┬────────────────────────┘  │
│                        │                            │
│                        ▼                            │
│  ┌──────────────────────────────────────────────┐  │
│  │  SkillsRequestProcessor                       │  │
│  │  读取 state_delta → 构建注入内容              │  │
│  │  L0: _inject_overview（始终执行）              │  │
│  │  L1: skill.body（已加载的技能）                │  │
│  │  L2: _build_docs_text（已选文档）              │  │
│  │  → _merge_into_system（追加到 system prompt）  │  │
│  └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────┐     ┌───────────────────────┐
│  FsSkillRepository   │     │  WorkspaceRuntime      │
│  扫描 → 解析 → 缓存  │     │  本地模式 / 容器模式    │
│  from_markdown()     │     │  隔离执行 + 文件收集    │
│  _parse_tools()      │     │  增量暂存 + 只读保护    │
└──────────────────────┘     └───────────────────────┘
```

完整的源码拆解（9 个文件，逐行分析）已整理成一份 2000+ 行的技术报告，包括从 SKILL.md 声明到 LLM function_call 的完整 7 步链路、DynamicSkillToolSet 的 Token 优化机制、_skill_processor.py 的 7 个内部方法等。如果有兴趣可以留言，后续单独成文。

## 12 写在最后

拆解完这 16 个 Skill 后，核心结论是：

**Anthropic 对 Skill 系统的设计哲学，本质上是对 LLM 能力边界的深刻理解。**

他们没有把 Skill 做成"给 LLM 注册更多 API"的系统，而是做成了"教 LLM 更多知识"的系统。因为他们比谁都清楚——LLM 最强的能力不是调 API，而是 **理解自然语言指令后自主解决问题。**

SKILL.md 本质上就是一种极其高效的知识传递方式：用最少的 Token，把最关键的领域知识、操作规范和工具用法，注入到 LLM 的上下文中。

从工程角度看，这套系统的架构也值得学习：

- **CQRS 解耦：** 写入方（_tools.py）和读取方（_skill_processor.py）通过 state_delta 解耦
- **漏斗式加载：** 三层信息模型，逐步注入，Token 效率最大化
- **声明式绑定：** 工具名字符串匹配而非 import 引用，松耦合
- **统一 IR：** FunctionDeclaration 作为中间表示，屏蔽不同 LLM API 的差异

如果你正在搭建自己的 Agent 系统，或者想给 AI 编程助手添加自定义技能，这五种模式应该能提供一些参考。

**最简单的起步方式：** 写一个只有 SKILL.md 的纯 Prompt 注入型技能，先跑起来，再根据需要逐步加脚本、加库、加文档。

毕竟，Anthropic 自己也是从一个 Markdown 文件开始的。