# EACP (环境感知上下文协议) 草案规范 v0.1

**版本**: 0.1.0（草案）  
**最后更新**: 2026-06-04  
**状态**: 草案 / 评审中

---

## 1. 概述与目标

**环境感知上下文协议 (Environment-Aware Context Protocol, EACP)** 是一个**框架无关的开放协议**，旨在使 AI 编程助手能够“理解”用户特定的本地执行环境，并生成针对该环境优化的代码。

EACP 不是静态的“环境快照”，而是一个**基于 MCP (Model Context Protocol) 服务器实现的动态查询型协议**。AI 代理可以按需获取所需信息，兼顾实时性与上下文窗口的节约。

### 1.1 解决的问题

- **环境假设错误**：“我用 Python 3.12 写的，但实际环境是 3.10。”
- **端口冲突**：“我写的代码要在 8080 端口启动，但该端口已被占用。”
- **重复发明**：“我重新实现了一个摘要功能，但本地已经存在相同功能的组件。”
- **权限/约束违反**：“我试图写入 `/etc/` 文件，但没有权限。”

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **面向决策** | 提供的是“AI 做出判断所需的材料”，而非原始数据本身。 |
| **惰性/按需** | 不一次性推送全部信息，而是让 AI 按需调用工具。 |
| **默认隐私** | 敏感信息（密码、API 密钥）**隐藏值，仅返回元数据**（如键名列表）。 |
| **框架无关** | 不依赖任何特定框架（LAAS/AAF/LangChain 等）。任何框架可通过 `project_context` 声明其名称。 |
| **可组合** | 支持部分获取，AI 代理可以只查询所需的层级。 |
| **人类意图感知** | 在上下文中不仅包含环境数据，还包含人类的目标、优先级和约束。 |

---

## 2. 信息模型（7 个类别）

EACP 提供的信息分为以下 7 个类别。AI 代理根据需要分别查询。

```text
EACP
├── 1. System & Hardware        ... 操作系统、架构、CPU/GPU/内存
├── 2. Runtime & Toolchain      ... 语言版本、包管理器、已安装的库
├── 3. Network & Ports          ... IP 地址、端口使用情况、代理、防火墙
├── 4. Ecosystem & Assets       ... 运行中的组件 & 能力图谱（核心）
├── 5. Filesystem & Permissions ... 可读写路径、禁止路径、gitignore
├── 6. Policies & Constraints   ... 设计规则、安全策略、允许/禁止的模式
└── 7. Intent & Context         ... 人类的目标、优先级、需要避免的事项（可选但重要）
```

### 2.1 系统与硬件（L0）

AI 判断“该环境下能做什么”的基础信息。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `os.type` | enum | `"linux"`, `"darwin"`, `"windows"` | 路径分隔符、系统调用、包管理器的差异 |
| `os.version` | string | `"24.04"`, `"14.5"`, `"22H2"` | 特定版本的功能可用性 |
| `os.wsl` | bool | `true` | WSL2 与原生 Linux 行为不同（路径转换、systemd 等） |
| `arch` | enum | `"x86_64"`, `"arm64"` | 二进制/Wheel 的选择 |
| `cpu.logical_cores` | int | `16` | 并行度决策 |
| `ram.total_mb` | int | `32768` | 本地 LLM 可行性、批处理大小 |
| `ram.available_mb` | int | `18432` | 当前可用资源 |
| `gpu.available` | bool | `true` | 是否生成 CUDA/MPS/Metal 代码 |
| `gpu.vendor` | string | `"nvidia"`, `"apple"` | 后端选择 |
| `gpu.vram_mb` | int | `12288` | 可加载的模型大小 |
| `gpu.cuda_version` | string | `"12.4"` | 依赖版本固定 |
| `disk.free_gb` | int | `120` | 是否可下载大模型/数据库 |

### 2.2 运行时与工具链（L1）

用于决定“如何编写和运行代码”的信息。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `runtime.python.version` | string | `"3.12.3"` | 语法可用性（match 语句、类型提示） |
| `runtime.python.versions_available` | [string] | `["3.10", "3.12"]` | 多版本环境下的选择 |
| `runtime.package_manager.type` | enum | `"uv"`, `"poetry"`, `"pip"`, `"conda"` | 统一安装命令 |
| `runtime.package_manager.version` | string | `"0.5.1"` | 兼容性检查 |
| `runtime.shell.type` | enum | `"bash"`, `"zsh"`, `"powershell"` | 环境变量设置语法 |
| `runtime.venv.path` | string | `"/home/user/project/.venv"` | 显式虚拟环境路径 |
| `runtime.binaries.available` | [string] | `["git", "docker", "sqlite3"]` | 是否可假定 Docker、SQLite 等已安装 |

