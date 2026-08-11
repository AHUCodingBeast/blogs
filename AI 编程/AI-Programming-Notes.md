# AI 编程方法论：如何让 AI 写出高质量代码

> 📌 **本笔记聚焦 AI 编程的核心方法论**，讲解如何通过规范（Rules/Spec）、任务拆解、上下文管理和审查闭环，让 AI 产出生产级代码。工具会迭代，方法论永不过时。

工具只是载体，**方法论才是核心竞争力**。同样的工具，有人用它写出生产级代码，有人只能生成 demo 级别的玩具代码，差距在于是否掌握了以下方法论。

### 1. 规范先行：Rules / Spec 是 AI 编程的"宪法"

AI 不知道你的项目约定、技术栈偏好、代码风格。**你不告诉它，它就会按训练数据的平均水平生成代码**——这往往不符合你的项目要求。

**规范的三个层次**：

| 层次 | 内容 | 示例 |
| :--- | :--- | :--- |
| **全局规范** | 语言/框架通用约定 | Java 用 P3C、Python 用 Black、React 用 ESLint |
| **项目规范** | 本项目特有约定 | Service 不依赖 DAO、异常统一处理、日志格式 |
| **任务规范** | 单次任务的约束 | "只修改 UserService，不改 Controller"、"保持向后兼容" |

**最佳实践**：
-   将规范文件（`CLAUDE.md` / `.cursorrules` / `AGENTS.md`）**纳入 Git 版本管理**，团队共享；
-   规范要**具体、可执行**，避免模糊表述（❌ "写好代码" → ✅ "public 方法必须有 Javadoc，参数用 @param 标注"）；
-   定期 Review 和更新规范，随项目演进持续优化。

### Spec-Driven Development：用规范文档驱动 AI 生成代码

Spec-Driven Development（规范驱动开发）是 AI 编程时代最重要的方法论升级。它的核心理念是：**先写规范（Spec），再让 AI 按规范生成代码**，而不是直接让 AI "帮我写一个 xxx"。

#### 为什么需要 Spec-Driven？

传统 AI 编程的典型问题是：
```
你："帮我写一个用户注册接口"
AI：生成了一个能跑但不符合项目规范的代码
你：花 30 分钟改成符合项目的样子
```

Spec-Driven 的做法是：
```
你：先写一份 Spec（接口定义 + 业务规则 + 异常处理 + 测试要求）
AI：严格按 Spec 生成代码，一次到位
你：Review 确认，5 分钟搞定
```

**本质区别**：从"AI 猜你想要什么"变成"你明确告诉 AI 要什么"。Spec 是你和 AI 之间的**契约**。

#### Spec 的三种形态

| 形态 | 适用场景 | 示例 |
| :--- | :--- | :--- |
| **接口 Spec** | API 开发、前后端联调 | OpenAPI/Swagger YAML、Protobuf 定义 |
| **功能 Spec** | 业务逻辑实现 | Markdown 文档描述输入/输出/边界条件/异常处理 |
| **架构 Spec** | 模块设计、重构 | 类图、时序图、职责划分、依赖关系 |

#### 功能 Spec 模板

```markdown
# 用户注册接口 Spec

## 接口定义
- POST /api/v1/users/register
- Request Body: { username: string, email: string, password: string }
- Response: { userId: long, username: string, createdAt: datetime }

## 业务规则
1. username 长度 4-20，仅允许字母数字下划线
2. email 必须通过格式校验且未注册
3. password 长度 8-32，必须包含大小写字母和数字
4. 注册成功后发送验证邮件（异步，不阻塞响应）

## 异常处理
- username 已存在 → 409 Conflict
- email 已注册 → 409 Conflict
- 参数校验失败 → 400 Bad Request + 具体字段错误信息
- 邮件发送失败 → 记录日志，不影响注册成功响应

## 测试要求
- 正常注册流程
- username/email 重复
- 各字段边界值校验
- 邮件发送失败的降级处理
```

#### Spec-Driven 的工作流

```
1. 写 Spec（你）          ← 定义"做什么"和"怎么做"
       ↓
2. AI 读 Spec 生成代码    ← 严格按契约执行
       ↓
3. AI 自测（跑测试/lint） ← 验证是否符合 Spec
       ↓
4. 你 Review              ← 确认 Spec 被正确实现
       ↓
5. Spec 纳入 Git          ← 成为活文档，后续维护有据可依
```

