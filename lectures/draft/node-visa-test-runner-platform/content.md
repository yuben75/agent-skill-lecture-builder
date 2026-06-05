# Node VISA Test Runner Platform：自動化儀器測試平台規劃
> 以 Node.js、Python/pyVISA、NI-VISA、SCPI 與 Binary 通訊為核心，設計一套 Server 管理測試知識、Client Local Runner 負責硬體執行的自動化測試平台。

[summary]
- 分工 **Server 管理測試資產** | 儀器型號、命令集、測試步驟、腳本模板、流程版本與報告查詢集中治理
- 執行 **Local Runner 靠近硬體** | 使用者電腦安裝 Node.js、Python/pyVISA、NI-VISA、NI-488.2 與 Driver Wrapper，負責實際儀器 I/O
- 抽象 **Command Profile 轉換命令** | 將標準測試動作轉成 SCPI、Binary 或各式儀器命令集
- 橋接 **Python 負責底層儀器控制** | 用 pyVISA 處理 VISA resource、SCPI query、binary block 與儀器相容性
- 追溯 **測試包與報告版本化** | 每份結果都能回查測試包、Runner、Driver Wrapper 與命令集版本
[/summary]

# 平台要解決的問題

### 儀器控制不是單一程式問題
- 自動化測試通常同時面對 DC/AC Source、DC/AC Load、示波器與 LCR Meter 等不同儀器
- 儀器可能使用 GPIB、USB、TCPIP、Serial 等不同通訊介面
- NI-VISA 與 NI-488.2 Driver 通常必須安裝在連接儀器的本機測試站
- 測試邏輯、命令集、版本與報告若散落在各測試站，後續維護與追溯會非常困難

> **先切開知識管理與硬體執行**
> Server 不應直接控制每一台儀器；Server 應管理測試知識、版本與結果。
>
> Local Runner 才是靠近硬體的一層，負責下載測試包、檢查環境、連接儀器、執行測試並回傳結果。

### 建議的 Client/Server 分工

[flow]
1. Server 定義測試資產 — 儀器型號、命令集、測試步驟、腳本模板與流程版本
2. Server 發布測試包 — 將流程、命令 profile、限制條件與報告格式打包
3. Local Runner 下載測試包 — 檢查本機 Node.js、Python、pyVISA、NI-VISA、NI-488.2 與 Wrapper 版本
4. Local Runner 執行測試 — 透過 Python VISA Bridge 或 Driver Wrapper 操作儀器並收集 raw log
5. Server 接收結果 — 保存報告、版本資訊、環境資訊與查詢索引
[/flow]

### 平台分層

| Layer | 主要責任 | 典型內容 |
|---|---|---|
| Server Portal | 測試資產管理 | 儀器型號、SCPI 命令、Binary profile、測試流程、版本管理 |
| Package Service | 測試包發布 | Manifest、流程定義、命令 profile、相依版本 |
| Local Runner | 本機測試執行 | 下載、版本檢查、儀器連線、測試流程、報告、本機備份 |
| Python VISA Bridge | 底層儀器 I/O | pyVISA ResourceManager、SCPI query、binary block、raw log |
| Driver Wrapper | 儀器抽象 | Node API、Python bridge、NI-VISA Session、SCPI query、Binary frame、response parser |
| Instrument | 實體設備 | Source、Load、Scope、LCR Meter 與其他實驗室儀器 |

# Server 端功能規劃

### 儀器型號管理
- 保存品牌、型號、序號規則、支援介面與通訊參數
- 描述儀器能力範圍，例如電壓、電流、頻率、功率、量測精度與安全限制
- 維護不同韌體版本或選配功能造成的命令差異
- 提供 Local Runner 用來比對實體儀器與測試包需求的 profile

### SCPI 與 Binary 命令管理
- 將平台標準動作對應到實際 SCPI 命令或 Binary frame
- 命令定義要包含參數型別、範圍、單位、timeout 與回應解析規則
- 對查詢類命令保存 response parser，避免每個測試腳本自行解析字串
- 對 Binary 通訊保存 frame header、payload layout、checksum、endianness 與 parser version

```prompt [label="Command Profile 範例"]
{
  "instrumentType": "dc-source",
  "model": "SOURCE-600W",
  "transport": ["usb", "tcpip", "gpib"],
  "actions": {
    "setVoltage": {
      "protocol": "scpi",
      "command": "SOUR:VOLT {voltage}",
      "params": {
        "voltage": { "type": "number", "min": 0, "max": 60, "unit": "V" }
      }
    },
    "readCurrent": {
      "protocol": "scpi",
      "command": "MEAS:CURR?",
      "response": { "type": "number", "unit": "A" }
    }
  }
}
```

