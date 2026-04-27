# Agent Host Development Spec v1

## Purpose

本文件定義實驗室主機端 agent runtime 的第一版開發規格。此系統是整體專題的主控端，負責接收邊緣節點事件、管理節點狀態、提供可組合的工具介面，並為未來的 ESP32-S3 harness tool node 預留接入能力。[cite:1][cite:17]

本版本聚焦在 **沒有實體 ESP32-S3 模組時** 仍可開始開發的主機端核心架構。第一版以 Rust 為主要實作語言，先完成 event-driven host runtime、資料管線、狀態儲存與 simulator 對接，不以 BusyBox applet 或 AI sandbox 完整功能為本階段必交內容。[cite:1][cite:63]

## Project Position

整體系統由兩個主要倉庫構成：

1. **主系統 Repo（Rust）**：實作 agent host runtime、通訊協定、事件資料管線、狀態儲存與節點模擬器。[cite:1][cite:17]
2. **作業 Repo（BusyBox/C）**：依課程選項 B、方向三，實作可整合至 BusyBox 的資料管線工具集，並視需要以 Git submodule 方式引用主系統 Repo 的文件、測試資料、協定樣本或開發腳本。[cite:63]

本文件只規範 **主系統 Repo** 的開發內容，不涵蓋 BusyBox applet 的 C 實作細節。[cite:63]

## System Goal

主系統的目標是建立一個可被未來邊緣節點接入的 agent host。此 host 應具備以下能力：[cite:1]

- 接收節點透過 WebSocket 傳入的結構化事件。
- 維護節點狀態、能力與最後活動時間。
- 以資料管線方式處理事件流，支援解析、過濾、轉換與持久化。
- 對外提供 agent 可調用的 API。
- 在沒有真實硬體時，能透過 simulator 驗證整個流程。[cite:1][cite:17]

## Non-Goals

本版本不包含以下內容：[cite:1][cite:63]

- 真實 ESP32-S3 韌體與硬體驅動。
- 完整 BusyBox applet 建置與單一執行檔整合。
- 完整 AI sandbox 隔離執行環境。
- 長時間視訊串流、音訊串流或高負載多媒體傳輸。
- 進階 agent 自動生成並執行程式碼的閉環流程。

## Architecture

### Core Components

| Module | Responsibility |
|---|---|
| `domain` | 定義共用資料模型，例如 `NodeState`、`EventType`、`ToolCall`、`ToolResult`。[cite:1] |
| `protocol` | 定義 WebSocket JSON schema、訊息版本與序列化格式。[cite:1] |
| `agent-host` | 提供 port `10002` 上的 HTTP/WebSocket 入口，管理 session、節點與命令下發。 |
| `pipeline` | 對事件資料做 parse、filter、transform、export。這對應到方向三的 embedded data pipeline 核心精神。[cite:63] |
| `store` | 提供 file-based 或 SQLite 狀態儲存，保存最後狀態、事件索引、TTL 資料。[cite:63] |
| `simulator` | 模擬一台或多台 ESP32-like node，驗證 host runtime 行為。[cite:1] |
| `cli` | 提供主系統自己的 Rust CLI，方便測試與除錯。 |

### Deployment Shape

第一版部署於實驗室 Linux 主機上，由單一 Rust service 啟動。外部可經由 `ws://<host>:10002/ws` 或 `http://<host>:10002` 與主機互動；未來若加上 TLS 或反向代理，不改變應用層訊息格式。[cite:17]

## Communication Model

主機與節點之間的第一版通訊採用 **WebSocket + JSON**。這符合原始規格中的 Wi‑Fi HTTP endpoint 或 WebSocket 方向，且訊息以 `event` / `toolcall` 為中心。[cite:1]

### Connection Lifecycle

1. Node 主動連上 host WebSocket。[cite:1]
2. Node 送出 `hello` 訊息，包含 `node_id`、`firmware_version`、`capabilities`。
3. Host 回覆 `hello_ack`。
4. Node 定期送出 `heartbeat`。
5. Node 有事件時送 `event`。
6. Host 需操作節點時送 `toolcall`。
7. Node 執行後回 `tool_result` 或 `error`。[cite:1]

### Message Types

| Type | Direction | Purpose |
|---|---|---|
| `hello` | node -> host | 節點註冊與能力宣告 |
| `hello_ack` | host -> node | 確認註冊成功 |
| `heartbeat` | node -> host | 維持 session 與可用性檢查 |
| `event` | node -> host | 回報感測或狀態事件。[cite:1] |
| `toolcall` | host -> node | 要求節點執行工具，例如 `displaystatus`、`capturesnapshot`。[cite:1] |
| `tool_result` | node -> host | 回報工具執行結果。[cite:1] |
| `error` | both | 回報協定錯誤、狀態錯誤或執行失敗 |

### Initial Event Scope

第一版至少支援以下事件與工具：[cite:1]

- Events: `PRESENCEON`, `PRESENCEOFF`, `AUDIOACTIVITY`, `NODEERROR`
- Tools: `getpresence`, `displaystatus`, `getnodestate`
- Optional stub: `capturesnapshot`, `speak`

## Data Model

### Required Node State

節點狀態至少包含以下欄位：[cite:1]

