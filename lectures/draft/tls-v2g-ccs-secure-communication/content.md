# TLS 安全通訊技術與 CCS/V2G 應用實務
> 從 TLS 密碼學、握手流程、憑證鏈與 OpenSSL API，銜接到 CCS Plug & Charge、DIN 70121、ISO 15118-2 與 ISO 15118-20 的車樁安全通訊設計。

[summary]
- 對象 **韌體與 EVSE 工程師** | 用工程實作角度理解 TLS Socket、OpenSSL API 與封包除錯
- 基礎 **密碼學到握手流程** | 串起對稱加密、非對稱加密、ECDHE、PFS、憑證驗證與 Session Key
- 應用 **CCS/V2G 安全模型** | 比較 DIN 70121、ISO 15118-2、ISO 15118-20 對 TLS 與憑證的要求
- 實驗 **可驗證的測試路徑** | 以 Wireshark、OpenSSL、EVCC/SECC 模擬與 Google Test 建立測試能力
[/summary]

## 為什麼 EV 充電通訊需要 TLS

### 從一般網路風險看充電場景
- 明文封包容易被 Sniffing 取得，例如充電參數、授權訊息或合約憑證交換資料
- 未受保護的訊息可能被 Tampering，造成參數被竄改或授權流程被干擾
- 若缺乏身分驗證，攻擊者可以偽裝成 EVCC 或 SECC，形成 Spoofing 與 MITM 風險
- HTTP 轉 HTTPS、線上銀行、VPN 與 CCS Plug & Charge，本質上都在處理「不可信網路上的可信通訊」

> **TLS 不是單純把資料加密**
> 在 V2G 場景中，TLS 同時負責保護通訊內容、驗證通訊對象，並讓上層 ISO 15118 流程可以建立在可信連線之上。

### TLS 的三個安全目標
- Confidentiality：用對稱式加密保護通訊內容，避免明文被第三方讀取
- Integrity：用 HMAC、AES-GCM 或 ChaCha20-Poly1305 這類機制確認資料未被修改
- Authentication：透過 X.509 Certificate 與 Digital Signature 驗證 Server 與 Client 身分

```prompt [label="明文與密文概念"]
明文：
ChargeParameterDiscoveryReq

加密後：
A7 9C 11 6F 82 ...
```

### SSL/TLS 的演進狀態
- SSL 2.0、SSL 3.0 已淘汰，不應再出現在新系統設計中
- TLS 1.0、TLS 1.1 也已不適合現代安全要求
- TLS 1.2 仍是大量既有系統與 ISO 15118-2 的重要基礎
- TLS 1.3 是新標準，對延遲、演算法組合與前向安全性做了更強約束

## 密碼學元件與金鑰交換

### 對稱式加密與非對稱式加密的分工
- AES-128、AES-256、ChaCha20 速度快，適合大量資料傳輸
- RSA、ECC 能處理身分驗證、金鑰交換或簽章驗證，但不適合直接加密大量資料
- TLS 的核心設計是先用非對稱機制建立信任與共享秘密，再用對稱金鑰保護後續資料

### ECC、ECDH 與 ECDHE
- ECC 常見曲線包含 secp256r1 與 secp384r1
- CCS/V2G 常見組合包含 ECDSA、ECDH 與 ECDHE
- ECDSA 主要用於簽章與身分驗證
- ECDH/ECDHE 用於雙方在公開網路上導出 shared secret

[flow]
1. Client 產生 Private A 與 Public A - 將 Public A 放入握手資料
2. Server 產生 Private B 與 Public B - 將 Public B 回傳給 Client
3. 雙方各自計算 Shared Secret - 不需要在網路上傳送真正的 Session Key
4. 從 Shared Secret 導出 Session Key - 後續 Record Layer 用它加密與驗證資料
[/flow]

### Hash 與 Digital Signature
- SHA256、SHA384 常用於摘要、憑證驗證與簽章流程
- Digital Signature 通常是先對 Message 做 Hash，再用 Private Key 簽章
- 驗證端用 Public Key 檢查 Signature，確認資料來源與內容一致性

```prompt [label="Digital Signature 流程"]
Message
  ↓
Hash
  ↓
Private Key Sign
  ↓
Signature

Signature
  ↓
Public Key Verify
```