### 測試步驟與腳本模板
- 測試步驟應描述「動作、輸入、輸出、判定條件、失敗處理」
- 腳本模板負責封裝可重複的測試片段，例如上電、穩定等待、量測、斷電
- 流程編輯器應組合步驟與模板，而不是讓使用者直接拼 SCPI 字串
- 流程輸出應可序列化成測試包，讓 Local Runner 離線或弱網路環境也能執行

### 版本管理與測試包下載

[flow]
1. Draft — 工程師編輯儀器 profile、步驟與流程
2. Validate — Server 檢查參數範圍、相依版本與必要儀器
3. Release — 發布不可變的測試包版本
4. Download — Local Runner 依 package id 與 version 下載內容
5. Trace — 報告回傳時綁定 package、runner、wrapper 與 command profile 版本
[/flow]

> **測試包發布後不要覆蓋**
> 自動化測試結果常用於品質判定、出貨判定或問題追查。
>
> 如果同一個 package version 的內容被修改，後續就無法回答「這份報告到底是用哪一版流程與命令跑出來的」。

### Server API

| API | 用途 | 呼叫時機 |
|---|---|---|
| `GET /api/packages/:id/manifest` | 取得測試包 manifest | Runner 開始測試前 |
| `GET /api/packages/:id/download` | 下載測試包內容 | 首次執行或版本不一致 |
| `POST /api/results` | 上傳測試結果 | 測試完成、中止或失敗後 |
| `POST /api/runner/heartbeat` | 回報 Runner 狀態 | 長時間測試期間 |
| `GET /api/reports/:id` | 查詢報告 | 工程師檢視與追溯 |

# Client 端功能規劃

### 使用者電腦需要安裝
- Node.js Runtime：執行 Local Runner 與測試流程控制邏輯
- Local Runner：測試平台的本機代理程式
- Python Runtime：執行 pyVISA 腳本、儀器診斷工具與底層控制 bridge
- pyVISA：Python 的 VISA 控制套件，用來掃描 resource、開啟 session、送出 SCPI 與讀取回應
- NI-VISA Runtime：提供 VISA resource 掃描、session 建立與底層 I/O
- NI-488.2 Driver：支援 GPIB 介面與設備通訊
- Instrument Driver Wrapper：把儀器命令封裝成平台一致的 API

### 環境檢查要能診斷問題
- 檢查 Node.js 版本是否符合 Runner 支援範圍
- 檢查 Python 版本、virtual environment、`pyvisa` 與 `pyvisa-py` 安裝狀態
- 檢查 NI-VISA 是否能列出 VISA resources
- 檢查 NI-488.2 是否能辨識 GPIB adapter 與設備位址
- 檢查 Driver Wrapper 版本、支援型號與支援 protocol
- 檢查本機權限、防火牆、USB 權限與測試站識別碼

### Local Runner 的責任

[flow]
1. 啟動 — 載入 station id、runner version 與本機設定
2. 下載測試包 — 從 Server 取得 manifest 並下載缺少的版本
3. 檢查相依 — 比對 Node.js、Python、pyVISA、NI-VISA、NI-488.2、Driver Wrapper 與儀器 profile
4. 連接儀器 — 掃描 VISA resources 並依測試包需求匹配儀器
5. 執行流程 — 逐步呼叫 Driver Wrapper 或 Python VISA Bridge，送出 SCPI 或 Binary 命令
6. 產生報告 — 保存量測值、判定結果、raw command log、錯誤與版本資訊
7. 上傳結果 — 將報告回傳 Server，失敗時保存本機備份並排程重試
[/flow]

### Node 與 Python 的分工
- Node.js 適合處理平台整合、測試流程狀態機、Server API、報告格式化與使用者介面
- Python 適合處理儀器 I/O、pyVISA 相容性、快速驗證 SCPI、解析 binary waveform 與工程師除錯
- Local Runner 可以由 Node 管理流程，再透過 child process、HTTP localhost service 或 IPC 呼叫 Python bridge
- 不建議讓每個測試流程直接執行任意 Python 腳本；應透過受控的 action API 呼叫，才方便做版本、權限與報告追溯

> **Python 是儀器控制的實用底層**
> 許多自動化測試工程師已經用 Python 驗證過儀器命令，pyVISA 的生態也成熟。
>
> 在平台化時，不必強迫所有底層控制都改成 Node；更務實的做法，是讓 Node 管平台，Python 管 VISA I/O，兩者用清楚的 bridge contract 串起來。

