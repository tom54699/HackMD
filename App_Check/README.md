# AI Init – API 防護實作與知識補全專案

## 專案背景

本專案起源於公司 SMS API 遭受異常流量攻擊，攻擊來源尚不明確，可能為：
- 直接對 API 端點進行暴力攻擊
- 使用模擬器偽造合法裝置環境呼叫 API

為降低風險並提升 API 安全性，本專案導入多層防護機制：

- **Google reCAPTCHA Enterprise**：判斷人類與機器流量，阻擋惡意請求  
- **Google Play Integrity API**：驗證 Android 裝置與應用執行環境完整性  
- **Apple App Attest**：驗證 iOS 裝置與 App 執行環境完整性  

---

## 專案目標

本專案的目標不僅是完成上述驗證機制的技術實作，更重要的是將整個過程中所涉及的**前置背景知識**系統化整理，協助團隊快速理解底層原理。

### 目標拆解

1. **實作文件化**  
   - 整理各驗證機制的整合流程與程式碼範例，作為公司內部技術參考文件。

2. **知識補全計畫**  
   - 將實作過程中遇到的各種底層概念（如 ECC、CBOR、ASN.1、X.509、OID 等）系統化拆解為系列文章，建立可重用的學習資源。

---

## 📚 系列文章大綱

以下為專案的完整知識地圖，分為四個模組，從安全背景到密碼學原理、格式解析再到協定應用，循序漸進構建完整知識體系。

---

### 📘 系列一：API 安全背景與驗證框架

**第 1 篇：API 攻擊模型與防禦思路**  
- 常見攻擊類型：暴力打 API、模擬器、Token 竊取  
- 為何需要「裝置驗證」這一層安全防護  

**第 2 篇：三大驗證機制全景圖**  
- reCAPTCHA、Play Integrity、App Attest 的定位與差異  
- 為何 App Attest 涉及更深的密碼學與憑證背景  

---

### 📙 系列二：密碼學演算法基礎（理論層）

**第 3 篇：ECC 與 ECDSA 原理**  
- 橢圓曲線密碼學 (ECC) 基本原理與數學直覺  
- 私鑰、公鑰產生流程  
- ECDSA 簽章與驗證演算法解析  

**第 4 篇：公開金鑰與信任鏈概念**  
- 公私鑰加密與簽章的不同應用  
- 信任鏈 (Trust Chain)、CA、Root、Intermediate 的角色  
- 憑證鏈驗證邏輯與常見陷阱  

**第 5 篇：數位簽章與驗證應用**  
- ECDSA vs RSA vs EdDSA  
- JWT、TLS、WebAuthn 中的簽章應用  
- Server 簽章驗證流程與注意事項  

---

### 📗 系列三：密碼學資料格式與編碼（中介層）

這是 App Attest 中最容易卡關的部分，理解這一層才能解讀 attestation payload 與憑證結構。

**第 9 篇：資料序列化與 CBOR**
- CBOR 與 JSON、ASN.1 的差異
- CBOR 資料結構解析與範例
- WebAuthn / App Attest payload 中的 CBOR 實例解析

**第 10 篇：X.509 憑證格式**
- X.509 憑證的三層結構
- 憑證本體（tbsCertificate）的關鍵欄位
- 憑證鏈如何運作

**第 11 篇：ASN.1 與編碼格式**
- 為什麼 X.509 憑證看起來像亂碼
- ASN.1 是什麼？DER、BER、PEM 編碼差異
- TLV 結構、SEQUENCE、INTEGER、BIT STRING

**第 12 篇：X9.62 橢圓曲線公鑰格式**
- X9.62 標準的橢圓曲線點編碼格式
- 未壓縮格式（0x04 || X || Y）與壓縮格式
- 在 App Attest 憑證中的應用與格式轉換

**第 13 篇：PKCS#7 - 簽章資料容器**
- PKCS#7 的用途：打包「資料 + 簽章 + 憑證」
- SignedData 結構
- App Store Receipt 驗證

**第 14 篇：JWT 與 JWS 格式解析**
- JWT 的三段式結構：Header.Payload.Signature
- 為什麼需要產生 JWT？Base64URL 編碼
- 呼叫 Apple API 的認證 token 產生

**第 15 篇：WebAuthn 與裝置信任模型**
- WebAuthn 標準與 App Attest 的關聯
- Authenticator Data、Client Data 結構
- 理解 App Attest 的設計理念

---

### 📒 系列四：協定層與實務應用（設計層）

**第 16 篇：Apple App Attest 資料格式實例解析**
- 整合前面所有格式知識
- 完整解析一個真實的 attestation object
- 實際開發時如何處理這些格式

**第 17 篇：Apple 安全生態與 App Attest 原理**
- Secure Enclave、DeviceCheck、App Attest 的角色定位
- attestation / assertion 流程解析與憑證鏈驗證
- Server 驗證邏輯與常見錯誤處理

**第 18 篇：實務錯誤與安全陷阱分析**
- 常見錯誤與失敗原因（CBOR parsing、簽章驗證、憑證鏈錯誤）
- 測試與 debug 技巧
- 常見風險與防禦策略總結  

---

## 📊 知識結構總覽（學習進階路線）

| 層級 | 主題 | 學習目標 |
|------|------|----------|
| 背景層 | 攻擊模型、防護需求 | 理解為什麼需要裝置驗證 |
| 理論層 | ECC、ECDSA、公鑰、信任鏈 | 掌握簽章的數學與安全原理 |
| 中介層 | CBOR、ASN.1、DER、OID、Base64 等 | 能解讀 payload、憑證與簽章資料結構 |
| 協定層 | WebAuthn、App Attest | 能實作、驗證並進行安全分析 |


## 參考 url

### Google Recaptcha
1. https://cloud.google.com/recaptcha/docs/interpret-assessment-website?hl=zh_TW
2. https://cloud.google.com/recaptcha/docs/best-practices-oat?hl=zh_TW

### Google Play Integrity
1. https://developer.android.com/google/play/integrity/verdicts?hl=zh-tw#environment-details-field
2. https://developer.android.com/google/play/integrity/additional-tools?hl=zh-tw
3. https://developer.android.com/google/play/integrity/standard?hl=zh-tw

### Apple App Attest
1. https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server
2. https://developer.apple.com/documentation/devicecheck/assessing-fraud-risk
3. https://developer.apple.com/help/account/keys/create-a-private-key/

### 相關知識
1. https://officeguide.cc/openssl-command-ecdsa-elliptic-curve-digital-signature-creation-and-verification-tutorial-examples/
2. https://zhaox.github.io/block%20chain/2018/04/26/ecc-public-key
3. https://www.cnblogs.com/charliecza/p/17467640.html
4. https://blog.techbridge.cc/2019/08/17/webauthn-intro/