#### Spec-Driven vs 传统 Prompt 的区别

| 维度 | 传统 Prompt | Spec-Driven |
| :--- | :--- | :--- |
| 输入 | 自然语言描述 | 结构化规范文档 |
| AI 理解 | 猜测意图，容易偏差 | 精确执行，减少歧义 |
| 输出质量 | 不稳定，每次不同 | 稳定可预期 |
| 可复用性 | 一次性 | Spec 可复用、可版本化 |
| 团队协作 | 口头传达 | Spec 即文档，新人也能用 |
| 审查效率 | 逐行看代码 | 对照 Spec 检查，聚焦偏差 |

> 💡 **面试话术**："Spec-Driven Development 是我在 AI 编程中最核心的方法论。不是直接让 AI '帮我写一个 xxx'，而是先写一份结构化的 Spec 文档——定义接口、业务规则、异常处理、测试要求，然后让 AI 严格按 Spec 生成代码。这样做的好处是：AI 输出稳定可预期、Review 时只需对照 Spec 检查偏差、Spec 本身成为活文档可复用可版本化。本质上是从'AI 猜你想要什么'变成'你明确告诉 AI 要什么'。"

### 2. 任务拆解：大任务拆成小步骤

AI 在处理**边界清晰、范围可控**的任务时表现最好。一个"重构整个订单模块"的请求，远不如拆成 5 个小任务效果好。

**拆解原则**：
-   **单一职责**：每个任务只做一件事；
-   **可验证**：每个任务完成后能通过测试/lint/编译验证；
-   **有上下文**：提供相关文件路径、接口定义、业务背景。

**反面案例 vs 正面案例**：

```
❌ "帮我重构用户模块"
✅ "第一步：把 UserController 中的参数校验逻辑抽取到 UserValidator 类中，
    参考 OrderValidator 的实现方式（见 src/main/java/.../OrderValidator.java）"
```

### 3. 上下文管理：给 AI 足够的信息

AI 的输出质量 = f(模型能力, 上下文质量)。模型能力你控制不了，但上下文质量完全由你决定。

**上下文提供策略**：

| 策略 | 说明 | 工具支持 |
| :--- | :--- | :--- |
| **@file 引用** | 显式指定相关文件 | Cursor @file、Claude Code 自动读取 |
| **@codebase 搜索** | 让 AI 自行搜索相关代码 | Cursor @codebase、Claude Code grep |
| **粘贴关键代码** | 接口定义、数据结构、错误信息 | 所有工具通用 |
| **描述业务背景** | "这个接口是给前端列表页用的，需要分页" | 所有工具通用 |
| **提供示例** | "参考 XxxService 的写法" | 所有工具通用 |

> ⚠️ **常见误区**：以为 AI "什么都知道"。AI 不知道你项目的私有 API、内部约定、历史决策。**你提供的上下文越精确，AI 的输出越可靠**。

### 4. 审查闭环：永远不要盲信 AI 生成的代码

AI 生成的代码**必须经过人类审查**，这是 AI 编程的铁律。

**审查清单**：
-   [ ] **正确性**：逻辑是否符合需求？边界条件是否处理？
-   [ ] **安全性**：是否有 SQL 注入、XSS、敏感信息泄露风险？
-   [ ] **性能**：是否有 N+1 查询、不必要的循环、内存泄漏？
-   [ ] **一致性**：是否符合项目规范和现有代码风格？
-   [ ] **可测试性**：是否方便写单元测试？依赖是否可 Mock？
-   [ ] **向后兼容**：是否破坏了现有接口契约？

**最佳实践**：
-   让 AI **自己先跑一遍测试/lint**，再交给你 Review；
-   对关键逻辑要求 AI **解释实现思路**，确认理解一致；
-   发现 AI 反复犯同类错误时，**更新规范文件**，从根源解决。

### 5. 渐进式信任：从小任务建立信心

不要一开始就让 AI 做核心业务逻辑的重构。**从小任务开始，逐步扩大 AI 的职责范围**：

