# EACP (Environment-Aware Context Protocol) Draft Specification v0.1

**Version**: 0.1.0 (Draft)  
**Last Updated**: 2026-06-04  
**Status**: Draft / Review

---

## 1. 概要と目的

**Environment-Aware Context Protocol (EACP)** は、AI コーディングエージェントがユーザー固有のローカル実行環境を「理解」し、その環境に最適化されたコードを生成できるようにするための、**フレームワーク非依存のオープンプロトコル**です。

EACP は従来の静的な「環境スナップショット」ではなく、**MCP (Model Context Protocol) サーバーとして実装される動的な問い合わせ型プロトコル**です。これにより、AI エージェントは必要な情報だけを必要な時に取得し、コンテキストウィンドウの節約とリアルタイム性を両立します。

### 1.1 解決する課題

- **環境前提の誤り**: 「Python 3.12 で書いたが実環境は 3.10 だった」
- **衝突の発生**: 「ポート 8080 で起動するコードを書いたが既に使われていた」
- **再発明**: 「要約機能を新規実装したが、同じものがローカルに既にあった」
- **権限・制約違反**: 「ファイルを `/etc/` に書こうとしたが権限がなかった」

### 1.2 設計原則

| 原則 | 説明 |
|------|------|
| **Decision-Oriented** | 「情報そのもの」ではなく「AI の判断材料」を提供する |
| **Lazy / On-Demand** | 全情報を一括で押し付けるのではなく、必要なツールだけを呼び出す |
| **Privacy-by-Default** | センシティブ情報（パスワード、API キー）は**値を隠し、メタデータのみ**を返す |
| **Framework-Agnostic** | 特定のフレームワーク（LAAS/AAF/LangChain 等）に依存しない。`project_context` で任意のフレームワーク名を宣言可能 |
| **Composable** | 部分取得可能。AI エージェントが必要なレイヤだけを問い合わせる |
| **Human-Intent Aware** | 環境情報だけでなく「人間が何を作りたいか」の目的・優先度も文脈に含める |

---

## 2. 情報モデル（7 つのカテゴリ）

EACP が提供する情報は、以下の 7 カテゴリに分類される。AI エージェントは必要に応じて個別に問い合わせる。

```text
EACP
├── 1. System & Hardware        ... OS、アーキテクチャ、CPU/GPU/メモリ
├── 2. Runtime & Toolchain      ... 言語バージョン、パッケージマネージャ、インストール済みライブラリ
├── 3. Network & Ports          ... IP アドレス、ポート使用状況、プロキシ、ファイアウォール
├── 4. Ecosystem & Assets       ... 稼働中コンポーネント・Capability Graph（核心）
├── 5. Filesystem & Permissions ... 読み書き可能パス、禁止パス、Gitignore
├── 6. Policies & Constraints   ... 設計規約、セキュリティポリシー、推奨/禁止パターン
└── 7. Intent & Context         ... 人間の目的・優先度・避けるべき事項（オプションだが重要）
```

### 2.1 System & Hardware（L0）

AI が「この環境に何が実現可能か」を判断する基礎情報。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `os.type` | enum | `"linux"`, `"darwin"`, `"windows"` | パス区切り、システムコール、パッケージマネージャの使い分け |
| `os.version` | string | `"24.04"`, `"14.5"`, `"22H2"` | バージョン固有の機能可否 |
| `os.wsl` | bool | `true` | WSL2 はネイティブ Linux と挙動が異なる（パス変換、systemd 等） |
| `arch` | enum | `"x86_64"`, `"arm64"` | バイナリ/Wheel の選定 |
| `cpu.logical_cores` | int | `16` | 並列処理数の判断 |
| `ram.total_mb` | int | `32768` | ローカル LLM 可否、バッチサイズの判断 |
| `ram.available_mb` | int | `18432` | 現在使えるリソース |
| `gpu.available` | bool | `true` | CUDA/MPS/Metal コードを生成するか |
| `gpu.vendor` | string | `"nvidia"`, `"apple"` | バックエンド選定 |
| `gpu.vram_mb` | int | `12288` | 読み込めるモデルサイズの判断 |
| `gpu.cuda_version` | string | `"12.4"` | 依存バージョン固定 |
| `disk.free_gb` | int | `120` | 大容量モデル/DB のダウンロード可否 |

