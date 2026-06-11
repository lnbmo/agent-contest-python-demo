# CLAUDE.md — agent-contest-python-demo

## 项目概述

Skill 蒸馏攻防 Agent 大赛（Python Demo）——一个 AI Agent 竞赛框架。参赛者构建 Agent 来回答赛方题目，通过 MCP-style tools、skills 和 sub-agents 完成任务。项目使用纯 Python 标准库，无需第三方依赖（可选 fastmcp）。

## 目录结构

```
source/
  main.py                # CLI 入口（argparse + asyncio）
  examples/              # 示例题目 JSON
    questions.json       # 5 个 mock demo 题目
    questions_with_file.json
    files/               # 题目附件
  runtime/               # 竞赛基础设施 —— 参赛者不可修改
    batch_runner.py      # 批量题目运行器
    agent_context.py     # AgentContext 数据类
    agent_registry.py    # 子 Agent 注册/调度
    env_config.py        # .env 加载 + ModelConfig
    mcp_client.py        # 本地 MCP 客户端（工具调用+文件安全）
    mcp_types.py         # MCPTool 数据类 + schema helpers
    openai_chat_client.py # OpenAI 兼容 HTTP 客户端
    question_loader.py   # 题目加载
    question_schema.py   # 题目解析，提取公开字段
    result_writer.py     # 原子化 JSON 结果写入
    skill_runtime.py     # Skill 发现/加载/执行引擎
    show_answers.py      # CLI 查看结果
  solution/              # 参赛者可编辑区域
    contestant_agent.py  # 主 Agent —— 参赛者核心修改点
    sub_agents.py        # BaseSubAgent + ScriptSubAgent 自动发现
    agents/              # 子 Agent 包（每个含 agent.json + AGENT.md + scripts/run.py）
    mcp/                 # 参赛者自定义 MCP 工具
      contestant_tools.py
    skills/              # Skill 包（每个含 skill.json + SKILL.md + scripts/run.py）
  toolkits/              # 工具注册桥接层
    main_mcp.py          # 内置工具注册（text_read_file, skill_load, skill_run 等）
```

## 运行方式

```bash
# 方式 1：Shell 脚本（推荐）
bash start.sh <question_path> <result_path> [package_id]
# 示例：
bash start.sh source/examples/questions.json source/outputs/result.json

# 方式 2：直接 Python
python -m source.main --question source/examples/questions.json --output source/outputs/result.json

# 查看结果
python -m source.runtime.show_answers source/outputs/result.json
```

## 环境配置

运行前必须在 `.env` 中配置：

| 变量 | 说明 |
|---|---|
| `MODEL_BASE_URL` | 模型网关基 URL（自动拼接 `/chat/completions`） |
| `MODEL_API_KEY` | API 密钥 |
| `MODEL_NAME` | 模型名称 |

可选配置：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `AGENT_DEMO_USE_LLM` | 1 | 是否使用 LLM |
| `AGENT_DEMO_NATIVE_TOOLS` | 1 | 使用 OpenAI 原生工具调用 |
| `AGENT_DEMO_JSON_TOOL_FALLBACK` | 1 | JSON 工具调用降级 |
| `AGENT_DEMO_MAX_ITER` | 10 | 最大工具调用迭代次数 |
| `AGENT_DEMO_TEMPERATURE` | 0.2 | LLM 温度 |
| `AGENT_DEMO_TIMEOUT_SECONDS` | 120 | HTTP 超时 |
| `AGENT_DEMO_MAX_TOKENS` | 3200 | 最大 token 数 |
| `AGENT_DEMO_STREAM` | 1 | SSE 流式响应 |

## 核心架构

### 两大区域

- **Runtime 区**（`source/runtime/`）——基础设施，参赛者不可修改
- **Solution 区**（`source/solution/`）——参赛者可编辑

### 执行流程

1. `main.py` → `BatchRunner.run_file()` 加载题目
2. 每题 → `_run_one()` 构建 `AgentContext` → `ContestantAgent.solve()`
3. Agent 通过 `context.call_tool()` → `LocalMCPClient.call_tool()` 调用工具
4. `public_question()` 过滤私有字段，Agent 只看到 `id`、`question`、`files`
5. 结果原子化写入 JSON

### Agent 工具调用模式

`ContestantAgent.solve()` 支持两种模式：
- **Native Tool Loop** — OpenAI 原生函数调用（优先）
- **JSON Tool Loop** — 当网关不支持 `tools` 字段时的降级方案

### 内置工具

| 工具 | 类型 | 作用 |
|---|---|---|
| `text_read_file` | skill | 读取题目声明的文件 |
| `skill_load` | skill | 加载 SKILL.md 指令 |
| `skill_read_resource` | skill | 读取技能资源文件 |
| `skill_run` | skill | 执行技能入口脚本 |
| `agent_delegate` | agent | 委派任务给子 Agent |

## 参赛者修改指南

主要修改点：

1. **`source/solution/contestant_agent.py`** — `ContestantAgent.solve()` 方法，核心逻辑
2. **`source/solution/mcp/contestant_tools.py`** — 自定义 MCP 工具
3. **`source/solution/agents/`** — 新增子 Agent 包（含 `agent.json` + `AGENT.md` + `scripts/run.py`）
4. **`source/solution/skills/`** — 新增 Skill 包（含 `skill.json` + `SKILL.md` + `scripts/run.py`）

### 子 Agent 包结构

```
agents/<agent_name>/
  agent.json          # {name, description, role, entrypoint, timeout_seconds}
  AGENT.md            # 子 Agent 描述
  scripts/run.py      # 入口脚本（stdin 接收 JSON，stdout 输出结果）
```

### Skill 包结构

```
skills/<skill_name>/
  skill.json          # {name, description, entrypoint, timeout_seconds, input_schema}
  SKILL.md            # YAML frontmatter + 指令正文
  scripts/run.py      # 入口脚本（stdin 接收 JSON，stdout 输出结果）
  references/         # 可选：参考资料（通过 skill_read_resource 读取）
  assets/             # 可选：资源文件
```

### 自定义 MCP 工具

在 `contestant_tools.py` 中通过 `register_tools(register_tool, object_schema)` 函数注册：

```python
def register_tools(*, register_tool, object_schema):
    @register_tool(
        name="my_tool",
        description="...",
        input_schema=object_schema({...),
    )
    def my_tool(...):
        ...
```

## 安全约束

- 文件读取：只能读取题目 `files` 字段声明的文件，路径遍历被阻止
- 题面过滤：Agent 只能看到 `id`、`question`、`files`，私有字段被移除
- Skill 资源：只能读取 `references/` 和 `assets/` 下的文件

## 注意事项

- `requirements.txt` 为空，项目仅依赖 Python 标准库
- `start.sh` 中 venv 创建和 pip install 默认注释掉，如需第三方包需取消注释并填写 `requirements.txt`
- `package_id` 作为 HTTP header 传递给模型网关
- 工具调用结果截断到 12000 字符，SKILL.md 截断到 20000 字符，文件读取截断到 64000 字符
- Windows/MINGW 环境下 `start.sh` 会优先检测 `python` 命令