```
Level 1: 代码补全、格式化、注释生成
    ↓ 建立信任
Level 2: 单元测试生成、简单 CRUD、工具方法
    ↓ 验证质量
Level 3: 业务逻辑修改、接口重构、Bug 修复
    ↓ 确认可靠
Level 4: 跨模块重构、架构调整、新功能开发
```

---

## Claude Code 与 Codex CLI 核心功能详解

方法论是通用的，但面试中经常考察你对**具体工具深度功能的理解**。以下是 Claude Code 和 Codex CLI 的高频考点。

### 1. Subagent（子代理）

Subagent 是 Claude Code 的**任务委派机制**。当主 Agent 遇到一个可以独立完成的子任务时，它会 spawn 一个子代理去执行，自己继续处理其他事情或等待结果。

**典型场景**：
-   主 Agent 在做跨模块重构时，spawn 子代理去"读取并分析 OrderService 的所有调用方"；
-   主 Agent 在写新功能时，spawn 子代理去"搜索项目中现有的参数校验写法作为参考"；
-   并行执行多个独立的文件修改任务。

**为什么重要**：Subagent 让 AI 具备了**任务分解和并行执行**的能力，而不是线性地一步步做。这是 Agent 和普通 Chat 的本质区别之一。

**面试话术**："Subagent 是 Claude Code 的任务委派机制，主 Agent 可以把独立的子任务交给子代理并行执行，比如一边改代码一边让子代理去搜索相关引用。这让复杂任务的完成效率远高于线性对话。"

### 2. 斜杠命令（Slash Commands）

斜杠命令是 Claude Code 的**快捷操作入口**，在对话中输入 `/` 触发。日常高频使用的命令：

| 命令 | 作用 | 使用场景 |
| :--- | :--- | :--- |
| `/init` | 自动生成 `CLAUDE.md` 项目规范文件 | 新项目接入时一键初始化 |
| `/context` | 查看当前会话的上下文使用情况 | 感觉 AI "记不住"前面内容时检查 |
| `/compact` | 压缩长对话，释放 token 空间 | 上下文快满时使用，支持指定保留重点 |
| `/diff` | 交互式查看代码改动 | 提交前逐个文件确认改了什么 |
| `/rewind` | 回滚到之前的对话点，文件改动一起还原 | AI 改错了，一键撤销 |
| `/btw` | 顺便问个问题，不打断当前上下文流 | 讨论 A 时突然想确认 B 的知识点 |
| `/export` | 导出对话记录到文件或剪贴板 | 保存架构决策过程、复盘用 |
| `/memory` | 查看和管理记忆系统 | 检查/编辑 AI 记住的项目信息 |
| `/review` | 对当前 git diff 进行代码审查 | 提交前让 AI 帮你 Review |
| `/simplify` | 检查代码质量 | 提交代码时使用 |
| `/batch` | 批量处理改动 | 如给所有公开函数加注释 |
| `/doctor` | 诊断并验证安装和设置 | 环境出问题时排查 |

> 💡 **关键细节**：
> -   `/compact` 支持指定保留重点：`/compact focus on the database migration plan`，只保留最终方案，清掉早期的拉扯；
> -   `/rewind` 是对话和代码的**联合撤销键**，比手动还原文件高效得多；
> -   `/btw` 不会污染主线上下文，适合临时插问；
> -   `/init` 不是生成模板，而是**先扫描项目结构、读取现有代码、分析技术栈**，再生成贴合项目的 CLAUDE.md。

#### 自定义斜杠命令

除了内置命令，你还可以在 `.claude/commands/` 目录下创建自定义命令（Markdown 文件），定义项目专属的快捷操作。例如创建一个 `/deploy-check` 命令，自动执行部署前检查清单。

### 3. Skills（技能）

Skills 是 Claude Code 的**可复用能力单元**，可以理解为"预定义的 Prompt 模板 + 工具链"。

**内置 Skills 示例**：
-   **代码审查 Skill**：自动读取 diff → 按安全检查清单逐项审查 → 输出结构化报告；
-   **测试生成 Skill**：分析目标方法 → 生成边界用例 + 异常用例 + 正常用例；
-   **文档生成 Skill**：读取接口定义 → 生成 API 文档 + 使用示例。

**自定义 Skill**：在 `.claude/skills/<skill-name>/` 目录下创建 `SKILL.md` 文件。文件需要包含 metadata 头部（name + description），AI 加载时只读取 metadata 部分，不会把所有提示词都塞进上下文：