> **PFS 是長期風險控制**
> Perfect Forward Secrecy 的價值在於：即使 Server 私鑰未來外洩，過去已擷取的通訊內容仍不應被還原。

## TLS Handshake 與版本差異

### TLS 1.2 Handshake
- TLS 1.2 的握手訊息較多，典型流程需要 2 RTT
- Server 會傳送 Certificate 與 ServerKeyExchange
- 雙方透過 ChangeCipherSpec 與 Finished 進入加密通訊狀態

```prompt [label="TLS 1.2 Handshake"]
ClientHello
ServerHello
Certificate
ServerKeyExchange
ServerHelloDone

ClientKeyExchange
ChangeCipherSpec
Finished

ChangeCipherSpec
Finished
```

### TLS 1.3 Handshake
- TLS 1.3 將握手流程簡化為 1 RTT
- EncryptedExtensions 之後的多數握手資料已受到保護
- 移除 RSA Key Exchange，強制使用具備 PFS 的金鑰交換設計

```prompt [label="TLS 1.3 Handshake"]
ClientHello
ServerHello

EncryptedExtensions
Certificate
CertificateVerify
Finished

Finished
```

### TLS 1.2 與 TLS 1.3 的工程差異
- RTT：TLS 1.2 通常 2 RTT，TLS 1.3 通常 1 RTT
- RSA Key Exchange：TLS 1.2 可支援，TLS 1.3 已移除
- PFS：TLS 1.2 視 cipher suite 而定，TLS 1.3 成為基本要求
- Cipher Suite：TLS 1.2 命名包含金鑰交換、簽章、加密與 MAC；TLS 1.3 大幅簡化
- 除錯方式：TLS 1.3 更多握手內容被加密，分析封包時更需要 key log

## Certificate、PKI 與 Plug & Charge

### X.509 憑證的最小閱讀重點
- Subject：憑證持有者身分
- Issuer：簽發此憑證的 CA
- Public Key：用於驗證簽章或進行協定所需的公鑰操作
- Signature：Issuer 對憑證內容的簽章
- Validity：憑證有效期間，測試時常見問題包含過期與時間設定錯誤

### Certificate Chain
- Root CA 是信任錨點，通常預先安裝或由信任儲存區管理
- Sub CA 負責中介簽發，降低 Root CA 直接暴露風險
- Leaf Certificate 是實際 Server、Client 或合約身分使用的憑證

```prompt [label="一般憑證鏈"]
Root CA
   ↓
Sub CA
   ↓
Leaf Certificate
```

### CCS/V2G 憑證鏈
- V2G Root CA 是整個 V2G PKI 的信任基礎
- OEM Certificate 與 Contract Certificate 讓車輛、使用者合約與充電服務之間可以建立信任
- Provisioning Certificate 常用於初始配置與合約憑證安裝流程

```prompt [label="CCS/V2G 憑證鏈"]
V2G Root CA
    ↓
Sub CA
    ↓
OEM Certificate
    ↓
Contract Certificate
```

> **Plug & Charge 的難點在憑證生命週期**
> PnC 不只是 TLS 能連上而已，還包含憑證安裝、更新、撤銷、鏈驗證、合約授權與跨角色信任管理。

## TCP Socket 到 TLS Socket 的程式設計

### TCP Socket 基礎流程
- Server 建立 socket、bind、listen，等待 accept 後進行 recv/send
- Client 建立 socket、connect 成功後進行 send/recv
- TCP 只提供可靠傳輸，不提供加密、完整性保護或對端身分驗證

```prompt [label="TCP Server API 順序"]
socket();
bind();
listen();
accept();
recv();
send();
close();
```

```prompt [label="TCP Client API 順序"]
socket();
connect();
send();
recv();
close();
```

### TLS Socket 架構
- Application Layer 不再直接把明文交給 TCP
- TLS Layer 負責握手、加解密、完整性驗證與 alert
- TCP/IP 仍然是底層傳輸基礎

