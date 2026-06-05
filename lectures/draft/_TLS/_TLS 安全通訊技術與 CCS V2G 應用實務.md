# TLS 安全通訊技術與 CCS/V2G 應用實務

## 課程目標

本課程旨在協助韌體工程師、EVSE工程師、V2G開發工程師及測試工程師深入了解：

* TLS加密技術原理
* SSL/TLS協定演進
* 金鑰交換與憑證驗證
* TCP Socket與TLS Socket程式設計
* OpenSSL重要API與Callback
* TLS 1.2與TLS 1.3差異
* CCS/V2G中的TLS應用
* DIN 70121、ISO 15118-2、ISO 15118-20安全架構

---

# 第一章 TLS 加密技術概論

## 1.1 為什麼需要 TLS

### 網路傳輸風險

* 明文傳輸問題
* 封包竊聽（Sniffing）
* 資料竄改（Tampering）
* 身分偽造（Spoofing）
* 中間人攻擊（MITM）

### 實際案例

* HTTP → HTTPS
* 線上銀行
* VPN
* CCS Plug & Charge

---

## 1.2 TLS 的三大安全目標

### Confidentiality（機密性）

資料加密保護

```text
明文：
ChargeParameterDiscoveryReq

加密後：
A7 9C 11 6F 82 ...
```

### Integrity（完整性）

避免資料遭修改

常用機制：

* HMAC
* AES-GCM
* ChaCha20-Poly1305

### Authentication（身份驗證）

驗證：

* Server 身份
* Client 身份

使用：

* X.509 Certificate
* Digital Signature

---

## 1.3 TLS 發展歷史

| 協定      | 狀態   |
| ------- | ---- |
| SSL 2.0 | 淘汰   |
| SSL 3.0 | 淘汰   |
| TLS 1.0 | 淘汰   |
| TLS 1.1 | 淘汰   |
| TLS 1.2 | 現行主流 |
| TLS 1.3 | 最新標準 |

---

# 第二章 SSL、TLS與金鑰交換原理

## 2.1 SSL 與 TLS 的關係

### SSL

由 Netscape 提出

### TLS

由 IETF 標準化

主要RFC：

* RFC 5246 (TLS 1.2)
* RFC 8446 (TLS 1.3)

---

## 2.2 對稱式加密

常見演算法：

* AES-128
* AES-256
* ChaCha20

特性：

* 加密速度快
* 適合大量資料傳輸

---

## 2.3 非對稱式加密

### RSA

```text
Public Key
Private Key
```

用途：

* 身分驗證
* 金鑰交換

### ECC

常見曲線：

* secp256r1
* secp384r1

CCS/V2G常用：

* ECDSA
* ECDH
* ECDHE

---

## 2.4 Hash Function

常見：

* SHA256
* SHA384

用途：

* Digital Signature
* Certificate Validation

---

## 2.5 Digital Signature

流程：

```text
Message
  ↓
Hash
  ↓
Private Key Sign
  ↓
Signature
```

驗證：

```text
Signature
  ↓
Public Key Verify
```

---

## 2.6 ECDH 金鑰交換

### 問題

如何透過公開網路建立共同Session Key？

### ECDH流程

Client：

```text
Private A
Public A
```

Server：

```text
Private B
Public B
```

雙方產生：

```text
Shared Secret
```

再導出：

```text
Session Key
```

---

## 2.7 Perfect Forward Secrecy (PFS)

使用：

```text
ECDHE
```

優點：

即使Server私鑰外洩

歷史通訊內容仍無法解密

---

# 第三章 TLS Handshake 詳解

## 3.1 TLS 1.2 Handshake

```text
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

---

## 3.2 TLS 1.3 Handshake

```text
ClientHello
ServerHello

EncryptedExtensions
Certificate
CertificateVerify
Finished

Finished
```

---

## 3.3 TLS 1.2 與 TLS 1.3 差異

| 項目               | TLS1.2 | TLS1.3 |
| ---------------- | ------ | ------ |
| RTT              | 2 RTT  | 1 RTT  |
| RSA Key Exchange | 支援     | 移除     |
| PFS              | 選用     | 強制     |
| Cipher Suite     | 複雜     | 簡化     |
| 安全性              | 高      | 更高     |

---

# 第四章 Certificate 與 PKI

## 4.1 X.509 憑證結構

包含：

* Subject
* Issuer
* Public Key
* Signature
* Validity

---

## 4.2 Certificate Chain

```text
Root CA
   ↓
Sub CA
   ↓
Leaf Certificate
```

---

## 4.3 CCS Certificate Chain

```text
V2G Root CA
    ↓
Sub CA
    ↓
OEM Certificate
    ↓
Contract Certificate
```

---

## 4.4 Plug & Charge Certificate

使用：

* Provisioning Certificate
* Contract Certificate

---

# 第五章 Socket 與 TLS Socket

## 5.1 TCP Socket 基礎

### Server

```c
socket();
bind();
listen();
accept();
recv();
send();
close();
```

### Client

```c
socket();
connect();
send();
recv();
close();
```

---

## 5.2 TLS Socket 架構

### TCP Socket

```text
APP
 ↓