### 本機備份與上傳重試
- 測試站可能處於工廠、實驗室或隔離網段，網路不穩定是常態
- 測試完成後要先保存本機報告，再嘗試上傳 Server
- 上傳 API 要支援 idempotency key，避免重試造成重複報告
- 本機備份需要清理策略，例如保留天數、最大容量與已上傳標記

# Python 控制儀器實作

### pyVISA 的角色
- pyVISA 是 Python 操作 VISA 的常用套件，背後可以使用 NI-VISA 作為 backend
- 它能透過 VISA resource string 控制 GPIB、USB、TCPIP、Serial 等儀器
- 對 SCPI 儀器，常見操作是 `write()` 設定、`query()` 查詢、`read_raw()` 讀取 binary data
- 對示波器 waveform、頻譜資料或大量量測資料，Python 的 `bytes`、`struct`、`numpy` 很適合做 binary parser

### 最小 pyVISA 範例

```prompt [label="Python 掃描儀器與查詢 *IDN?"]
import pyvisa

rm = pyvisa.ResourceManager()
resources = rm.list_resources()
print("VISA resources:", resources)

inst = rm.open_resource("USB0::0x1234::0x5678::INSTR")
inst.timeout = 5000
inst.write_termination = "\n"
inst.read_termination = "\n"

print(inst.query("*IDN?"))
inst.close()
```

### SCPI 控制範例

```prompt [label="Python 控制 DC Source"]
import pyvisa

def run_source_test(resource: str, voltage: float, current_limit: float):
    rm = pyvisa.ResourceManager()
    source = rm.open_resource(resource)
    source.timeout = 5000
    source.write_termination = "\n"
    source.read_termination = "\n"

    idn = source.query("*IDN?").strip()
    source.write("*CLS")
    source.write(f"SOUR:VOLT {voltage}")
    source.write(f"SOUR:CURR {current_limit}")
    source.write("OUTP ON")

    measured_v = float(source.query("MEAS:VOLT?"))
    measured_i = float(source.query("MEAS:CURR?"))
    error = source.query("SYST:ERR?").strip()

    source.write("OUTP OFF")
    source.close()

    return {
        "idn": idn,
        "setVoltage": voltage,
        "measuredVoltage": measured_v,
        "measuredCurrent": measured_i,
        "error": error,
    }
```

### Binary 資料讀取重點
- 示波器 waveform 常使用 IEEE 488.2 definite-length block，例如 `#900001024...`
- 讀取 binary block 時要使用 `read_raw()` 或儀器支援的 binary value API，避免被文字 termination 截斷
- Parser 要保存 endian、資料型別、scale、offset、單位與 parser version
- 報告中應保存 raw length、前幾 bytes hex dump、解析後點數與錯誤訊息，方便追查

```prompt [label="Python 讀取 binary raw data 概念"]
raw = scope.read_raw()

if not raw.startswith(b"#"):
    raise ValueError("Not an IEEE 488.2 binary block")

digits = int(raw[1:2])
payload_len = int(raw[2:2 + digits])
payload_start = 2 + digits
payload = raw[payload_start:payload_start + payload_len]

print("payload bytes:", len(payload))
```

### Node 呼叫 Python Bridge
- Node Runner 可以把測試 action 轉成 JSON，交給 Python bridge 執行
- Python bridge 回傳 JSON result，內容包含 `ok`、量測值、單位、raw log、錯誤碼與耗時
- Bridge 要固定輸入輸出 schema，避免測試流程直接依賴特定 Python 腳本細節
- Python 程式應由測試包或 Runner 管理版本，報告中要保存 bridge version 與 pyVISA backend 資訊

```prompt [label="Node 呼叫 Python bridge 的 JSON contract"]
{
  "action": "source.setVoltageAndMeasure",
  "resource": "USB0::0x1234::0x5678::INSTR",
  "params": {
    "voltage": 5.0,
    "currentLimit": 0.5
  },
  "timeoutMs": 5000
}
```

```prompt [label="Python bridge 回傳結果"]
{
  "ok": true,
  "values": {
    "voltage": { "value": 5.002, "unit": "V" },
    "current": { "value": 0.121, "unit": "A" }
  },
  "rawLog": [
    { "direction": "write", "data": "SOUR:VOLT 5.0" },
    { "direction": "query", "data": "MEAS:VOLT?", "response": "5.002" }
  ],
  "versions": {
    "bridge": "1.2.0",
    "pyvisa": "1.x",
    "backend": "NI-VISA"
  }
}
```

# Driver Wrapper 設計

### Node 與 Python 都不應直接散落 SCPI 字串
- 測試流程直接寫 `SOUR:VOLT 5`，會把儀器型號差異綁死在流程內
- 流程應呼叫標準 action，例如 `source.setVoltage(5)`
- Driver Wrapper 或 Python bridge 根據 instrument profile 轉成 SCPI 或 Binary 命令
- Wrapper 要集中處理參數驗證、timeout、retry、error queue 與 response parser