```prompt [label="TLS Socket 層次"]
APP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

### TLS Record Layer
- Record Header 描述資料型態、版本與長度
- Encrypted Payload 承載受保護的應用資料或握手資料
- Authentication Tag 用於 AEAD 模式的完整性驗證

## OpenSSL 實作流程與重要 Callback

### OpenSSL 核心物件
- SSL_CTX：全域設定與憑證、CA、版本、cipher suite 等共用狀態
- SSL：單一連線的 TLS 狀態
- BIO：I/O 抽象，可連接 socket、memory 或自訂傳輸
- X509：憑證解析與驗證
- EVP：加密、簽章、雜湊等高階密碼學 API

### TLS Server 建立流程

[flow]
1. SSL_CTX_new - 建立 Server 共用 TLS 設定
2. Load Certificate - 載入憑證鏈與私鑰
3. socket/bind/listen/accept - 建立 TCP Server 並接受連線
4. SSL_new 與 SSL_set_fd - 將 TLS 狀態綁定到 TCP fd
5. SSL_accept - 執行 Server 端 TLS Handshake
6. SSL_read 與 SSL_write - 進入受 TLS 保護的資料收發
[/flow]

### TLS Client 建立流程

[flow]
1. SSL_CTX_new - 建立 Client TLS 設定
2. socket/connect - 先完成 TCP 連線
3. SSL_new 與 SSL_set_fd - 將 TLS 狀態綁定到 TCP fd
4. SSL_connect - 執行 Client 端 TLS Handshake
5. SSL_get_verify_result - 檢查憑證驗證結果
6. SSL_shutdown - 正常關閉 TLS 連線
[/flow]

### 常用 Callback 的使用時機
- `SSL_CTX_set_verify()`：自訂憑證驗證策略或蒐集驗證錯誤資訊
- `SSL_CTX_set_tlsext_servername_callback()`：處理 SNI，依 server name 選擇憑證或設定
- `SSL_CTX_set_alpn_select_cb()`：協商上層協定，例如 HTTP/2 或特定應用 profile
- `SSL_CTX_set_keylog_callback()`：輸出 key log，搭配 Wireshark 解密 TLS 封包
- `SSL_CTX_sess_set_new_cb()`：觀察或管理 session 建立
- `SSL_CTX_set_info_callback()`：追蹤握手狀態，協助定位 handshake failure

> **Callback 是可觀測性的入口**
> TLS 連不上時，單看錯誤碼通常不夠。把 verify、info、key log callback 補齊，才容易判斷問題在憑證鏈、版本、cipher suite 還是狀態機。

## TLS 1.2 與 TLS 1.3 相容設計

### Version Negotiation
- ClientHello 會攜帶可支援版本資訊
- TLS 1.3 使用 `supported_versions` extension 表達可接受版本
- 實務上常見 EVCC、SECC、OpenSSL 版本不一致，因此要明確定義 fallback 策略

### Cipher Suite Negotiation
- TLS 1.2 範例：`TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`
- TLS 1.3 範例：`TLS_AES_128_GCM_SHA256`
- TLS 1.2 的 cipher suite 名稱包含金鑰交換與簽章資訊，TLS 1.3 則把這些設定拆到其他 extension 與演算法協商中

### 相容性問題
- 舊版 OpenSSL 不一定支援 TLS 1.3 或特定曲線
- 舊版 EVCC/SECC 可能只實作 TLS 1.2 與固定 cipher suite
- 測試環境若 CA、系統時間、憑證用途或 chain order 錯誤，也會被誤判為 TLS 版本問題

```prompt [label="雙版本支援策略"]
優先嘗試：
TLS 1.3

必要時 fallback：
TLS 1.2
```

## TLS 在 CCS/V2G 架構中的位置

### CCS 通訊堆疊
- EVCC 與 SECC 在底層透過 IPv6 與 HomePlug GreenPHY 建立通訊
- TCP/TLS 提供可靠且安全的通訊通道
- EXI 負責 ISO 15118 XML 訊息的高效率編碼
- ISO 15118 上層處理充電協商、授權、憑證安裝與合約驗證

```prompt [label="CCS TLS 位置"]
EVCC
 ↓
TCP
 ↓
TLS
 ↓
EXI
 ↓
