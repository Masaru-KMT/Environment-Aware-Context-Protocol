# EACP (Environment-Aware Context Protocol) Draft Specification v0.1

**Version**: 0.1.0 (Draft)  
**Last Updated**: 2026-06-04  
**Status**: Draft / Review

---

## 1. Overview and Purpose

**Environment-Aware Context Protocol (EACP)** is a **framework-agnostic open protocol** that enables AI coding agents to “understand” a user’s specific local execution environment and generate code that is optimally adapted to that environment.

EACP is not a static “environment snapshot” but a **dynamic query‑based protocol implemented as an MCP (Model Context Protocol) server**. This allows AI agents to retrieve only the information they need, when they need it, balancing real‑time awareness with context window economy.

### 1.1 Problems Solved

- **Incorrect environment assumptions**: “I wrote for Python 3.12, but the actual environment is 3.10.”
- **Port conflicts**: “I wrote code that starts on port 8080, but it was already in use.”
- **Reinvention**: “I implemented a summarization feature from scratch, but a local component already provided it.”
- **Permission / constraint violations**: “I tried to write a file to `/etc/`, but I had no permission.”

### 1.2 Design Principles

| Principle | Description |
|-----------|-------------|
| **Decision‑Oriented** | Provide information that helps the AI make decisions, not raw data for its own sake. |
| **Lazy / On‑Demand** | Do not push all information at once; let the AI call only the tools it needs. |
| **Privacy‑by‑Default** | Sensitive information (passwords, API keys) is **hidden; only metadata** (e.g., key names) may be returned. |
| **Framework‑Agnostic** | No dependency on any specific framework (LAAS, AAF, LangChain, etc.). Any framework can declare its name via `project_context`. |
| **Composable** | Partial retrieval is supported. The AI agent can query only the layers it requires. |
| **Human‑Intent Aware** | Include not only environment data but also the human’s goal, priorities, and constraints in the context. |

---

## 2. Information Model (7 Categories)

The information provided by EACP is divided into the following 7 categories. The AI agent queries them individually as needed.

```text
EACP
├── 1. System & Hardware        ... OS, architecture, CPU/GPU/memory
├── 2. Runtime & Toolchain      ... language version, package manager, installed libraries
├── 3. Network & Ports          ... IP addresses, port usage, proxy, firewall
├── 4. Ecosystem & Assets       ... running components & capability graph (core)
├── 5. Filesystem & Permissions ... readable/writable paths, forbidden paths, gitignore
├── 6. Policies & Constraints   ... design rules, security policies, allowed / prohibited patterns
└── 7. Intent & Context         ... human goal, priorities, things to avoid (optional but important)
```

### 2.1 System & Hardware (L0)

Basic information for the AI to determine “what is possible in this environment”.

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `os.type` | enum | `"linux"`, `"darwin"`, `"windows"` | Path separators, system calls, package manager differences |
| `os.version` | string | `"24.04"`, `"14.5"`, `"22H2"` | Feature availability per OS version |
| `os.wsl` | bool | `true` | WSL2 behaves differently from native Linux (path conversion, systemd, etc.) |
| `arch` | enum | `"x86_64"`, `"arm64"` | Selection of binaries / wheels |
| `cpu.logical_cores` | int | `16` | Parallelism decisions |
| `ram.total_mb` | int | `32768` | Feasibility of local LLM, batch size |
| `ram.available_mb` | int | `18432` | Currently usable resources |
| `gpu.available` | bool | `true` | Whether to generate CUDA/MPS/Metal code |
| `gpu.vendor` | string | `"nvidia"`, `"apple"` | Backend selection |
| `gpu.vram_mb` | int | `12288` | Model size that can be loaded |
| `gpu.cuda_version` | string | `"12.4"` | Dependency version pinning |
| `disk.free_gb` | int | `120` | Feasibility of downloading large models / databases |

### 2.2 Runtime & Toolchain (L1)

Information for deciding “how to write and run the code”.

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `runtime.python.version` | string | `"3.12.3"` | Syntax availability (match, type hints) |
| `runtime.python.versions_available` | [string] | `["3.10", "3.12"]` | Choices when multiple versions exist |
| `runtime.package_manager.type` | enum | `"uv"`, `"poetry"`, `"pip"`, `"conda"` | Standardized install commands |
| `runtime.package_manager.version` | string | `"0.5.1"` | Compatibility checks |
| `runtime.shell.type` | enum | `"bash"`, `"zsh"`, `"powershell"` | Environment variable syntax |
| `runtime.venv.path` | string | `"/home/user/project/.venv"` | Explicit virtual environment path |
| `runtime.binaries.available` | [string] | `["git", "docker", "sqlite3"]` | Whether Docker, SQLite, etc. can be assumed |