```markdown
---
"name": "onboard",
"description": "Help new team members get up to speed with the project"
---
你是一个项目引导员。请按以下步骤帮助新成员熟悉项目：
1. 读取 CLAUDE.md 和项目 README，介绍项目定位和技术栈
2. 运行 ls -la 和关键目录结构，解释项目组织方式
3. 展示最近的 3 个 commit 和对应的 PR，说明当前开发重点
4. 指出代码中容易让人困惑的"历史遗留"部分
```

**Skill 的作用域**：
-   **项目级**：放在 `<project>/.claude/skills/` 下，仅当前项目可用；
-   **用户级**：放在 `~/.claude/skills/` 下，所有项目可用。

**使用方式**：① 直接输入 `/onboard` 触发；② 对话中 AI 自动识别并调用合适的 Skill。也可以通过自然语言让 AI 把之前的交互内容沉淀为 Skill。

**面试话术**："Skills 是可复用的能力单元，相当于预定义的 Prompt 模板加工具链。SKILL.md 文件有固定的 metadata 格式，AI 加载时只读 metadata 不占上下文。除了内置 Skill，我会自定义项目专属的 Skill，比如 onboard 引导新成员、标准化的 Service 重构流程等，确保同类任务每次都按相同标准执行。"

### 4. 自定义开发规划（Beyond CLAUDE.md）

除了项目级的 `CLAUDE.md`，你还可以定义**个人级的开发规划和偏好**：

| 配置文件 | 作用域 | 是否入 Git | 内容示例 |
| :--- | :--- | :--- | :--- |
| `~/.claude/CLAUDE.md` | 用户级，所有项目 | ❌ | 个人编码偏好、沟通风格 |
| `<项目>/CLAUDE.md` | 项目级，全目录生效 | ✅ | 技术栈、构建命令、代码规范 |
| `<项目>/子目录/CLAUDE.md` | 目录级，仅该目录生效 | ✅ | 模块专属约定 |
| `CLAUDE.local.md` | 个人偏好，不入版本控制 | ❌（.gitignore） | 个人习惯覆盖 |
| `.claude/settings.json` | 项目级配置 | ✅ | 允许的 Shell 命令、Hooks、MCP |
| `.claude/settings.local.json` | 个人设置，不入版本控制 | ❌（.gitignore） | 个人权限/模型偏好 |
| `.claude/skills/*/SKILL.md` | 项目级 Skill | ✅ | 自定义工作流模板 |
| `.claude/agents/*.md` | 项目级 Agent | ✅ | 自定义子代理定义 |
| `.claude/commands/*.md` | 项目级自定义命令 | ✅ | 自定义斜杠命令 |

> ⚠️ **优先级规则**：**子目录 CLAUDE.md > 项目根 CLAUDE.md > 用户级 ~/.claude/CLAUDE.md**。越具体的优先级越高。`CLAUDE.local.md` 和 `settings.local.json` 用于个人覆盖，**必须加入 .gitignore**。

**用户级 CLAUDE.md 示例**：
```markdown
# 个人偏好

## 沟通风格
- 回答简洁直接，不要过度解释
- 修改代码前先说明思路，等我确认再动手
- 遇到不确定的地方主动问我，不要猜测

## 编码习惯
- 变量命名用 camelCase，常量用 UPPER_SNAKE_CASE
- 优先使用 Stream API 而非 for 循环
- 日志用 SLF4J，格式：log.info("methodName - description, param={}", value)

## 工具链
- 构建工具：Maven
- 测试框架：JUnit 5 + Mockito
- 代码格式化：Spotless
```

> 💡 **关键区别**：`CLAUDE.md` 是"项目宪法"，团队共享、纳入 Git；用户级配置是"个人习惯"，不入 Git、跟随账号。两者叠加生效，项目规范优先级高于个人偏好。

### 5. 记忆管理（Memory System）

Claude Code 的记忆系统是它区别于普通 Chat 工具的**核心差异化能力**。AI 不再是"金鱼记忆"，而是能跨会话记住你的项目和偏好。

**记忆的层次**：