TCP
 ↓
IP
```

### TLS Socket

```text
APP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

---

## 5.3 TLS Record Layer

```text
Record Header
Encrypted Payload
Authentication Tag
```

---

# 第六章 TCP/TLS 程式設計實務

## 6.1 OpenSSL 架構

主要物件：

* SSL_CTX
* SSL
* BIO
* X509
* EVP

---

## 6.2 TLS Server 建立流程

```c
SSL_CTX_new();

Load Certificate();

socket();

bind();

listen();

accept();

SSL_new();

SSL_set_fd();

SSL_accept();
```

---

## 6.3 TLS Client 建立流程

```c
SSL_CTX_new();

socket();

connect();

SSL_new();

SSL_set_fd();

SSL_connect();
```

---

## 6.4 資料傳輸

```c
SSL_write();

SSL_read();
```

---

## 6.5 關閉連線

```c
SSL_shutdown();
```

---

# 第七章 TLS重要API與Callback

## 憑證驗證 Callback

```c
SSL_CTX_set_verify();
```

---

## SNI Callback

```c
SSL_CTX_set_tlsext_servername_callback();
```

---

## ALPN Callback

```c
SSL_CTX_set_alpn_select_cb();
```

---

## Key Log Callback

```c
SSL_CTX_set_keylog_callback();
```

用途：

* Wireshark解密TLS封包

---

## Session Callback

```c
SSL_CTX_sess_set_new_cb();
```

---

## Info Callback

```c
SSL_CTX_set_info_callback();
```

用途：

* Handshake除錯

---

# 第八章 TLS 1.2 與 TLS 1.3 相容設計

## Version Negotiation

ClientHello：

```text
supported_versions
```

---

## Cipher Suite Negotiation

### TLS 1.2

```text
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```

### TLS 1.3

```text
TLS_AES_128_GCM_SHA256
```

---

## 相容性問題

* 舊版OpenSSL
* 舊版EVCC
* 舊版SECC

---

## 雙版本支援策略

```text
TLS 1.3
   ↓
TLS 1.2
```

---

# 第九章 TLS 在 CCS/V2G 中的應用

## CCS 通訊架構

```text
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

---

## V2G Stack

```text
Application Layer
ISO15118
EXI
TCP/TLS
IPv6
HomePlug GreenPHY
```

---

# 第十章 DIN 70121 與 TLS

## DIN 70121 特性

* 早期CCS標準
* 不支援PnC
* 無憑證管理

---

## Security Model

```text
No TLS
TCP Only
```

---

## 風險分析

* 無加密
* 無驗證
* MITM風險

---

# 第十一章 ISO 15118-2 與 TLS 1.2

## 使用 TLS 1.2

支援：

* ECDHE
* ECDSA
* AES128-GCM

---

## Plug & Charge 流程

```text
SLAC
 ↓
TCP
 ↓
TLS
 ↓
Certificate Exchange
 ↓
Authorization
```

---

## Certificate Installation

```text
CertificateInstallationReq
CertificateInstallationRes
```

---

## Contract Authentication

```text
AuthorizationReq
AuthorizationRes
```

---

# 第十二章 ISO 15118-20 與 TLS 1.3

## ISO15118-20 新功能

支援：

* AC Charging
* DC Charging
* BPT
* Wireless Charging

---

## TLS Requirement

```text
TLS 1.3
```

---

## TLS 1.3 安全提升

* 1 RTT
* 強制PFS
* 移除RSA Key Exchange
* Cipher Suite簡化

---

## Cipher Suite

```text
TLS_AES_128_GCM_SHA256

TLS_AES_256_GCM_SHA384
```

---

## 測試重點

### EVCC

* Certificate Verify
* TLS Handshake
* Contract Certificate

### SECC

* Certificate Chain
* OCSP
* Session Management

---

# 第十三章 實驗課程

## Lab 1

Wireshark分析TLS Handshake

## Lab 2

OpenSSL建立TLS Client

## Lab 3

OpenSSL建立TLS Server

## Lab 4

Google Test驗證TLS API

## Lab 5

EVCC ↔ SECC TLS Handshake模擬

## Lab 6

ISO 15118-2 Plug & Charge測試

## Lab 7

ISO 15118-20 TLS 1.3測試

---

# 附錄

## Appendix A OpenSSL常用指令

```bash
openssl x509 -text

openssl verify

openssl s_client

openssl s_server
```

---

## Appendix B Wireshark分析技巧

* ClientHello
* ServerHello
* Certificate
* Finished

---

## Appendix C CCS / ISO15118 常見測項

* TLS Negotiation
* Certificate Validation
* Certificate Installation
* Contract Authentication
* Plug & Charge Authorization
* Session Resume
* OCSP Validation
* Certificate Chain Verification
* TLS 1.2 / TLS 1.3 Compatibility Test

---