### 2.3 网络与端口（L2）

**避免端口冲突、正确绑定的关键信息。**

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `network.lan_ip` | string | `"192.168.1.50"` | 生成局域网可访问的 URL |
| `network.loopback` | string | `"127.0.0.1"` | 默认绑定地址 |
| `network.used_ports` | [int] | `[4000, 8001, 10423]` | 黑名单——绝不能分配的端口 |
| `network.available_port_ranges` | [obj] | `[{"start":8000,"end":8999}]` | 系统推荐的端口范围 |
| `network.suggested_next_port` | int | `8002` | 可立即使用的代表性端口 |
| `network.localhost_only_default` | bool | `true` | 安全策略是否强制绑定 `127.0.0.1` |
| `network.proxy.http` | string | `"http://proxy:8080"` | 自动注入外部 API 调用代码 |
| `network.firewall.enforced` | bool | `true` | 防火墙是否启用 |

### 2.4 生态系统与资产（L3）——能力图谱

**EACP 的核心**。能力图谱帮助 AI 判断“创建新组件还是复用现有组件”。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `ecosystem.components` | [obj] | 见下文 | 可复用的组件 |
| `ecosystem.mcp_tools` | [obj] | 见下文 | 外部 MCP 服务器 |
| `ecosystem.llm_gateways` | [obj] | 见下文 | LiteLLM 等统一端点 |

#### 组件对象

```json
{
  "name": "summarize_agent",
  "version": "1.0.0",
  "capabilities": ["summarization", "text_processing"],
  "endpoint": "http://127.0.0.1:8001",
  "input_schema": { "prompt": "string" },
  "output_schema": { "summary": "string" },
  "deployment_mode": "standalone",
  "status": "running"
}
```

#### MCP 工具对象

```json
{
  "name": "filesystem",
  "transport": "stdio",
  "capabilities": ["read_file", "write_file", "list_directory"]
}
```

#### LLM 网关对象

```json
{
  "name": "litellm_proxy",
  "endpoint": "http://127.0.0.1:4000",
  "configured_models": ["gpt-4o", "ollama/llama3"]
}
```

### 2.5 文件系统与权限（L4）

使 AI 能够提前知道“哪些地方允许创建文件”。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `fs.project_root` | path | `"/home/user/projects/myapp"` | 相对路径的基准 |
| `fs.cwd` | path | `"/home/user/projects/myapp"` | 当前工作目录 |
| `fs.writable_paths` | [path] | `["~/projects", "~/.local/share"]` | 允许写入的白名单 |
| `fs.readable_paths` | [path] | `["~/documents", "/etc/ssl"]` | 允许读取的路径 |
| `fs.forbidden_paths` | [path] | `["~/.ssh", "/etc/passwd", "/System"]` | **绝对禁止访问** |
| `fs.sticky_dirs` | [path] | `["~/.local/share/aaf", "~/.cache"]` | 持久化数据的推荐位置 |
| `fs.env_files` | [string] | `[".env", ".env.local"]` | 机密信息的候选存放位置 |
| `fs.gitignored` | [string] | `[".env", "*.db", "__pycache__/"]` | 防止误提交的 Git 忽略模式 |

### 2.6 策略与约束（L5）

告诉 AI“不能做什么”的设计规则。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `policies.security.bind_host_default` | enum | `"127.0.0.1"` | 默认绑定限制 |
| `policies.security.secrets_mgmt` | enum | `"env_file"` | 机密信息的处理方式（env_file / keyring / none） |
| `policies.security.network_egress` | enum | `"allow_all"`, `"offline"`, `"whitelist"` | 外部通信策略 |
| `policies.code.style` | enum | `"black"`, `"ruff"`, `"pep8"` | 代码格式化风格 |
| `policies.code.async_preference` | enum | `"always"`, `"only_if_needed"`, `"never"` | 异步编程偏好 |
| `policies.code.prohibited_patterns` | [string] | `["dynamic_port_assignment", "bind_0.0.0.0"]` | **禁止的模式** |
| `policies.code.preferred_patterns` | [string] | `["fixed_port_via_env", "trace_id_propagation"]` | **推荐的模式** |
| `policies.testing.required` | bool | `true` | 是否必须生成测试代码 |
| `policies.framework.name` | string | `"laas"`, `"langchain"`, `"custom"` | 框架名称（保持框架无关的同时应用其规则） |
| `policies.framework.version` | string | `"1.1.0"` | 框架版本 |

### 2.7 意图与上下文（可选但重要）

**将人类的目标融入环境上下文**（整合自 Masaru/ChatGPT5 提案）。

