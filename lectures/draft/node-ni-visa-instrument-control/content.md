# Node 整合 NI-VISA 儀器控制：自動化測試平台架構
> 從 NI-VISA、SCPI、Binary 通訊到 Local Runner，規劃一套可派發、可版本化、可回收結果的 Client/Server 自動化測試平台。

[summary]
- 架構 **Client/Server 分工** | Server 管理測試知識與版本，Client 負責靠近硬體的實際執行
- 核心 **命令與儀器抽象** | 用 Driver Wrapper 將 SCPI、Binary 與廠商命令集收斂成一致 API
- 執行 **Local Runner** | 在使用者電腦處理 VISA Runtime、儀器連線、流程執行、報告產生與結果上傳
- 治理 **版本與追溯** | 儀器型號、命令集、測試流程、測試包與報告都需要可查詢的版本紀錄
[/summary]

## 平台設計的核心問題

### 為什麼不能只做雲端控制
- NI-VISA 與 NI-488.2 通常安裝在靠近儀器的使用者電腦或測試站
- GPIB、USB、Serial、LAN 儀器會受到實體連線、區網、防火牆與驅動程式限制
- 測試流程可以由 Server 管理，但真正的儀器 I/O 必須由本機 Runner 執行
- Server 的價值不是直接碰硬體，而是管理「怎麼測、用哪個版本測、測完如何追溯」

> **平台邊界要先畫清楚**
> 儀器控制系統最容易失控的地方，是把命令、流程、驅動、報告全部寫死在同一支程式。
>
> 正確的做法是讓 Server 管理測試知識，讓 Local Runner 管理硬體執行，兩者透過明確 API 與測試包版本協作。

### 整體角色分工

[flow]
1. Server — 管理儀器型號、命令集、測試流程、腳本模板、版本與報告查詢
2. Test Package — 將測試流程、命令設定、限制條件與報告格式打包成可下載版本
3. Local Runner — 下載測試包、檢查版本、連接儀器、執行流程並產生報告
4. Driver Wrapper — 將 NI-VISA、SCPI、Binary 與廠商命令轉成一致的呼叫介面
5. Result API — 將測試結果、儀器資訊、執行紀錄與本機備份狀態回傳 Server
[/flow]

### 平台分層總覽

| Layer | 責任 | 典型內容 |
|---|---|---|
| Server Portal | 管理測試資產 | 儀器型號、命令集、測試步驟、流程編輯器、版本管理 |
| Package Service | 發布可執行內容 | 測試包、manifest、相依版本、腳本模板 |
| Local Runner | 本機執行核心 | 下載、驗證、連線、執行、報告、上傳、本機備份 |
| Driver Wrapper | 儀器抽象層 | VISA Session、SCPI 命令、Binary frame、response parser |
| Instrument | 實體設備 | DC/AC Source、DC/AC Load、示波器、LCR Meter |

# Server 端：把測試知識產品化
> Server 不是儀器控制程式，而是測試平台的大腦：它負責管理知識、版本、流程與結果。

## 儀器型號與命令集管理

### 儀器型號管理不是單純清單
- 型號資料要描述能力，而不是只記錄名稱與廠牌
- 同一類儀器可能支援不同介面，例如 GPIB、USB、TCPIP 或 Serial
- 同一個測試動作在不同型號上可能有不同 SCPI 語法或 Binary frame
- 型號設定應包含通訊參數、能力邊界、命令集版本與回應解析規則

### 命令集轉換層的目的

[flow]
1. Standard Action — 定義平台通用動作，例如 `setVoltage`、`measureCurrent`、`captureWaveform`
2. Instrument Profile — 依型號對應 SCPI、Binary 或廠商私有命令
3. Command Renderer — 將標準動作轉成實際送出的命令字串或二進位封包
4. Response Parser — 將儀器原始回應轉成數值、狀態、錯誤碼與單位
5. Validation Rule — 在送出前檢查量測範圍、參數型別與安全限制
[/flow]

```prompt [label="儀器命令設定草案"]
{
  "model": "DC-SOURCE-1000",
  "interfaces": ["USB", "TCPIP", "GPIB"],
  "actions": {
    "setVoltage": {
      "transport": "scpi",
      "command": "SOUR:VOLT {voltage}",
      "params": {
        "voltage": { "type": "number", "min": 0, "max": 60, "unit": "V" }
      }
    },
    "measureCurrent": {
      "transport": "scpi",
      "command": "MEAS:CURR?",
      "response": { "type": "number", "unit": "A" }
    }
  }
}
```

### Server 應保存的核心資料
- 儀器型號：品牌、型號、支援介面、通訊參數、能力範圍
- 命令集：SCPI 命令、Binary frame、參數規格、回應解析、錯誤碼對照
- 測試步驟：單一步驟的輸入、輸出、前置條件、通過條件
- 測試腳本模板：可重複套用的測試邏輯與報告格式
- 測試流程：流程編輯器輸出的步驟順序、分支、等待、判斷與失敗處理