| 记忆类型 | 存储位置 | 生命周期 | 内容示例 |
| :--- | :--- | :--- | :--- |
| **项目记忆** | `~/.claude/projects/<项目路径>/memory/*.md` | 项目级，跨会话持久化 | "OrderService 超时设 30s"、"支付模块用策略模式" |
| **用户记忆** | `~/.claude/memory/*.md` | 用户级，跨项目 | "用户偏好简洁回复"、"用户是资深 Java 后端" |
| **会话记忆** | 内存中 | 单次会话 | 当前对话上下文、已读取的文件、已做的修改 |

> ⚠️ **注意**：项目记忆的存储路径是 `~/.claude/projects/<项目路径>/memory/`，**不是**项目目录下的 `.claude/memory/`。这意味着项目记忆跟随用户账号而非项目仓库，换电脑后需要重新积累或手动迁移。

**记忆的工作机制**：
1.  **自动学习**：Claude Code 在对话中发现值得记住的信息时，会主动询问"是否要记住这个？"或自动写入记忆文件；
2.  **主动提醒**：你也可以用自然语言告诉 AI "请记住 xxx"，它会持久化到记忆文件中；
3.  **主动回忆**：新会话开始时，Claude Code 会自动加载相关的记忆文件作为上下文；
4.  **手动管理**：通过 `/memory` 命令查看、编辑、删除记忆条目。

> 💡 **实践建议**：关闭终端再打开是新会话，AI 可能忘记之前聊过的内容。**重要的对话信息要主动让 AI 记住**，而不是依赖它自动学习。

**CLAUDE.md vs Memory 的区别**：
-   **CLAUDE.md** 是你**主动写的"规矩"**，团队共享、纳入 Git、结构化；
-   **Memory** 是 Claude **自己记的"笔记"**，个人专属、碎片化、自动积累。
-   两者每次启动时都会自动加载，互补而非替代。

**面试话术**："Claude Code 的记忆系统分三层：项目记忆存在 `~/.claude/projects/<项目路径>/memory/` 下跨会话持久化；用户记忆存在 `~/.claude/memory/` 下跨项目生效；会话记忆仅在单次对话中有效。CLAUDE.md 是我主动写的团队规范，Memory 是 AI 自己记的个人笔记，两者互补。新会话启动时自动加载，但重要信息我会主动让 AI 记住，避免丢失。"

### 6. Codex CLI 对应功能对照

| 功能 | Claude Code | Codex CLI |
| :--- | :--- | :--- |
| 项目规范 | `CLAUDE.md` | `AGENTS.md` |
| 子代理 | Subagent（自动 spawn） | 类似机制，沙箱隔离执行 |
| 斜杠命令 | `/init` `/compact` `/review` 等 | `/init` `/compact` 等（命令集略有差异） |
| 记忆系统 | 三层记忆（项目/用户/会话） | 基于 `AGENTS.md` + 会话上下文 |
| 安全机制 | 权限审批 + 沙箱 | 沙箱隔离 + 网络隔离 + 审批机制 |
| 多模型支持 | Claude Sonnet/Opus/Haiku | o3/o4-mini/GPT-4o |
| 自定义 Skill | `.claude/skills/*.md` | 通过 `AGENTS.md` 定义工作流 |

> ⚠️ **注意**：Codex CLI 的安全模型更严格——默认在沙箱中执行，网络访问受限，敏感操作需要显式审批。这在企业环境中是优势，但也意味着某些自动化场景需要额外配置。

### 7. `!` 和 `@` 符号：日常最高频的交互方式

这两个符号是 Claude Code 日常使用中**频率最高的交互方式**：

-   **`!` 执行命令**：在对话中输入 `!命令`，直接在当前项目目录执行 Shell 命令，**输出结果会进入 AI 的上下文**，AI 可以基于结果继续分析。这是和普通终端执行命令的核心区别。
    ```bash
    !ls -la                    # 查看目录结构，AI 能看到结果
    !curl http://localhost:3000/health  # 检查服务健康状态
    !git log --oneline -5      # 查看最近提交
    ```
    > 💡 长时间运行的命令可按 `Ctrl+B` 放后台，`Ctrl+X Ctrl+K` 终止所有后台代理。

-   **`@` 引用文件**：直接引用文件或目录作为上下文，AI 会读取并理解其内容。
    ```bash
    @src/main/java/UserService.java 帮我重构这个方法
    @docs/api-spec.yaml 根据这个接口定义生成 Controller
    @markdown-note/ 统计下这个文件夹下有多少个 md 文件
    ```