- `node_id`
- `connection_state`
- `runtime_state`，例如 `BOOTING`, `IDLE`, `PRESENCEDETECTED`, `TOOLEXECUTING`, `COOLDOWN`, `ERROR`
- `last_heartbeat_at`
- `last_event`
- `capabilities`
- `metadata`

### Event Record

每筆事件至少包含：

- `id`
- `node_id`
- `event_type`
- `timestamp`
- `payload`（JSON object）
- `ingested_at`
- `source`（simulator / real-node）

### Tool Call Record

每筆命令至少包含：

- `request_id`
- `node_id`
- `tool`
- `args`
- `issued_at`
- `status`
- `result_payload`

## Storage

第一版以簡單、可測試為優先，可採以下兩種方案之一：[cite:63]

- **方案 A：JSONL + file-based state**：事件寫入 `.jsonl`，狀態寫入單獨 JSON 檔。
- **方案 B：SQLite**：事件、節點、工具執行紀錄落地到單一資料庫。

建議優先採用 SQLite，原因如下：

- 容易查詢最後狀態與時間範圍事件。
- 容易支撐後續 dashboard 或 BusyBox 作業工具的資料來源。
- 部署與測試成本低。[cite:63]

## Pipeline Requirements

依照選項 B、方向三，本系統的資料處理應具備 pipe-friendly 與結構化資料導向的特性。[cite:63]

第一版 pipeline 需具備：

- 讀取 JSON event stream。
- 依 `node_id`、`event_type`、時間範圍過濾。
- 輸出 JSON Lines。
- 可轉出 CSV 作為後續報告與 benchmark 材料。
- 可將節點最新狀態寫入 key-value 風格儲存。[cite:63]

## Simulator Requirements

在無 ESP32-S3 實體模組時，simulator 是本階段必要元件。[cite:17]

Simulator 應支援：

- 模擬單一節點啟動與註冊。
- 週期性 heartbeat。
- 手動或腳本觸發 `PRESENCEON`、`AUDIOACTIVITY`、`NODEERROR`。
- 接收 host 發出的 `displaystatus` 與 `getnodestate`。
- 模擬狀態轉換，例如 `IDLE -> PRESENCEDETECTED -> TOOLEXECUTING -> COOLDOWN -> IDLE`。[cite:1]

## API Requirements

### HTTP Endpoints

第一版至少提供：

- `GET /health`：服務健康檢查
- `GET /nodes`：列出所有節點
- `GET /nodes/:id`：查單一節點狀態
- `GET /events`：查詢事件
- `POST /nodes/:id/toolcalls`：送命令給節點

### WebSocket Endpoint

- `GET /ws`：供 node 連線與交換事件/命令。

## Rust Technical Choices

建議技術選型如下：

- `tokio`：非同步 runtime
- `axum`：HTTP / WebSocket server
- `serde`, `serde_json`：資料序列化
- `sqlx` 或 `rusqlite`：SQLite 存取
- `tracing`：結構化日誌
- `clap`：CLI 參數解析
- `thiserror` / `anyhow`：錯誤處理

此組合可支撐 Linux 主機上的 event-driven service，並保持未來擴充性。[cite:17]

## Repository Structure

```text
agent-host-rs/
├── Cargo.toml
├── crates/
│   ├── domain/
│   ├── protocol/
│   ├── agent-host/
│   ├── pipeline/
│   ├── store/
│   ├── simulator/
│   └── cli/
├── config/
│   └── dev.toml
├── scripts/
├── fixtures/
│   └── sample-events/
├── docs/
│   └── architecture.md
└── README.md
```

## Milestone Plan

### M0: Workspace Bootstrap

- 建立 Rust workspace。
- 建立 `domain` 與 `protocol` crate。
- 定義基礎 message schema。

### M1: Host Connectivity

- 建立 `agent-host`。
- 開放 port `10002`。
- 完成 `hello` / `heartbeat` / `event` 收取。

### M2: Event Persistence

- 實作事件寫入。
- 提供 `GET /nodes` 與 `GET /events`。
- 建立基本 CLI 查詢能力。

### M3: Tool Call Loop

- Host 可對 simulator 發 `displaystatus`。
- Simulator 回 `tool_result`。
- 完成最小 command loop。[cite:1]

### M4: Pipeline Prototype

- 完成 parse / filter / export。
- 輸出 JSONL 與 CSV。
- 準備後續對應 BusyBox 方向三的工具功能。[cite:63]

## Acceptance Criteria

第一版完成的標準如下：

- Host 能在 Linux 主機上啟動於 `10002` port。
- Simulator 能連線、註冊並送 heartbeat。
- 至少一種事件可被成功接收與保存。
- Host 能查詢節點狀態。
- Host 能下發至少一個 tool call 並收到結果。
- Pipeline 能對事件資料做過濾與輸出。
- 程式碼具備基本測試、錯誤處理與文件。[cite:1][cite:63]

## Future Integration

後續整合方向如下：

- 接入真實 ESP32-S3 節點。[cite:1]
- 將 Rust 主系統 repo 以 submodule 方式被作業 repo 引用。[cite:17]
- 將 pipeline 中穩定的 3 個工具映射為 BusyBox/C applet，以符合選項 B 交付要求。[cite:63]
- 將 agent 任務執行層擴充為受控 sandbox，但不作為 v1 核心需求。[cite:63]