### 2.2 Runtime & Toolchain（L1）

「コードをどう書き、どう実行するか」の判断材料。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `runtime.python.version` | string | `"3.12.3"` | 構文（match 文、型ヒント）の可否 |
| `runtime.python.versions_available` | [string] | `["3.10", "3.12"]` | 複数バージョンがある場合の選択肢 |
| `runtime.package_manager.type` | enum | `"uv"`, `"poetry"`, `"pip"`, `"conda"` | インストールコマンドの統一 |
| `runtime.package_manager.version` | string | `"0.5.1"` | 互換性チェック |
| `runtime.shell.type` | enum | `"bash"`, `"zsh"`, `"powershell"` | 環境変数設定構文の選定 |
| `runtime.venv.path` | string | `"/home/user/project/.venv"` | 仮想環境の明示 |
| `runtime.binaries.available` | [string] | `["git", "docker", "sqlite3"]` | Docker や SQLite を前提にできるか |

### 2.3 Network & Ports（L2）

**ポート競合を回避し、適切なバインドを行う**ための最重要情報。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `network.lan_ip` | string | `"192.168.1.50"` | LAN 内アクセス URL の生成 |
| `network.loopback` | string | `"127.0.0.1"` | デフォルトバインド先 |
| `network.used_ports` | [int] | `[4000, 8001, 10423]` | ブラックリスト。絶対に割り当てない |
| `network.available_port_ranges` | [obj] | `[{"start":8000,"end":8999}]` | システムが推奨するレンジ |
| `network.suggested_next_port` | int | `8002` | 即座に使える代表値 |
| `network.localhost_only_default` | bool | `true` | セキュリティポリシーが `127.0.0.1` 固定を求めているか |
| `network.proxy.http` | string | `"http://proxy:8080"` | 外部 API 呼び出しコードに自動注入 |
| `network.firewall.enforced` | bool | `true` | ファイアウォールが有効か |

### 2.4 Ecosystem & Assets（L3）—— Capability Graph

**EACP の最も核心となるカテゴリ。** AI が「作るか、既存を使うか」を判断するための Capability Graph。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `ecosystem.components` | [obj] | 下記参照 | 再利用可能なコンポーネント群 |
| `ecosystem.mcp_tools` | [obj] | 下記参照 | 外部 MCP サーバー群 |
| `ecosystem.llm_gateways` | [obj] | 下記参照 | LiteLLM 等の統合エンドポイント |

#### Component オブジェクト

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

#### MCP Tool オブジェクト

```json
{
  "name": "filesystem",
  "transport": "stdio",
  "capabilities": ["read_file", "write_file", "list_directory"]
}
```

#### LLM Gateway オブジェクト

```json
{
  "name": "litellm_proxy",
  "endpoint": "http://127.0.0.1:4000",
  "configured_models": ["gpt-4o", "ollama/llama3"]
}
```

### 2.5 Filesystem & Permissions（L4）

AI が「どこにファイルを作ってよいか」を事前に把握する。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `fs.project_root` | path | `"/home/user/projects/myapp"` | 相対パスの基準 |
| `fs.cwd` | path | `"/home/user/projects/myapp"` | 現在の作業ディレクトリ |
| `fs.writable_paths` | [path] | `["~/projects", "~/.local/share"]` | 書き込み許可ホワイトリスト |
| `fs.readable_paths` | [path] | `["~/documents", "/etc/ssl"]` | 読み取り許可パス |
| `fs.forbidden_paths` | [path] | `["~/.ssh", "/etc/passwd", "/System"]` | **絶対にアクセス禁止** |
| `fs.sticky_dirs` | [path] | `["~/.local/share/aaf", "~/.cache"]` | 永続化の推奨場所 |
| `fs.env_files` | [string] | `[".env", ".env.local"]` | 機密情報の保存先候補 |
| `fs.gitignored` | [string] | `[".env", "*.db", "__pycache__/"]` | 誤コミット防止 |