ISO15118
```

```prompt [label="V2G Stack"]
Application Layer
ISO15118
EXI
TCP/TLS
IPv6
HomePlug GreenPHY
```

### DIN 70121 的安全模型
- DIN 70121 屬於較早期 CCS 標準
- 不支援 Plug & Charge
- 沒有完整憑證管理與 TLS 安全通道設計
- 安全模型多為 TCP Only，因此具備明文、無驗證與 MITM 風險

> **DIN 70121 的限制要被清楚標註**
> 若產品同時支援舊標準與新標準，測試報告應明確區分「協定相容」與「安全保證」。能通訊不代表具備 PnC 等級的安全性。

## ISO 15118-2 與 TLS 1.2

### ISO 15118-2 的 TLS 重點
- 主要使用 TLS 1.2
- 常見能力包含 ECDHE、ECDSA 與 AES128-GCM
- Plug & Charge 流程依賴憑證鏈與 Contract Certificate 完成授權

[flow]
1. SLAC - 建立電力線通訊鏈路
2. TCP - 建立 EVCC 與 SECC 的可靠連線
3. TLS - 完成憑證驗證與安全通道
4. Certificate Exchange - 安裝或交換 PnC 所需憑證資料
5. Authorization - 使用 Contract Certificate 完成合約授權
[/flow]

### Certificate Installation 與 Contract Authentication
- `CertificateInstallationReq` / `CertificateInstallationRes` 用於合約憑證安裝
- `AuthorizationReq` / `AuthorizationRes` 用於合約授權與 PnC 驗證
- 測試時要同時看 TLS 層與 ISO 15118 訊息層，避免只驗證其中一半

## ISO 15118-20 與 TLS 1.3

### ISO 15118-20 的新範圍
- 支援 AC Charging 與 DC Charging
- 支援 BPT，也就是 Bidirectional Power Transfer
- 支援 Wireless Charging 等更完整的充電情境
- 對 TLS 1.3 的採用讓安全性與握手效率同步提升

### TLS 1.3 安全提升
- 1 RTT 握手降低連線建立延遲
- 強制 PFS，降低長期私鑰外洩造成的歷史資料風險
- 移除 RSA Key Exchange，減少不安全或過時組合
- Cipher Suite 簡化，降低配置錯誤機率

```prompt [label="TLS 1.3 Cipher Suite"]
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
```

### EVCC 與 SECC 測試重點
- EVCC：Certificate Verify、TLS Handshake、Contract Certificate
- SECC：Certificate Chain、OCSP、Session Management
- 雙方：TLS 1.2 / TLS 1.3 compatibility、session resume、憑證有效期與 chain order

## 實驗與測試路線

### 實驗設計
- Lab 1：用 Wireshark 分析 TLS Handshake，識別 ClientHello、ServerHello、Certificate 與 Finished
- Lab 2：用 OpenSSL 建立 TLS Client，觀察憑證驗證與錯誤處理
- Lab 3：用 OpenSSL 建立 TLS Server，載入憑證鏈與私鑰
- Lab 4：用 Google Test 驗證 TLS API 封裝與錯誤分支
- Lab 5：模擬 EVCC 與 SECC TLS Handshake
- Lab 6：驗證 ISO 15118-2 Plug & Charge 流程
- Lab 7：驗證 ISO 15118-20 TLS 1.3 流程

### OpenSSL 常用指令

```prompt [label="憑證與連線檢查"]
openssl x509 -text
openssl verify
openssl s_client
openssl s_server
```

### Wireshark 分析重點
- ClientHello：版本、cipher suite、supported groups、signature algorithms
- ServerHello：選定版本與 cipher suite
- Certificate：憑證鏈是否完整、Issuer/Subject 是否符合預期
- Finished：確認握手進入受保護狀態
- Key Log：需要時搭配 OpenSSL keylog callback 解密應用資料

## 課後檢查清單

- [x] 能說明 TLS 的機密性、完整性與身分驗證分別由哪些機制支撐
- [x] 能畫出 TLS 1.2 與 TLS 1.3 的握手差異
- [x] 能辨識 X.509 憑證鏈與 CCS/V2G 憑證鏈角色
- [x] 能用 OpenSSL 建立基本 TLS Client/Server 並處理憑證驗證
- [x] 能說明 DIN 70121、ISO 15118-2、ISO 15118-20 的安全模型差異
- [x] 能規劃 EVCC/SECC 的 TLS handshake、PnC 與 compatibility 測項

> **最後帶走的一件事**
> 在 CCS/V2G 系統中，TLS 不是附加功能，而是 Plug & Charge 與合約授權能否被信任的基礎。工程實作要同時掌握協定、憑證、OpenSSL API 與封包觀測能力。