## 測試流程與版本管理

### 三層測試描述
- Step Primitive：最小測試動作，例如設定電壓、讀取電流、擷取波形
- Procedure Template：可重複的測試片段，例如上電、穩定等待、讀值、判定
- Test Package：可派發給 Local Runner 的完整版本，包含流程、命令集與報告設定

### 測試包應該不可變
- 已發布的測試包不應被直接修改，否則報告無法追溯當時使用的測試邏輯
- 修改流程、命令或限制條件時，應建立新的 package version
- Runner 上傳結果時，要附上 package id、package version、runner version 與 driver wrapper version
- 報告查詢不能只看 pass/fail，還要能回查完整執行環境

> **版本管理不是工程潔癖**
> 自動化測試平台的結果會被拿來做品質判斷、出貨判斷或問題追查。
>
> 如果測試包沒有版本，日後就無法回答「這份報告到底是用哪一版流程、哪一版命令、哪一版 Driver Wrapper 跑出來的」。

### Server API 規劃

| API | 用途 | Local Runner 何時呼叫 |
|---|---|---|
| `GET /api/packages/:id` | 取得測試包 manifest | 開始測試前 |
| `GET /api/packages/:id/download` | 下載測試包內容 | 版本不一致或首次執行 |
| `POST /api/results` | 上傳測試結果 | 流程完成或中止後 |
| `POST /api/runner/heartbeat` | 回報 Runner 狀態 | 長時間測試期間 |
| `GET /api/reports/:id` | 查詢報告 | 使用者或工程師檢視結果 |

# Client 端：Local Runner 是硬體執行邊界
> Client 的任務不是管理所有知識，而是可靠地執行 Server 派發下來的測試包。

## 使用者電腦需要安裝什麼

### 基礎執行環境
- Node.js Runtime：執行 Local Runner 與測試流程控制邏輯
- Local Runner：平台派發或安裝的本機測試執行程式
- NI-VISA Runtime：提供 VISA Resource 掃描、Session 開啟與低階 I/O
- NI-488.2 Driver：支援 GPIB 介面與相關設備通訊
- Instrument Driver Wrapper：將儀器命令封裝成平台統一 API

### 安裝檢查不應只看檔案存在
- Node.js 版本要符合 Runner 支援範圍
- NI-VISA Runtime 要能列出 resource，而不是只檢查安裝目錄
- NI-488.2 要能辨識 GPIB 介面與設備位址
- Driver Wrapper 要回報自己的版本、支援型號與可用 transport
- 本機權限、防火牆與 USB/GPIB 權限要納入診斷報告

## Local Runner 生命週期

### 執行流程

[flow]
1. 啟動 Runner — 載入本機設定、識別 runner version 與 station id
2. 下載測試包 — 從 Server 取得 manifest，必要時下載新版 package
3. 檢查相依版本 — 比對 Node、NI-VISA、NI-488.2、Driver Wrapper 與 package 要求
4. 掃描儀器 — 列出 VISA resources，依測試包需求匹配指定儀器
5. 建立連線 — 開啟 VISA Session，設定 timeout、termination、buffer 與通訊參數
6. 執行流程 — 逐步送出 SCPI 或 Binary 命令，收集回應並做通過條件判斷
7. 產生報告 — 保存 raw log、標準化量測值、判定結果與錯誤資訊
8. 上傳結果 — 將報告送回 Server，失敗時保留本機備份並排程重試
[/flow]

### 本機備份是必要設計
- 測試站可能在工廠、實驗室或隔離網段，網路不穩定是常態
- 上傳失敗時不能丟失測試結果，應先保存本機報告與 raw log
- 重試上傳要具備 idempotency，避免同一份結果被重複建立多次
- 本機備份要有清理策略，例如保留天數、最大容量與已上傳標記

### Runner 錯誤分類

| 錯誤類型 | 範例 | 建議處理 |
|---|---|---|
| Environment Error | Node 或 NI-VISA 版本不符 | 中止測試並產生環境診斷 |
| Resource Error | 找不到指定 VISA resource | 提示接線、位址與儀器電源檢查 |
| Command Error | SCPI 回應錯誤或逾時 | 記錄命令、參數、timeout 與 raw response |
| Parse Error | Binary frame 或量測值解析失敗 | 保存原始資料並標記 parser version |
| Upload Error | 結果無法送回 Server | 本機備份並排程重試 |

# Driver Wrapper：讓 Node 安全地控制儀器
> Node 應該負責流程控制與平台整合；底層 VISA I/O、SCPI 差異與 Binary 解析，應封裝在可測試的 Driver Wrapper 裡。

## Wrapper 的設計原則

### 不要讓測試流程直接拼命令
- 測試流程如果直接寫 `SOUR:VOLT 5`，未來換儀器型號就會牽動流程邏輯
- 流程應呼叫標準 action，例如 `source.setVoltage(5)`
- Driver Wrapper 根據 instrument profile 轉成 SCPI 或 Binary 命令
- 這樣才能做到命令集轉換、參數驗證與回應解析集中管理

