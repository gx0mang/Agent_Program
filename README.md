<div align="center">

# 智扫通 · 扫地机器人智能客服 Agent

**基于 LangChain `create_agent` + ReAct 范式 + RAG 检索增强的智能客服系统，以扫地机器人为示例场景**

[![Python](https://img.shields.io/badge/Python-3.13-blue)](https://www.python.org/)
&nbsp;
[![LangChain](https://img.shields.io/badge/LangChain-1.3.15-green)](https://www.langchain.com/)
&nbsp;
[![LangGraph](https://img.shields.io/badge/LangGraph-1.2.11-orange)](https://github.com/langchain-ai/langgraph)
&nbsp;
[![Chroma](https://img.shields.io/badge/Chroma-1.5.9-blueviolet)](https://www.trychroma.com/)
&nbsp;
[![Streamlit](https://img.shields.io/badge/Streamlit-1.62.0-red)](https://streamlit.io/)
&nbsp;
[![License](https://img.shields.io/badge/License-MIT-yellow)](./LICENSE)

</div>

---

## 项目简介

基于 LangChain 新版 `create_agent` 构建的扫地机器人领域智能客服。Agent 以 ReAct 方式自主决策 —— 判断是否需要调用工具、调用哪一个、是否需要二次调用，并结合向量库检索到的资料生成回答。

系统支持两类任务：**知识问答**（选购 / 使用 / 故障 / 保养）与**使用报告生成**。二者共用同一套模型与工具，通过中间件依据运行时上下文自动切换提示词，无需为报告场景单独维护一套 Agent。

> **说明**：本项目为个人学习实践，其中 4 个外部数据工具为模拟实现，业务数据亦为构造数据，详见文末「已知限制」。

## 效果展示

<!-- 截图待补充：建议放置三张 —— 知识库问答 / 工具调用链路 / 报告生成结果 -->

<div align="center">

_（界面截图待补充）_

</div>

## 技术架构

<div align="center">

```
┌────────────────────────────────────────────────────────┐
│  app.py · Streamlit 交互层                             │
│  流式逐字输出 · 多轮会话记忆 · 用户输入                │
└────────────────────────────┬───────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────┐
│  agent/react_agent.py · ReAct Agent                    │
│  create_agent(model, system_prompt, tools, middleware) │
│                                                        │
│     Thought  ──→  Action  ──→  Observation             │
│         ↑                            │                 │
│         └────────────────────────────┘                 │
│                                                        │
│  Middleware：工具监控 · 模型日志 · 动态提示词切换      │
└────────────────────────────┬───────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ RAG 检索       │  │ Tools 工具     │  │ Prompt 提示词  │
├────────────────┤  ├────────────────┤  ├────────────────┤
│ Chroma 向量库  │  │ 7 个工具       │  │ 客服 / 报告    │
│ MD5 去重       │  │ 按需调用       │  │ 动态切换       │
└────────────────┘  └────────────────┘  └────────────────┘
```

</div>

### 核心特性

| 特性 | 说明 |
| --- | --- |
| **ReAct 自主决策** | Thought → Action → Observation 循环，不预设固定流程，由模型判断工具调用时机与顺序 |
| **RAG 检索增强** | Chroma 向量库 + DashScope 嵌入，MD5 文件去重，支持 txt / pdf 混合加载 |
| **动态提示词切换** | 中间件依据运行时上下文，在同一套工具上切换「客服」与「报告写手」两套 System Prompt |
| **中间件可观测** | 工具调用与模型调用全链路结构化日志，异常可追溯 |
| **报告生成闭环** | 从用户意图识别到数据拉取、报告成文，由多工具按序协作完成 |
| **配置驱动** | 模型、分片、检索参数走 YAML，提示词独立成文件，调整行为无需改动代码 |

### 上下文驱动的提示词切换

项目中最能体现 Agent 编排能力的一处设计，是「报告生成」这条路径：

1. 用户提出「生成我的使用报告」，模型识别出报告意图，调用 `fill_context_for_report`。
2. 该工具本身不产生业务数据，作用是让 `monitor_tool` 中间件拦截到这次调用，并向运行时上下文写入标记 `context["report"] = True`。
3. `report_prompt_switch` 中间件（`@dynamic_prompt`）在下一次生成提示词前读取该标记，把系统提示词从「客服」切换为「报告写手」。
4. 模型在报告提示词的约束下，按固定顺序调用 `get_user_id` → `get_current_month` → `fetch_external_data`，取出该用户该月的使用记录。
5. 输出一份 Markdown 格式的使用情况报告与保养建议。

## 技术栈

| 层级 | 技术 | 版本 |
| --- | --- | --- |
| 语言 / 运行时 | Python | 3.13 |
| LLM | 通义千问（DashScope / ChatTongyi，qwen3-max） | dashscope 1.27.0 |
| Agent 框架 | LangChain（`create_agent`）+ LangGraph | 1.3.15 / 1.2.11 |
| 向量数据库 | Chroma | 1.5.9 |
| 嵌入模型 | DashScope Embeddings（text-embedding-v4） | — |
| 文档处理 | PyPDF + RecursiveCharacterTextSplitter | pypdf 6.16.1 |
| 前端 | Streamlit | 1.62.0 |
| 配置 / 日志 | PyYAML + logging | PyYAML 6.0.3 |

## 快速开始

### 环境要求

- **Python** 3.13
- **DashScope API Key**（[阿里云百炼控制台](https://bailian.console.aliyun.com/) 申请）

### 1. 克隆仓库

```bash
git clone https://github.com/gx0mang/Agent_Program.git
cd Agent_Program
```

### 2. 安装依赖

```bash
python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

### 3. 配置 API Key

参考 `.env.example`，设置阿里云百炼 API Key：

```bash
# Windows (CMD)
set DASHSCOPE_API_KEY=your-api-key

# Windows (PowerShell)
$env:DASHSCOPE_API_KEY="your-api-key"

# macOS / Linux
export DASHSCOPE_API_KEY="your-api-key"
```

### 4. 初始化知识库（首次运行）

**请在项目根目录下执行**：

```bash
python -m rag.vector_store
```

该脚本会读取 `data/` 下所有 txt / pdf 文件，分片、向量化后写入 Chroma。已入库的文件（MD5 记录在 `md5.txt`）会被自动跳过，可重复执行。

> 注意：不要用 `python rag/vector_store.py` 直接执行脚本 —— 代码使用 `from utils.xxx` 这类基于项目根目录的绝对导入，直接运行会因 `sys.path` 不含根目录而失败。

### 5. 启动应用

```bash
streamlit run app.py
```

浏览器自动打开 http://localhost:8501

### 验证运行

启动后在聊天框输入以下测试问题：

- *小户型适合什么样的扫地机器人？*（RAG 知识库问答）
- *我所在的城市当前天气下该如何保养？*（工具调用：定位 + 天气 + 知识库）
- *生成我的使用报告*（报告生成：上下文注入 + 动态提示词切换）

## 项目结构

```
Agent_Program/

├── agent/                            # Agent 核心
│   ├── react_agent.py                # Agent 装配与流式执行
│   └── tools/
│       ├── agent_tools.py            # 工具定义（7 个）
│       └── middleware.py             # 中间件定义（3 个）
│
├── rag/                              # RAG 检索增强
│   ├── vector_store.py               # Chroma 向量库 · 分片 · MD5 去重
│   └── rag_service.py                # 检索 → 拼接上下文 → 模型总结
│
├── model/
│   └── factory.py                    # 模型工厂（对话模型 / 嵌入模型）
│
├── utils/                            # 通用工具
│   ├── config_handler.py             # YAML 配置加载
│   ├── path_tool.py                  # 项目路径解析
│   ├── prompt_loader.py              # 提示词加载
│   ├── file_handler.py               # 文档解析与 MD5 计算
│   └── logger_handler.py             # 日志管理
│
├── config/                           # YAML 配置
│   ├── agent.yml                     # 外部数据路径
│   ├── rag.yml                       # 对话模型与嵌入模型名称
│   ├── chroma.yml                    # 向量库与分片参数
│   └── prompts.yml                   # 提示词文件路径
│
├── prompts/                          # 提示词模板
│   ├── main_prompt.txt               # 客服 System Prompt
│   ├── rag_summarize.txt             # RAG 总结 Prompt
│   └── report_prompt.txt             # 报告生成 System Prompt
│
├── data/                             # 知识库文档与业务数据
│   ├── 扫地机器人100问.txt
│   ├── 选购指南.txt
│   ├── 故障排除.txt
│   ├── 维护保养.txt
│   └── external/records.csv          # 用户使用记录（构造数据）
│
├── assets/                           # 效果展示截图（待补充）
├── app.py                            # Streamlit 应用入口
├── requirements.txt
├── .env.example                      # 环境变量模板
├── LICENSE                           # MIT 开源许可证
└── README.md
```

## 工具与中间件

### 工具（`agent/tools/agent_tools.py`）

| 工具 | 入参 | 说明 | 数据来源 |
| --- | --- | --- | --- |
| `rag_summarize` | `query` | 从向量库检索资料并总结 | `data/` 知识库（**真实检索**） |
| `get_weather` | `city` | 获取指定城市天气 | 模拟 |
| `get_user_location` | — | 获取用户所在城市 | 模拟 |
| `get_user_id` | — | 获取用户 ID | 模拟 |
| `get_current_month` | — | 获取当前月份 | 模拟 |
| `fetch_external_data` | `user_id`, `month` | 查询用户指定月份的使用记录 | `data/external/records.csv`（构造数据） |
| `fill_context_for_report` | — | 触发中间件注入报告上下文标记 | — |

### 中间件（`agent/tools/middleware.py`）

| 中间件 | 装饰器 | 作用 |
| --- | --- | --- |
| `monitor_tool` | `@wrap_tool_call` | 记录工具名与入参、捕获异常；拦截 `fill_context_for_report` 写入报告标记 |
| `log_before_model` | `@before_model` | 模型调用前记录当前消息数量与最新消息 |
| `report_prompt_switch` | `@dynamic_prompt` | 依据运行时上下文动态选择系统提示词 |

## 配置说明

项目通过 `config/` 目录下的 YAML 文件统一管理配置：

| 文件 | 说明 |
| --- | --- |
| `rag.yml` | 对话模型名称、嵌入模型名称 |
| `chroma.yml` | 向量库集合名、持久化路径、检索 Top-K、分片参数、支持的文件类型、MD5 记录文件 |
| `prompts.yml` | 各场景提示词模板文件路径 |
| `agent.yml` | 外部数据文件路径 |

首次运行只需确保 **DashScope API Key 已设置**，且 `data/` 目录下有知识库文档即可。

## 已知限制

本项目为学习实践，以下几处需要说明：

1. **4 个外部数据工具为模拟实现**。`get_weather`、`get_user_location`、`get_user_id`、`get_current_month` 均返回预设的固定值或随机值，未接入任何真实数据源。
2. **月份数据固定在 2025 年**。`get_current_month` 的可选值数组写死为 2025 年 1–12 月。
3. **业务数据为构造数据**。`data/external/records.csv` 中的用户使用记录为人工构造，仅用于演示报告生成的完整链路。
4. **向量库持久化路径为相对路径**。`config/chroma.yml` 中的 `persist_directory` 未做绝对路径解析，实际存储位置会随执行时的工作目录变化，不同目录下运行可能生成多份索引。
5. 暂未编写自动化测试。

只有 `rag_summarize` 是基于真实知识库的语义检索；其余涉及「用户」「天气」的能力均为占位实现。

## License

本项目基于 [MIT License](./LICENSE) 开源。

---

仓库地址：https://github.com/gx0mang/Agent_Program