### 2.3 Network & Ports (L2)

**Critical information to avoid port conflicts and bind correctly.**

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `network.lan_ip` | string | `"192.168.1.50"` | Generating LAN‑accessible URLs |
| `network.loopback` | string | `"127.0.0.1"` | Default bind address |
| `network.used_ports` | [int] | `[4000, 8001, 10423]` | Blacklist – never allocate these |
| `network.available_port_ranges` | [obj] | `[{"start":8000,"end":8999}]` | Ranges recommended by the system |
| `network.suggested_next_port` | int | `8002` | A representative value that can be used immediately |
| `network.localhost_only_default` | bool | `true` | Whether security policy enforces `127.0.0.1` binding |
| `network.proxy.http` | string | `"http://proxy:8080"` | Automatically injected into external API call code |
| `network.firewall.enforced` | bool | `true` | Whether a firewall is active |

### 2.4 Ecosystem & Assets (L3) – Capability Graph

**The core of EACP.** A capability graph that helps the AI decide whether to “build or reuse”.

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `ecosystem.components` | [obj] | See below | Reusable components |
| `ecosystem.mcp_tools` | [obj] | See below | External MCP servers |
| `ecosystem.llm_gateways` | [obj] | See below | Unified endpoints such as LiteLLM |

#### Component Object

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

#### MCP Tool Object

```json
{
  "name": "filesystem",
  "transport": "stdio",
  "capabilities": ["read_file", "write_file", "list_directory"]
}
```

#### LLM Gateway Object

```json
{
  "name": "litellm_proxy",
  "endpoint": "http://127.0.0.1:4000",
  "configured_models": ["gpt-4o", "ollama/llama3"]
}
```

### 2.5 Filesystem & Permissions (L4)

Allows the AI to know in advance “where it is allowed to create files”.

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `fs.project_root` | path | `"/home/user/projects/myapp"` | Base for relative paths |
| `fs.cwd` | path | `"/home/user/projects/myapp"` | Current working directory |
| `fs.writable_paths` | [path] | `["~/projects", "~/.local/share"]` | Write‑allowed whitelist |
| `fs.readable_paths` | [path] | `["~/documents", "/etc/ssl"]` | Read‑allowed paths |
| `fs.forbidden_paths` | [path] | `["~/.ssh", "/etc/passwd", "/System"]` | **Absolutely forbidden** |
| `fs.sticky_dirs` | [path] | `["~/.local/share/aaf", "~/.cache"]` | Recommended locations for persistent data |
| `fs.env_files` | [string] | `[".env", ".env.local"]` | Candidate locations for secrets |
| `fs.gitignored` | [string] | `[".env", "*.db", "__pycache__/"]` | Patterns to prevent accidental commits |

### 2.6 Policies & Constraints (L5)

Design rules that tell the AI “what it must not do”.

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `policies.security.bind_host_default` | enum | `"127.0.0.1"` | Default bind restriction |
| `policies.security.secrets_mgmt` | enum | `"env_file"` | How to handle secrets (env_file / keyring / none) |
| `policies.security.network_egress` | enum | `"allow_all"`, `"offline"`, `"whitelist"` | External communication policy |
| `policies.code.style` | enum | `"black"`, `"ruff"`, `"pep8"` | Code formatting style |
| `policies.code.async_preference` | enum | `"always"`, `"only_if_needed"`, `"never"` | Asyncio preference |
| `policies.code.prohibited_patterns` | [string] | `["dynamic_port_assignment", "bind_0.0.0.0"]` | **Prohibited patterns** |
| `policies.code.preferred_patterns` | [string] | `["fixed_port_via_env", "trace_id_propagation"]` | **Recommended patterns** |
| `policies.testing.required` | bool | `true` | Whether test code must be generated |
| `policies.framework.name` | string | `"laas"`, `"langchain"`, `"custom"` | Framework name (keeping framework‑agnostic while applying its rules) |
| `policies.framework.version` | string | `"1.1.0"` | Framework version |

### 2.7 Intent & Context (Optional but Important)

**Embed the human’s purpose into the environment context** (incorporating the Masaru/ChatGPT5 proposal).