| 字段 | 类型 | 示例 | 依据 |
|------|------|------|------|
| `intent.goal` | string | `"整合本地文档的摘要与搜索"` | 生成代码的方向 |
| `intent.priority` | enum | `"low_latency"`, `"accuracy"`, `"low_cost"` | 权衡指导 |
| `intent.avoid` | [string] | `["duplicate_implementation", "external_api_dependency"]` | 需要避免的实现 |
| `intent.preferred_tech` | [string] | `["sqlite", "fastapi"]` | 用户偏好的技术 |
| `intent.notes` | string | `"内存不足16GB时不要使用大型模型"` | 自由形式的补充说明 |

---

## 3. MCP 服务器的工具定义

EACP 提供以下 MCP `Tool`。AI 代理按需调用。

### 3.1 按类别获取的工具

| 工具名称 | 说明 | 输入 | 输出 |
|---------|------|------|------|
| `eacp_get_system_profile` | 获取 L0（系统与硬件） | `{}` | System & Hardware |
| `eacp_get_runtime` | 获取 L1（运行时） | `{}` | Runtime & Toolchain |
| `eacp_get_network` | 获取 L2（网络与端口） | `{}` | Network & Ports |
| `eacp_query_ecosystem` | 获取 L3（生态系统），支持能力过滤器 | `{ "capability": "summarization" }` | 匹配的组件列表 |
| `eacp_get_filesystem` | 获取 L4（文件系统与权限） | `{}` | Filesystem & Permissions |
| `eacp_get_policies` | 获取 L5（策略） | `{}` | Policies & Constraints |
| `eacp_get_intent` | 获取 L7（意图） | `{}` | Intent & Context |

### 3.2 实用工具 / 决策支持工具

| 工具名称 | 说明 | 输入 | 输出 |
|---------|------|------|------|
| `eacp_find_available_port` | **保留并返回**指定范围内的一个空闲端口 | `{ "preferred_range": { "start": 8000, "end": 8999 } }` | `{ "port": 8003 }` |
| `eacp_check_capability` | 检查具有指定能力的组件是否存在 | `{ "capability": "summarization" }` | `{ "found": true, "component": {...} }` |
| `eacp_check_path_permission` | 查询特定路径的读/写权限 | `{ "path": "/home/user/data", "mode": "write" }` | `{ "allowed": true }` |
| `eacp_get_full_snapshot` | 一次性获取**所有类别（L0–L6）**（用于静态上下文包生成） | `{ "include_sensitive": false }` | 合并后的全类别 JSON |

### 3.3 工具调用流程示例

```
用户: “添加本地文档搜索和摘要功能。”

AI:
  1. eacp_get_system_profile()
     → Mac(arm64), RAM 16GB, GPU(MPS) 存在
     → “不要编写 CUDA 代码；使用 MPS 或 CPU 实现。”

  2. eacp_query_ecosystem({ "capability": "semantic_search" })
     → rag_component (port 8002) 已存在
     → “搜索功能已经存在。不要重新实现，直接调用。”

  3. eacp_check_capability({ "capability": "summarization" })
     → summarize_agent (port 8001) 已存在
     → “摘要功能也存在。编写一个包装器，同时调用两个组件。”

  4. eacp_find_available_port({ "preferred_range": { "start": 8000, "end": 8999 } })
     → 分配端口 8005

  5. eacp_check_path_permission({ "path": "/home/user/docs", "mode": "read" })
     → { allowed: true }

  6. eacp_get_policies()
     → localhost_only: true, secrets_mgmt: env_file, async: always

→ 生成的代码:
   端口=8005, 绑定 127.0.0.1, uv add, asyncio,
   读取 /home/user/docs, 调用 rag_component(8002) → summarize_agent(8001)
```

---

## 4. 数据模式与版本管理

### 4.1 模式定义