### 2.6 Policies & Constraints（L5）

AI が「どう書いてはいけないか」を知るための設計規約。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `policies.security.bind_host_default` | enum | `"127.0.0.1"` | デフォルトバインド制限 |
| `policies.security.secrets_mgmt` | enum | `"env_file"` | キーの扱い方（env_file / keyring / none）|
| `policies.security.network_egress` | enum | `"allow_all"`, `"offline"`, `"whitelist"` | 外部通信方針 |
| `policies.code.style` | enum | `"black"`, `"ruff"`, `"pep8"` | フォーマット統一 |
| `policies.code.async_preference` | enum | `"always"`, `"only_if_needed"`, `"never"` | 非同期方針 |
| `policies.code.prohibited_patterns` | [string] | `["dynamic_port_assignment", "bind_0.0.0.0"]` | **禁止パターン** |
| `policies.code.preferred_patterns` | [string] | `["fixed_port_via_env", "trace_id_propagation"]` | **推奨パターン** |
| `policies.testing.required` | bool | `true` | テストコード生成の要否 |
| `policies.framework.name` | string | `"laas"`, `"langchain"`, `"custom"` | フレームワーク非依存を保ちつつ、規約を適用 |
| `policies.framework.version` | string | `"1.1.0"` | フレームワークバージョン |

### 2.7 Intent & Context（オプションだが重要）

**人間の目的を環境文脈に組み込む**（Masaru/ChatGPT5  Proposal を統合）。

| 項目 | 型 | 例 | 根拠 |
|------|-----|-----|------|
| `intent.goal` | string | `"ローカル文書の要約と検索統合"` | 生成コードの方向性 |
| `intent.priority` | enum | `"low_latency"`, `"accuracy"`, `"low_cost"` | トレードオフ判断 |
| `intent.avoid` | [string] | `["duplicate_implementation", "external_api_dependency"]` | 避けるべき実装 |
| `intent.preferred_tech` | [string] | `["sqlite", "fastapi"]` | ユーザーが好む技術選択 |
| `intent.notes` | string | `"16GB未満のメモリでは重いモデルを使わないで"` | 補足的自由記述 |

---

## 3. MCP サーバーとしてのツール定義

EACP は MCP の `Tool` として以下を提供する。AI エージェントは必要に応じてこれらを呼び出す。

### 3.1 カテゴリ別取得ツール

| ツール名 | 説明 | 入力 | 出力 |
|---------|------|------|------|
| `eacp_get_system_profile` | L0（システム・ハードウェア）取得 | `{}` | System & Hardware |
| `eacp_get_runtime` | L1（ランタイム）取得 | `{}` | Runtime & Toolchain |
| `eacp_get_network` | L2（ネットワーク・ポート）取得 | `{}` | Network & Ports |
| `eacp_query_ecosystem` | L3（Ecosystem）取得。capability フィルタ対応 | `{ "capability": "summarization" }` | マッチしたコンポーネント一覧 |
| `eacp_get_filesystem` | L4（ファイルシステム）取得 | `{}` | Filesystem & Permissions |
| `eacp_get_policies` | L5（ポリシー）取得 | `{}` | Policies & Constraints |
| `eacp_get_intent` | L7（人間の意図）取得 | `{}` | Intent & Context |

### 3.2 ユーティリティ・判断支援ツール