### 8. Hooks：事件驱动的自动化脚本

Hook 绑定在特定事件上，事件发生时**自动执行脚本**，无需手动触发。配置在 `.claude/settings.json` 中：

```json
{
  "hooks": {
    "PreCompact": [
      { "type": "command", "command": "git diff --stat" }
    ],
    "Notification": [
      { "type": "command", "command": "echo '对话已结束' | growlnotify" }
    ]
  }
}
```

**常用 Hook 类型**：

| Hook 类型 | 触发时机 | 典型用途 |
| :--- | :--- | :--- |
| `PreCompact` | 压缩上下文之前 | 保存关键信息、导出当前改动 |
| `PostToolUse` | 使用某个工具之后 | 自动格式化、日志记录 |
| `Notification` | 对话结束时 | 发送桌面通知提醒 |
| `PreStop` | Claude Code 退出之前 | 清理临时文件、保存状态 |

### 9. MCP（Model Context Protocol）：连接外部服务

MCP 让 Claude Code 能**直接调用外部服务**——GitHub Issues、数据库、地图 API、内部系统等，无需手动复制粘贴数据。

**配置方式**：在 `.claude/settings.json` 中添加 MCP Server：
```json
{
  "mcpServers": {
    "amap-maps": {
      "url": "https://mcp.amap.com/mcp?key=${AMAP_KEY}"
    }
  }
}
```

**管理命令**：
```bash
claude mcp list              # 查看已连接的 MCP 服务
claude mcp get <name>        # 查看某个服务详情
claude mcp remove <name>     # 移除服务
/mcp                         # 在对话中查看活动连接
```

> ⚠️ **安全注意事项**：
> -   凭证用环境变量存（`${AMAP_KEY}`），**禁止硬编码**；
> -   数据库查询用**只读权限**；
> -   遵循最小权限原则，第三方 MCP 服务需审慎评估安全风险；
> -   推荐使用 HTTP 和 Stdio 两种传输类型，不推荐 SSE；
> -   MCP 会占用上下文空间，对于简单场景优先考虑 CLI 工具替代。

### 10. Plugin：能力的集合体

Plugin 是 **Skills + MCP + Agents + Hooks 的集合体**，可以打包分发和安装：

```bash
/plugin install github@claude-plugins-official   # 安装官方插件
/plugin                                           # 查找可用插件
```

社区和官方提供的 Plugin 可以快速扩展 Claude Code 的能力，也可以自行开发 Plugin 供团队使用。

---

## 面试高频问题速查

### Q1: 你知道什么是 Subagent 吗？
**答**：Subagent 是 Claude Code 的任务委派机制。当主 Agent 遇到可以独立完成的子任务时，会 spawn 一个子代理去并行执行，自己继续处理其他事情或等待结果。典型场景：跨模块重构时让子代理去分析调用方、写新功能时让子代理搜索现有写法作为参考。这是 Agent 和普通 Chat 的本质区别——具备任务分解和并行执行能力。

### Q2: Claude Code 的斜杠命令有哪些？常用的有哪几个？
**答**：日常高频命令：`/init`（扫描项目后生成定制化 CLAUDE.md）、`/context`（查看上下文使用情况）、`/compact`（压缩上下文，支持 `focus on` 指定保留重点）、`/diff`（交互式查看代码改动）、`/rewind`（对话+文件联合撤销）、`/btw`（插问不打断主线）、`/export`（导出对话记录）、`/memory`（管理记忆）、`/review`（代码审查）、`/simplify`（代码质量检查）、`/batch`（批量处理）。此外还支持在 `.claude/commands/` 下自定义斜杠命令。

### Q3: Claude Code 有哪些常用的 Skill？能自定义吗？
**答**：内置 Skill 包括代码审查、测试生成、文档生成等标准化工作流。可以在 `.claude/skills/` 目录下创建自定义 Skill 文件，定义项目专属的标准化流程，比如"Service 重构六步法"、"接口上线检查清单"等。Skill 本质是"预定义的 Prompt 模板 + 工具链"，确保同类任务每次都按相同标准执行。