每个 EACP 响应都使用 **JSON Schema** 进行类型定义，以确保机器可读性和向后兼容性。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "eacp_version": "0.1",
  "type": "object",
  "required": ["eacp_version", "generated_at"],
  "properties": {
    "eacp_version": { "type": "string", "const": "0.1" },
    "generated_at": { "type": "string", "format": "date-time" }
  }
}
```

### 4.2 必需字段与可选字段

| 类别 | 必需程度 | 理由 |
|------|---------|------|
| L0 系统与硬件 | **必需** | 决定代码的基本假设 |
| L1 运行时 | **必需** | 依赖解决和语法选择的基础 |
| L2 网络 | **必需** | 端口冲突避免的核心 |
| L3 生态系统 | **必需** | 防止重复发明的核心（能力图谱） |
| L4 文件系统 | **推荐** | 安全文件操作所需 |
| L5 策略 | **推荐** | 自动遵守设计规则所需 |
| L7 意图 | **可选** | 若存在，可显著提高准确性 |

### 4.3 扩展字段

框架特定的信息放在 `extensions` 对象中，以保持核心协议的通用性。

```json
{
  "extensions": {
    "laas": {
      "registry_path": "~/.local/share/aaf/registry.db",
      "lite_llm_endpoint": "http://127.0.0.1:4000"
    },
    "custom": {
      "any_key": "any_value"
    }
  }
}
```

---

## 5. 安全与隐私

### 5.1 默认返回最少信息

- **标准输出**：L0 + L1（部分）+ L2（端口列表）+ L3（组件名称、能力、端点信息）。
- **机密信息处理**：
  - API 密钥、密码、令牌**绝不返回值**。服务器可以仅返回密钥名称列表（`available_secret_keys`）。
  - **不读取** `.env` 文件的内容。
  - 可以阻止对 `forbidden_paths` 中路径的访问，甚至不让 AI 感知到它们的存在。
- **敏感信息的披露（L4/L5）**：只有在 EACP 服务器以 `--expose-sensitive` 启动或通过配置文件明确允许时，才返回详细信息。

### 5.2 TTL 与缓存策略（MCP 服务器实现建议）

| 类别 | TTL | 理由 |
|------|-----|------|
| L0 系统 | 会话期间缓存 | 几乎不变 |
| L1 运行时 | 1 小时 | 包可能被添加或更新 |
| L2 网络 | **10 秒** | 端口使用情况变化迅速 |
| L3 生态系统 | 30 秒 | 组件启动/停止频繁 |
| L4 文件系统 | 5 分钟 | 变化不频繁 |
| L5 策略 | 会话期间缓存 | 用户设置通常不会在运行中改变 |

---

## 6. 框架无关性策略

EACP 不限于任何特定框架（不绑定 LAAS/AAF）。它可以与任何本地 AI 应用基础设施一起使用。

- `policies.framework.name` 字段可以声明 `"laas"`, `"langchain"`, `"crewai"`, `"comfyui"`, `"custom"` 等。
- 每个框架可以在 `extensions.<framework_name>` 内定义其自己的模式。
- LAAS/AAF 将提供 EACP 的**参考实现**，但协议本身是开放的。

---

## 7. 实现路线图

### 阶段 1：EACP 概念验证（2–4 周）
**目标**：协议声明 + 可工作的最小 MCP 服务器。

- 建立 GitHub 仓库
- `eacp_get_system_profile`（L0）
- `eacp_find_available_port`（L2 核心）
- `eacp_query_ecosystem`（L3 核心，带能力过滤器）
- `eacp_get_full_snapshot`（批量获取）

即使是这样最小的集合，也可以显著减少 AI 代理的“端口冲突”和“重复发明”问题。

### 阶段 2：EACP v0.5（2–3 个月）
- `eacp_get_filesystem`（L4）和路径权限检查
- `eacp_get_policies`（L5）设计规则支持
- 更严格的 JSON Schema 和公开的校验器
- 通过 `extensions` 支持 LAAS/AAF 扩展字段

### 阶段 3：EACP v1.0 RFC（6 个月以上）
- 标准化 `eacp_get_intent`（L7）：让 EACP 服务器通过向导式对话捕获人类意图
- 双向功能：`eacp_propose_allocation`（AI 请求预留端口 → EACP 锁定并响应）
- 多方（多个 LLM 供应商、框架）交叉评审和标准化

---

## 8. 结论：为什么是“协议”？

传统的“上下文包”（ZIP 快照）存在**快照迅速过时且浪费 AI 上下文窗口**的局限性。

EACP 将其重新定义为**基于 MCP 的动态协议**，从而实现：

1. **实时性** – 端口和进程状态不断变化。按需查询，不依赖静态文件。
2. **恰到好处的信息** – AI 只获取所需信息，节约 Token。
3. **防止重复发明**（通过能力图谱）– AI 按能力搜索已在运行的组件，复用现有资产。
4. **安全执行**（通过策略与约束）– AI 从系统学习“什么不能做”，自动遵守用户策略。
5. **框架无关** – EACP 可以成为任何本地 AI 开发环境的标准，而不仅仅是 LAAS/AAF。

**下一步行动**：
1. 将本草案规范以 Markdown 格式发布到 GitHub。
2. 实现一个最小的 Python MCP 服务器（FastAPI + `mcp` SDK），提供 `eacp_find_available_port` 和 `eacp_query_ecosystem` 作为原型。
3. 连接到 Claude Code / Cursor 等，发布演示视频，展示“使用 EACP 实现本地环境优化的代码生成”。

这就是一个**中立的、开放的“现实约束模型”**的价值，它架起了 AI 编程助手与本地环境之间的桥梁。

---

*EACP 草案规范 v0.1*  
*贡献者：LAAS/AAF 项目*