| ツール名 | 説明 | 入力 | 出力 |
|---------|------|------|------|
| `eacp_find_available_port` | 指定レンジ内の空きポートを **1つ確保して返す** | `{ "preferred_range": { "start": 8000, "end": 8999 } }` | `{ "port": 8003 }` |
| `eacp_check_capability` | 指定 capability を持つコンポーネントが存在するか確認 | `{ "capability": "summarization" }` | `{ "found": true, "component": {...} }` |
| `eacp_check_path_permission` | 特定パスへの read/write 可否を問い合わせ | `{ "path": "/home/user/data", "mode": "write" }` | `{ "allowed": true }` |
| `eacp_get_full_snapshot` | L0〜L6 までを **一括取得**（静的 Context Package 生成時に使用） | `{ "include_sensitive": false }` | 全カテゴリマージ JSON |

### 3.3 ツール呼び出しフロー例

```
ユーザー: 「ローカル文書を検索して要約する機能を追加して」

AI:
  1. eacp_get_system_profile()
     → Mac(arm64), RAM 16GB, GPU(MPS) あり
     → 「CUDA コードは書かず、MPS または CPU 実装にする」

  2. eacp_query_ecosystem({ "capability": "semantic_search" })
     → rag_component (port 8002) が発見
     → 「検索機能は既にある。新規実装せず呼び出す」

  3. eacp_check_capability({ "capability": "summarization" })
     → summarize_agent (port 8001) が発見
     → 「要約機能も既にある。両方を呼ぶラッパーを書く」

  4. eacp_find_available_port({ "preferred_range": { "start": 8000, "end": 8999 } })
     → port 8005 を確保

  5. eacp_check_path_permission({ "path": "/home/user/docs", "mode": "read" })
     → { allowed: true }

  6. eacp_get_policies()
     → localhost_only: true, secrets_mgmt: env_file, async: always

→ 生成コード:
   port=8005, 127.0.0.1 固定, uv add 方式, asyncio,
   /home/user/docs を読み込み rag_component(8002) → summarize_agent(8001) へリクエスト
```

---

## 4. データスキーマとバージョニング

### 4.1 スキーマ定義

EACP の各レスポンスは **JSON Schema** で型定義される。機械可読性と後方互換性を担保する。

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

### 4.2 必須フィールドと任意フィールド

| カテゴリ | 必須度 | 理由 |
|---------|--------|------|
| L0 System & Hardware | **必須** | コードの前提を決定する基盤 |
| L1 Runtime | **必須** | 依存解決と構文選定に不可欠 |
| L2 Network | **必須** | ポート競合回避の核心 |
| L3 Ecosystem | **必須** | 再発明防止の核心（Capability Graph）|
| L4 Filesystem | **推奨** | 安全なファイル操作のため |
| L5 Policies | **推奨** | 設計規約の自動遵守のため |
| L7 Intent | **オプション** | あれば精度が劇的に向上 |

### 4.3 拡張フィールド

フレームワーク固有の情報は `extensions` オブジェクト内に配置し、EACP コアの汎用性を損なわない。

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

## 5. セキュリティとプライバシー

### 5.1 デフォルトで最小限の情報を返す

- **標準出力**: L0 + L1（一部）+ L2（ポート一覧）+ L3（コンポーネント名・capabilities・エンドポイントのみ）
- **秘匿情報の取り扱い**:
  - API キー、パスワード、トークンは **一切値を返さない**。キー名のリスト（`available_secret_keys`）のみ返してもよい。
  - `.env` ファイルの内容は読まない。
  - `forbidden_paths` に該当するパスへのアクセスは、MCP サーバー側でブロックし、AI にその存在すら見せないことも可能。
- **センシティブ情報（L4/L5）の開示**: EACP サーバー起動時に `--expose-sensitive` や設定ファイルで明示的に許可した場合のみ詳細を返す。

### 5.2 TTL とキャッシュ戦略（MCP サーバー実装上の推奨）