### Q4: 除了项目级 CLAUDE.md，还能定义个人专属的开发规划吗？
**答**：可以，Claude Code 支持多层配置体系：① `~/.claude/CLAUDE.md`：用户级全局偏好，跨项目生效；② `<项目>/CLAUDE.md`：项目级规范，团队共享入 Git；③ `<项目>/子目录/CLAUDE.md`：目录级约定，仅该目录生效；④ `CLAUDE.local.md` + `.claude/settings.local.json`：个人覆盖，**必须加入 .gitignore**。优先级：子目录 > 项目根 > 用户级。此外还可以在 `.claude/commands/` 自定义斜杠命令、`.claude/agents/` 自定义子代理、`.claude/skills/` 自定义 Skill。
### Q5: Claude Code 的记忆管理大致是怎样的？
**答**：三层记忆体系：① **项目记忆**（`~/.claude/projects/<项目路径>/memory/*.md`）：跨会话持久化，注意存储路径在用户目录下而非项目目录中；② **用户记忆**（`~/.claude/memory/*.md`）：跨项目生效，记录个人偏好；③ **会话记忆**：仅单次对话有效。CLAUDE.md 是你主动写的"规矩"，Memory 是 AI 自己记的"笔记"，两者互补。新会话启动时自动加载，但重要信息建议主动让 AI 记住（自然语言说"请记住 xxx"），避免关闭终端后丢失。通过 `/memory` 命令可手动管理。

### Q6: 如何保证 AI 生成的代码质量？
**答**：三层保障：① **规范先行**：通过 CLAUDE.md 定义代码风格、架构约定、测试要求，AI 生成代码时自动遵循；② **任务拆解**：大任务拆成边界清晰的小步骤，每步可独立验证；③ **审查闭环**：AI 生成后用 `/test` `/lint` 自测，再由人工 Review 正确性、安全性、性能、一致性。发现反复犯的错就更新 CLAUDE.md，从根源解决。

### Q7: 什么是 MCP？使用时需要注意什么？
**答**：MCP（Model Context Protocol）让 Claude Code 能直接调用外部服务（GitHub、数据库、地图 API 等），无需手动复制数据。配置在 `.claude/settings.json` 的 `mcpServers` 中。安全注意事项：① 凭证用环境变量存，禁止硬编码；② 数据库用只读权限；③ 最小权限原则，第三方服务需审慎评估；④ 推荐 HTTP/Stdio 传输，不推荐 SSE；⑤ MCP 会占用上下文，简单场景优先用 CLI 工具替代。

### Q8: 什么是 Hooks？有哪些常用类型？
**答**：Hook 是绑定在特定事件上的自动化脚本，事件触发时自动执行，无需手动调用。配置在 `.claude/settings.json` 的 `hooks` 字段中。常用类型：`PreCompact`（压缩前保存关键信息）、`PostToolUse`（工具使用后自动格式化）、`Notification`（对话结束发通知）、`PreStop`（退出前清理临时文件）。适合把重复性的检查/清理/通知操作自动化。

### Q9: AI 编程和传统编程的核心区别是什么？
**答**：角色转变。传统编程你是"执行者"，AI 编程你是"架构师 + Reviewer"。核心价值从"写代码"转向"定义规范、拆解任务、审查结果"。AI 是高效执行者，但它不知道你的项目约定和业务背景，需要你通过 CLAUDE.md、Skills、Hooks 和上下文来引导。掌握这个方法论，无论工具怎么迭代都能快速适应。

---

## 小结

AI 编程的核心不是"会用哪个工具"，而是**掌握让 AI 高效产出高质量代码的方法论**：

-   **工具选型**：IDE 集成型（Cursor/Copilot）适合日常开发，终端 Agent 型（Claude Code/Codex CLI）适合复杂任务；
-   **规范先行**：Rules/Spec 是 AI 编程的"宪法"，决定了 AI 输出的基线质量；
-   **任务拆解**：大任务拆成小步骤，每步边界清晰、可验证；
-   **上下文管理**：给 AI 足够的信息，它才能给出精准的回答；
-   **审查闭环**：永远不要盲信 AI，人类 Review 是最后一道防线；
-   **渐进信任**：从小任务建立信心，逐步扩大 AI 的职责范围。

> 📌 **记忆口诀**：**规范定基线、拆解控粒度、上下文喂饱、审查守底线、工具随势变、方法永不过时**。