| Field | Type | Example | Justification |
|-------|------|---------|----------------|
| `intent.goal` | string | `"Integrate local document summarization and search"` | Direction for generated code |
| `intent.priority` | enum | `"low_latency"`, `"accuracy"`, `"low_cost"` | Trade‑off guidance |
| `intent.avoid` | [string] | `["duplicate_implementation", "external_api_dependency"]` | Implementation to avoid |
| `intent.preferred_tech` | [string] | `["sqlite", "fastapi"]` | User’s technology preferences |
| `intent.notes` | string | `"Do not use heavy models with less than 16GB RAM"` | Free‑form supplementary notes |

---

## 3. Tool Definitions as an MCP Server

EACP provides the following MCP `Tool`s. The AI agent calls them as needed.

### 3.1 Category‑Specific Retrieval Tools

| Tool Name | Description | Input | Output |
|-----------|-------------|-------|--------|
| `eacp_get_system_profile` | Retrieve L0 (System & Hardware) | `{}` | System & Hardware |
| `eacp_get_runtime` | Retrieve L1 (Runtime) | `{}` | Runtime & Toolchain |
| `eacp_get_network` | Retrieve L2 (Network & Ports) | `{}` | Network & Ports |
| `eacp_query_ecosystem` | Retrieve L3 (Ecosystem) with capability filter | `{ "capability": "summarization" }` | Matching components |
| `eacp_get_filesystem` | Retrieve L4 (Filesystem & Permissions) | `{}` | Filesystem & Permissions |
| `eacp_get_policies` | Retrieve L5 (Policies) | `{}` | Policies & Constraints |
| `eacp_get_intent` | Retrieve L7 (Intent) | `{}` | Intent & Context |

### 3.2 Utility / Decision Support Tools

| Tool Name | Description | Input | Output |
|-----------|-------------|-------|--------|
| `eacp_find_available_port` | **Reserve and return** one free port within the specified range | `{ "preferred_range": { "start": 8000, "end": 8999 } }` | `{ "port": 8003 }` |
| `eacp_check_capability` | Check whether a component with a given capability exists | `{ "capability": "summarization" }` | `{ "found": true, "component": {...} }` |
| `eacp_check_path_permission` | Query read/write permission for a specific path | `{ "path": "/home/user/data", "mode": "write" }` | `{ "allowed": true }` |
| `eacp_get_full_snapshot` | Retrieve **all categories (L0–L6)** at once (for static context package generation) | `{ "include_sensitive": false }` | Merged JSON of all categories |

### 3.3 Example Tool Calling Flow

```
User: “Add functionality to search and summarize local documents.”

AI:
  1. eacp_get_system_profile()
     → Mac(arm64), RAM 16GB, GPU(MPS) available
     → “Do not write CUDA code; use MPS or CPU implementation.”

  2. eacp_query_ecosystem({ "capability": "semantic_search" })
     → rag_component (port 8002) found
     → “Search functionality already exists. Do not re‑implement; just call it.”

  3. eacp_check_capability({ "capability": "summarization" })
     → summarize_agent (port 8001) found
     → “Summarization also exists. Write a wrapper that calls both components.”

  4. eacp_find_available_port({ "preferred_range": { "start": 8000, "end": 8999 } })
     → port 8005 allocated

  5. eacp_check_path_permission({ "path": "/home/user/docs", "mode": "read" })
     → { allowed: true }

  6. eacp_get_policies()
     → localhost_only: true, secrets_mgmt: env_file, async: always

→ Generated code:
   port=8005, bind to 127.0.0.1, uv add, asyncio,
   read /home/user/docs, call rag_component(8002) → summarize_agent(8001)
```

---

## 4. Data Schema and Versioning

### 4.1 Schema Definition

Each EACP response is typed with **JSON Schema** to ensure machine readability and backward compatibility.

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

### 4.2 Required vs. Optional Fields

| Category | Required? | Reason |
|----------|-----------|--------|
| L0 System & Hardware | **Required** | Foundation for code assumptions |
| L1 Runtime | **Required** | Essential for dependency resolution and syntax selection |
| L2 Network | **Required** | Core for port conflict avoidance |
| L3 Ecosystem | **Required** | Core for preventing reinvention (Capability Graph) |
| L4 Filesystem | **Recommended** | For safe file operations |
| L5 Policies | **Recommended** | For automatic compliance with design rules |
| L7 Intent | **Optional** | Dramatically improves accuracy when present |