| カテゴリ | TTL | 理由 |
|---------|-----|------|
| L0 System | セッション中キャッシュ | ほぼ不変 |
| L1 Runtime | 1時間 | パッケージの追加・更新がありうる |
| L2 Network | **10秒** | ポートは瞬時に変化する |
| L3 Ecosystem | 30秒 | コンポーネントの起動/停止が頻繁 |
| L4 Filesystem | 5分 | 頻繁には変わらない |
| L5 Policies | セッション中キャッシュ | ユーザー設定は運用中変更されにくい |

---

## 6. フレームワーク非依存性の方針

EACP は特定のフレームワーク（LAAS/AAF に限定されない）。任意のローカル AI アプリケーション基盤で利用可能。

- `policies.framework.name` フィールドで `"laas"`, `"langchain"`, `"crewai"`, `"comfyui"`, `"custom"` 等を宣言可能。
- 各フレームワークは `extensions.<framework_name>` 内に固有スキーマを定義できる。
- LAAS/AAF は ECP を **参照実装（Reference Implementation）** として提供するが、プロトコル自体はオープン。

---

## 7. 実装ロードマップ

### Phase 1: EACP PoC（2〜4週間）
**目標**: 「プロトコルとしての宣言」と、最小限の動作する MCP サーバー

- GitHub リポジトリ設立
- `eacp_get_system_profile`（L0）
- `eacp_find_available_port`（L2 核心）
- `eacp_query_ecosystem`（L3 核心、capability フィルタ付き）
- `eacp_get_full_snapshot`（バッチ取得）

これだけでも、AI エージェントの「ポート競合」「再発明」を劇的に減らせる。

### Phase 2: EACP v0.5（2〜3ヶ月）
- `eacp_get_filesystem`（L4）とパス権限チェック
- `eacp_get_policies`（L5）の設計規約サポート
- JSON Schema の厳密化とバリデータ公開
- `extensions` による LAAS/AAF 拡張フィールドの整備

### Phase 3: EACP v1.0 RFC（6ヶ月〜）
- `eacp_get_intent`（L7）の標準化：人間の目的を MCP サーバーがヒアリングするウィザード機能
- 双方向機能：`eacp_propose_allocation`（AI が「ポート予約したい」と要求 → EACP がロックして応答）
- 複数 LLM ベンダー・フレームワークによるクロスレビューと標準化

---

## 8. 結論：なぜ「プロトコル」として定義するのか

従来の「Context Package（ZIP）」は、**スナップショットが瞬時に陳腐化し、AI のコンテキストウィンドウを圧迫する**限界がありました。

EACP はこれを **MCP 上の動的プロトコル**として再定義することで、以下を実現します。

1. **リアルタイム性**: ポートやプロセス状態は常に変化する。静的ファイルではなく都度問い合わせる。
2. **必要十分な情報取得**: AI が「今必要なものだけ」を取得し、トークンを節約する。
3. **再発明防止（Capability Graph）**: 「何が既に動いているか」を capabilities 単位で検索し、AI が既存資産を再利用する。
4. **安全な実行（Policies & Constraints）**: AI が「やってはいけないこと」をシステムから学び、ユーザーのポリシーを自動遵守する。
5. **フレームワーク非依存**: LAAS/AAF に留まらず、あらゆるローカル AI 開発環境に適用可能な標準となる。

**次のアクション**:
1. 本 Draft Specification を Markdown として GitHub に公開する。
2. `eacp_find_available_port` と `eacp_query_ecosystem` を持つ最小限の Python MCP サーバー（FastAPI + `mcp` SDK）をプロトタイプとして実装する。
3. Claude Code / Cursor 等に接続し、「EACP を使ったローカル最適化コード生成」のデモを公開する。

これが、AI コーディングエージェントとローカル環境の間に立つ、**中立的でオープンな「現実制約モデル」**としての価値です。

---

*EACP Draft Specification v0.1*  
*Contributors: LAAS/AAF Project