```prompt [label="TypeScript Driver Wrapper 介面草案"]
type VisaResource = string;

interface InstrumentDriver {
  model: string;
  wrapperVersion: string;

  open(resource: VisaResource): Promise<void>;
  close(): Promise<void>;
  identify(): Promise<{ manufacturer: string; model: string; serial?: string }>;
  execute(action: string, params?: Record<string, unknown>): Promise<DriverResult>;
}

interface DriverResult {
  ok: boolean;
  value?: number | string | Buffer;
  unit?: string;
  raw?: string | Buffer;
  error?: { code: string; message: string };
}
```

### 建議模組切分
- `visa-transport`：負責開關 VISA Session、read/write、timeout 與 resource 掃描
- `scpi-session`：負責 command/query、termination、錯誤查詢與基本 response parse
- `binary-codec`：負責 frame header、payload、checksum、endianness 與 raw buffer 解析
- `instrument-driver`：依儀器型號實作標準 action
- `runner-runtime`：負責測試流程、報告、上傳與本機備份

### Binary 通訊要明確規格化
- Binary frame 要定義 header、length、command id、payload、checksum 與結尾規則
- endian、signed/unsigned、float 格式與縮放倍率要寫在 profile
- 每個 parser 都要有版本，報告中要保存 parser version
- raw buffer 要能保存，否則發生解析錯誤時無法追查

## SCPI 與儀器命令策略

### SCPI 命令要標準化但不能過度假設
- `*IDN?` 可以做基本識別，但不同儀器的回應格式仍可能不同
- 設定類命令與查詢類命令要分開描述
- 重要動作後要設計確認機制，例如 query 回讀或檢查 error queue
- timeout、等待穩定時間與 retry 次數應由 profile 或測試步驟控制

### 儀器類型差異

| 儀器類型 | 常見動作 | 設計注意 |
|---|---|---|
| DC/AC Source | 設定電壓、頻率、輸出開關 | 安全範圍、ramp、輸出前確認 |
| DC/AC Load | 設定負載模式、電流、功率 | 模式切換時間、保護條件 |
| 示波器 | 設定 trigger、timebase、擷取波形 | 大量資料、Binary waveform parse |
| LCR Meter | 設定頻率、量測模式、讀值 | 校正狀態、量測速度與精度 |

# 從草案到可落地系統
> 先讓平台跑通一條端到端測試，再逐步擴充命令集、流程編輯器與報告查詢。

## MVP 落地順序

### 第一階段：證明端到端可行

[flow]
1. 單一儀器 — 先支援一台 DC Source 或一台 LCR Meter
2. 單一通訊方式 — 先選 USB 或 TCPIP，降低 GPIB 與多站點變因
3. 基本命令 — 完成 `identify`、設定值、讀值與錯誤查詢
4. 測試包下載 — Server 發布一個固定 manifest，Runner 可下載與執行
5. 報告上傳 — Runner 上傳 pass/fail、量測值、raw log 與版本資訊
[/flow]

### 第二階段：擴充平台能力
- 加入儀器型號管理與命令集版本
- 建立測試步驟管理與腳本模板
- 做出流程編輯器，讓工程師以步驟組裝測試
- 加入測試包版本管理與下載權限
- 完成報告查詢、結果趨勢與錯誤追蹤

### 第三階段：強化可靠性
- Runner 自動更新或版本提醒
- 本機備份與上傳重試機制
- Driver Wrapper 自我診斷與相依檢查
- 測試流程 dry-run 與參數驗證
- Server 端審核流程，避免未驗證命令集被發布到產線

## 上線前檢查清單

### 平台檢查
- 測試包是否包含 package id、version、相依 runner version 與 driver version
- 報告是否保存儀器型號、序號、resource、命令 log、raw response 與判定結果
- Runner 是否能在 Server 離線時完成本機備份
- 上傳 API 是否能避免重複結果
- 命令集修改是否會產生新版本而不是覆蓋舊版本

### 儀器控制檢查
- 每個 action 是否有參數範圍與單位
- 每個 SCPI query 是否定義 response parser
- Binary frame 是否保存 raw buffer 與 parser version
- timeout、retry、等待穩定時間是否可由設定控制
- 錯誤碼是否能回到報告與查詢頁被追溯

# 總結

[summary]
- 分工 **Server 管知識，Client 控硬體** | 儀器 I/O 留在 Local Runner，測試流程與版本由 Server 統一治理
- 抽象 **Action 優先於命令字串** | 用 Driver Wrapper 把 SCPI、Binary 與廠商差異封裝起來
- 可靠 **版本與報告必須成對** | 每份測試結果都要能回查測試包、命令集、Runner 與 Wrapper 版本
- 落地 **先跑通端到端，再做編輯器** | MVP 先完成單一儀器、單一流程、報告上傳，再逐步擴充平台能力
[/summary]
