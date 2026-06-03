# Environment-Aware Context Protocol (EACP) – Specification

This directory contains the core specification of the **Environment-Aware Context Protocol (EACP)**.

EACP is a **framework-agnostic open protocol** that enables AI coding agents to understand a user’s local execution environment and generate code that is optimally adapted to that environment.

---

## 📖 Specification by Language

| Language | File |
|----------|------|
| English | [specification.en.md](specification.en.md) |
| 日本語 | [specification.ja.md](specification.ja.md) |
| 中文 (简体) | [specification.zh.md](specification.zh.md) |

---

## 🎯 What is EACP?

Unlike static environment snapshots, EACP is a **dynamic query‑based protocol** implemented as an MCP (Model Context Protocol) server. AI agents retrieve only the information they need, when they need it – balancing real‑time awareness with context window economy.

Key principles:

- **Decision‑Oriented** – Provide information that helps the AI make decisions, not raw data.
- **Lazy / On‑Demand** – Call only the tools you need.
- **Privacy‑by‑Default** – Sensitive values (passwords, API keys) are hidden by default.
- **Framework‑Agnostic** – Not tied to LAAS, LangChain, or any specific framework.
- **Human‑Intent Aware** – Embed the user’s goal and constraints into the context.

---

## 🛠️ Example Tools (provided via MCP)

- `eacp_find_available_port` – Find and reserve an unused port.
- `eacp_query_ecosystem` – Search for existing components by capability.

These tools let AI agents avoid port conflicts and reuse existing assets instead of reinventing them.

---

## 🤝 Related Projects

- **[Local AI App Stack (LAAS) / AI App Forge (AAF)](https://github.com/Masaru-KMT/Local-AI-App-Stack)** – Reference implementation and application platform that realize the EACP concept.
- **[Model Context Protocol (MCP)](https://modelcontextprotocol.io)** – The underlying protocol EACP builds upon.

---

## 📄 License

This specification is released under the [MIT License](https://github.com/Masaru-KMT/Environment-Aware-Context-Protocol/blob/main/LICENSE).

---

*EACP – A neutral, open reality‑constraint model between AI coding agents and the local environment.*