### 4.3 Extension Fields

Framework‑specific information is placed inside the `extensions` object, preserving the core protocol’s generality.

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

## 5. Security and Privacy

### 5.1 Return Minimal Information by Default

- **Standard output**: L0 + parts of L1 + L2 (port list) + L3 (component names, capabilities, endpoints only).
- **Handling of secrets**:
  - API keys, passwords, tokens **are never returned**. The server may return only the list of key names (`available_secret_keys`).
  - It **does not read** the content of `.env` files.
  - Access to paths listed in `forbidden_paths` can be blocked at the MCP server level; the AI may not even see their existence.
- **Disclosure of sensitive information (L4/L5)**: Detailed information is returned only when the EACP server is started with `--expose-sensitive` or explicitly allowed via configuration.

### 5.2 TTL and Caching Strategy (Recommendation for MCP Server Implementations)

| Category | TTL | Reason |
|----------|-----|--------|
| L0 System | Cache for session | Almost immutable |
| L1 Runtime | 1 hour | Packages may be added or updated |
| L2 Network | **10 seconds** | Port usage changes rapidly |
| L3 Ecosystem | 30 seconds | Components start / stop frequently |
| L4 Filesystem | 5 minutes | Does not change often |
| L5 Policies | Cache for session | User settings rarely change during operation |

---

## 6. Framework‑Agnosticism Policy

EACP is not limited to any specific framework (not tied to LAAS/AAF). It can be used with any local AI application infrastructure.

- The `policies.framework.name` field can declare `"laas"`, `"langchain"`, `"crewai"`, `"comfyui"`, `"custom"`, etc.
- Each framework may define its own schema inside `extensions.<framework_name>`.
- LAAS/AAF will provide a **reference implementation** of EACP, but the protocol itself is open.

---

## 7. Implementation Roadmap

### Phase 1: EACP PoC (2–4 weeks)
**Goal**: Protocol declaration + minimal working MCP server.

- GitHub repository established
- `eacp_get_system_profile` (L0)
- `eacp_find_available_port` (L2 core)
- `eacp_query_ecosystem` (L3 core, with capability filter)
- `eacp_get_full_snapshot` (batch retrieval)

Even this minimal set can dramatically reduce “port conflicts” and “reinvention” by AI agents.

### Phase 2: EACP v0.5 (2–3 months)
- `eacp_get_filesystem` (L4) and path permission checks
- `eacp_get_policies` (L5) design rule support
- Stricter JSON Schema and public validator
- LAAS/AAF extension fields via `extensions`

### Phase 3: EACP v1.0 RFC (6+ months)
- Standardise `eacp_get_intent` (L7): a wizard that lets the EACP server interview the human to capture intent
- Bidirectional features: `eacp_propose_allocation` (AI asks to reserve a port → EACP locks and responds)
- Cross‑review and standardisation involving multiple LLM vendors and frameworks

---

## 8. Conclusion: Why “Protocol”?

Traditional “context packages” (ZIP snapshots) suffer from **immediate staleness and waste of the AI’s context window**.

By redefining the idea as a **dynamic MCP‑based protocol**, EACP achieves:

1. **Real‑time awareness** – Port and process states change constantly. Query them on demand, do not rely on static files.
2. **Just‑enough information** – The AI retrieves only what it needs, saving tokens.
3. **Reinvention prevention** via Capability Graph – The AI searches for already‑running components by capability and reuses existing assets.
4. **Safe execution** via Policies & Constraints – The AI learns “what not to do” from the system and automatically follows user policies.
5. **Framework agnosticism** – EACP can become a standard for any local AI development environment, not only LAAS/AAF.

**Next actions**:
1. Publish this Draft Specification as Markdown on GitHub.
2. Implement a minimal Python MCP server (FastAPI + `mcp` SDK) that provides `eacp_find_available_port` and `eacp_query_ecosystem` as a prototype.
3. Connect it to Claude Code / Cursor etc., and publish a demo showing “local environment‑optimised code generation using EACP”.

This is the value of a **neutral, open “reality‑constraint model”** that stands between AI coding agents and the local environment.

---

*EACP Draft Specification v0.1*  
*Contributors: LAAS/AAF Project*

---

上記の英語版を `specification.en.md` として保存・コミットしてください。原文と同様に `docs/` ディレクトリに配置することをお勧めします。また、`docs/README.md` で言語選択リンクを提供すると親切です。