```prompt [label="TypeScript Wrapper 介面草案"]
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
  rawLog?: Array<{ direction: "write" | "read" | "query"; data: string }>;
  error?: { code: string; message: string };
}
```

### 建議模組切分
- `node-runner`：處理流程狀態、Server API、測試包下載、報告與本機備份
- `python-visa-bridge`：處理 pyVISA backend、resource 掃描、session open/close、read/write 與 timeout
- `scpi-session`：處理 command/query、termination、error queue 與字串回應解析，可由 Python 或 Node wrapper 實作
- `binary-codec`：處理 frame header、payload、checksum、endianness 與 Buffer parser，通常 Python 實作更容易驗證
- `instrument-driver`：依儀器型號實作標準 action
- `runner-runtime`：處理測試流程、報告、本機備份與 Server API

### 錯誤分類

| 錯誤類型 | 範例 | 建議處理 |
|---|---|---|
| Environment Error | Node.js、Python、pyVISA 或 NI-VISA 版本不符 | 中止測試並產生環境診斷 |
| Resource Error | 找不到指定 VISA resource | 提示接線、位址、電源與 driver 檢查 |
| Command Error | SCPI error queue 有錯或 query timeout | 保存命令、參數、timeout 與 raw response |
| Parse Error | Binary frame 解析失敗 | 保存 raw buffer、parser version 與錯誤位置 |
| Upload Error | 報告無法送回 Server | 保存本機備份並排程重試 |

# 從草稿到 MVP

### 第一階段：先跑通端到端
- 選一台儀器，例如 DC Source 或 LCR Meter
- 選一種通訊方式，例如 USB 或 TCPIP，先降低 GPIB 與多站點變因
- 先用 Python/pyVISA 完成 `identify`、設定值、讀值與錯誤查詢
- 再讓 Node Runner 呼叫 Python bridge，完成 report upload 與版本追溯
- Server 先提供固定 manifest，不急著做完整流程編輯器
- Runner 必須能下載測試包、執行、產生報告、上傳與保存本機備份

### 第二階段：擴充測試資產管理
- 加入儀器型號管理與 command profile 版本
- 建立測試步驟管理與腳本模板
- 加入流程編輯器，讓工程師以步驟組合測試流程
- 測試包發布要經過驗證與版本鎖定
- 報告查詢要能回查儀器、測試站、命令 log 與版本資訊

### 第三階段：提升可靠性
- Runner 自動更新或版本提醒
- Driver Wrapper 自我診斷與相依檢查
- Python bridge 自我診斷、virtual environment 固定與 pyVISA backend 偵測
- 測試流程 dry-run 與參數範圍檢查
- 本機備份容量控管與上傳重試佇列
- Server 端審核流程，避免未驗證命令集直接發布到產線

# 上線檢查清單

### Server 檢查
- 儀器型號是否描述能力範圍、支援介面與通訊參數
- 命令集是否包含參數驗證、response parser、錯誤碼與版本
- 測試包是否不可變，並包含 package id、version 與相依版本
- 結果 API 是否支援重試與避免重複上傳
- 報告查詢是否能追溯 package、runner、wrapper、profile 與儀器資訊

### Client 檢查
- Runner 是否能在 Server 離線或網路失敗時保存本機備份
- VISA resource 掃描是否能輸出可診斷的錯誤訊息
- Python/pyVISA 是否能獨立完成 `*IDN?` 與基本 SCPI query
- Node Runner 呼叫 Python bridge 是否有 timeout、schema validation 與錯誤分類
- Driver Wrapper 是否保存 raw command log 與 raw response
- Binary parser 是否保存 raw buffer 與 parser version
- 上傳失敗後是否能安全重試且不建立重複報告

# 總結

[summary]
- 架構 **Server 管知識，Runner 控硬體** | 這能同時保留中央治理與本機儀器 I/O 的可靠性
- 分工 **Node 管平台，Python 管 VISA I/O** | Node 負責流程與 API，Python/pyVISA 負責儀器連線、SCPI、Binary 與相容性
- 命令 **Action 優先於字串** | 用 Driver Wrapper、Python Bridge 與 Command Profile 隔離 SCPI、Binary 與儀器差異
- 結果 **版本追溯不可省略** | 測試報告必須能回查當時的測試包、Runner、Wrapper 與命令集版本
- 落地 **先完成一條端到端流程** | 不要一開始就做完整平台，先讓一台儀器、一個測試包、一份報告跑通
[/summary]
