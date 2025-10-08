# 第 18 篇：Apple App Attest - Receipt 風險評估

## 前言：風險指標的取得方式

Receipt 提供 **風險評估指標**（Risk Metric），幫助判斷裝置是否可疑。

**本文重點：**
- Receipt 的取得流程（attestation receipt → API → 新 receipt）
- PKCS#7 Receipt 結構規格
- Receipt Fields 欄位定義（type 編號對應）
- Risk Metric 解讀

**前置知識整合：**
- PKCS#7（第13篇）→ SignedData 容器結構
- ASN.1（第11篇）→ DER 編碼格式
- Attestation（第16篇）→ attestation object 中的 receipt

## Receipt 的取得流程

### 兩種 Receipt

**1. Attestation Receipt（Type: "ATTEST"）**
```
來源：attestation object 中的 receipt 欄位
用途：用來向 Apple 換取新的 receipt
```

**2. Risk Receipt（Type: "RECEIPT"）**
```
來源：向 Apple API 請求取得
用途：包含 Risk Metric 風險分數
特性：✅ 含欄位 17（Risk Metric）
```

### 取得流程（簡要）

```
Step 1: 從 Attestation Object 提取 receipt
   ↓ attestationObject.attStmt.receipt

Step 2: 發送 HTTP POST 到 Apple Server
   ↓ URL: https://data.appattest.apple.com/v1/attestationData
   ↓ Body: Base64(receipt)
   ↓ Header: Authorization: <JWT>

Step 3: 收到新 receipt（含 Risk Metric）
   ↓ Response: Base64 編碼的新 receipt
   ↓ 此 receipt 欄位 6 = "RECEIPT"
   ↓ 欄位 17 包含風險分數

Step 4: 儲存新 receipt 供下次更新使用
   ↓ 下次用新 receipt 再打 API 可刷新風險分數
```

**重點：**
- Attestation 時的 receipt 本身**沒有**風險分數
- 必須用它向 Apple **換取**新的 receipt
- 新 receipt 才包含 Risk Metric（欄位 17）

## Receipt 的整體結構

### 📦 PKCS#7 外層

```
Receipt (PKCS#7 SignedData)
├── contentInfo
│   ├── contentType: 1.2.840.113549.1.7.2 (SignedData)
│   └── content: SEQUENCE {
│         version: 1
│         digestAlgorithms: [SHA-256]
│         contentInfo: {
│           contentType: 1.2.840.113549.1.7.1 (data)
│           content: SET OF Attributes ← 重點在這裡！
│         }
│         certificates: [Apple 簽發憑證]
│         signerInfos: [簽章資訊]
│       }
```

**這是 PKCS#7 的標準結構**（第13篇詳細介紹過）

## Receipt Fields：核心資料

### 📊 Receipt Fields 欄位定義（根據 Apple 官方）

| Field | 欄位名稱 | 資料類型 | 說明 | 範例 |
|-------|---------|---------|------|------|
| **2** | App ID | String | Team ID + Bundle ID | A1B2C3D4E5.com.example.app |
| **3** | Attested Public Key | Data | DER ASN.1 編碼的公鑰 | MIICxTCCAkugAwIBAgI... |
| **4** | Client Hash | Data | challenge 的 SHA-256 | 4f0e5a36eedd8009f255... |
| **5** | Token | Data | Apple 內部使用 | NojAMV3DBZGAqbUyKSGU... |
| **6** | Receipt Type | String | "ATTEST" 或 "RECEIPT" | RECEIPT |
| **12** | Creation Time | String (ISO 8601) | 收據建立時間 | 2020-06-22T14:40:08.819Z |
| **17** | Risk Metric | String | 風險分數（僅 Type=RECEIPT） | 5 |
| **19** | Not Before | String (ISO 8601) | 最早刷新時間 | 2020-07-22T14:40:38.819Z |
| **21** | Expiration Time | String (ISO 8601) | 過期時間 | 2020-08-22T14:40:38.819Z |

### 🔍 contentInfo.content 的結構

```
content: SET OF Attributes
├── Attribute (Field 2) {
│     type: 2
│     version: 1
│     value: "A1B2C3D4E5.com.example.app"
│   }
│
├── Attribute (Field 3) {
│     type: 3
│     version: 1
│     value: <DER 公鑰資料>
│   }
│
├── Attribute (Field 6) {
│     type: 6
│     version: 1
│     value: "RECEIPT"  ← 或 "ATTEST"
│   }
│
├── Attribute (Field 17) {  ← 僅 Type=RECEIPT 時存在
│     type: 17
│     version: 1
│     value: "5"  ← 風險分數
│   }
│
└── ... (其他欄位)
```


## Risk Metric 詳解

### 🎯 風險分數的意義（Field 17）

```
Risk Metric = "5"  // String 格式，表示數字
```

**官方定義：**

Risk Metric 表示**過去 30 天內，該裝置上產生的 attested keys 數量**。

```
數值越低 → 風險越低
數值越高 → 可能有問題

正常情況：
- 首次使用：1
- 重新安裝 App：+1
- 從備份還原：+1
- 裝置轉移給其他使用者：+1

異常情況（可疑）：
- 短時間內數值過高（例如 > 10）
- 可能是被破解的裝置為多個假 App 提供 attestation
```

**Apple 的設計目的：**

防止單一破解裝置服務多個訂閱者：
- 攻擊者破解一台裝置的 OS
- 繞過限制，為多個假 App 實例產生有效 assertion
- Risk Metric 會異常增長（同一裝置產生大量 keys）

**為什麼會增長？**
1. 使用者重新安裝 App（金鑰不保留）
2. 從備份還原（金鑰不保留）
3. 裝置轉移（前使用者的金鑰 + 新使用者的金鑰）

**隱私考量：**
- App Attest 金鑰不隨備份保留
- 每次重新產生金鑰會增加計數

## Receipt 驗證概念（根據 Apple 官方）

### 🔍 驗證重點

收到 receipt 後（包括 attestation 中的和 API 回傳的），需要驗證：

```
1. 驗證簽章
   ↓ PKCS#7 簽章驗證
   ↓ 使用 Apple App Attest Root CA

2. 評估憑證鏈
   ↓ 檢查到 Apple 根憑證的信任鏈

3. 解析 ASN.1 payload
   ↓ 提取 SET OF Attributes 中的欄位

4. 驗證 App ID (Field 2)
   ↓ 格式：TeamID.BundleID
   ↓ 防止跨 App 攻擊

5. 檢查 Creation Time (Field 12)
   ↓ 不超過 5 分鐘（防重放攻擊）

6. 驗證 Attested Public Key (Field 3)
   ↓ 與 attestation 時儲存的公鑰匹配

7. 評估 Risk Metric (Field 17)
   ↓ 僅 Receipt Type = "RECEIPT" 時存在
   ↓ 根據業務需求決定處理
```

**關鍵要點：**
- Creation Time 不超過 5 分鐘（官方建議）
- App ID 格式為 `TeamID.BundleID`（注意有點號）
- Risk Metric 僅在 Type="RECEIPT" 時存在
- 詳細驗證實作見「實作文件化」系列

### 💡 何時需要 Receipt？

**需要風險評估時：**
- 高價值交易（金融、支付）
- 敏感操作（修改密碼、刪除帳號）
- 異常行為偵測

**可以省略：**
- 一般 API 請求（查詢資料）
- 高頻率操作（效能考量）
- 已有其他風險控管機制

### 🔄 完整流程整合

```
📱 客戶端                          🖥️ 伺服器

1️⃣ Attestation（註冊）
   ↓
   generateKey()
   attest(challenge)
   ↓                              ↓
   attestationObject ─────────→   驗證 9 步
   (含 x5c + receipt)              ├─ 驗證憑證鏈 ✅
                                   ├─ 驗證 nonce ✅
                                   └─ (可選) 驗證 receipt ✅
                                   ↓
                                   儲存 (keyId, publicKey, counter=0)

2️⃣ Assertion（每次請求）
   ↓
   sign(requestData)
   ↓                              ↓
   assertion ─────────────────→   驗證 5 步
   (authData + signature)          ├─ 驗證簽章 ✅
                                   ├─ 驗證 counter ✅
                                   └─ 處理請求 ✅
                                   ↓
                                   更新 counter

⚠️ Receipt 只在 Attestation 階段出現
   用於額外的風險評估
```

## 本文小結

✅ **Receipt 取得流程**：Attestation receipt → 打 Apple API → 新 receipt（含風險分數）  
✅ **兩種 Receipt**：Type="ATTEST"（無風險分數）vs Type="RECEIPT"（含 Field 17）  
✅ **Receipt 結構**：PKCS#7 SignedData 容器，內含 SET OF Attributes  
✅ **Receipt Fields（官方定義）**：App ID、Attested Public Key、Client Hash、Receipt Type、Risk Metric 等 9 個欄位  
✅ **Risk Metric 意義**：過去 30 天內該裝置產生的 attested keys 數量（數值越低越好）  
✅ **驗證重點**：Creation Time < 5分鐘、App ID 匹配、公鑰一致、簽章有效  
✅ **使用時機**：高價值交易、敏感操作時才需要風險評估

### 🎓 格式知識應用

**本文用到的所有格式：**
- **PKCS#7**（第13篇）→ Receipt 的外層容器
- **ASN.1**（第11篇）→ DER 編碼格式
- **SET OF**（ASN.1）→ Attributes 集合結構
- **SEQUENCE**（ASN.1）→ 每個 Attribute 的結構

---

## 💡 補充資料

### 常見問題

**Q: Receipt 驗證會拖慢效能嗎？**
A: 會。PKCS#7 解析 + 簽章驗證需要 10-50ms。建議：
   - 只在必要時驗證（高價值交易）
   - 快取驗證結果（同一 session 內）

**Q: Risk Metric = 0 就絕對安全嗎？**
A: ❌ 不是！Risk Metric 只是參考指標，不能完全依賴。應結合：
   - 憑證驗證（必須）
   - 簽章驗證（必須）
   - 業務邏輯（使用者行為分析）

**Q: 可以跳過 Receipt 驗證嗎？**
A: ✅ 可以！Apple 官方說明 Receipt 是 **可選** 的。大多數 App 只驗證憑證和簽章就足夠了。

**Q: Development environment 的 Receipt 有什麼不同？**
A:
   - Environment = "development"（而非 "production"）
   - Risk Metric 可能不準確
   - 簽發憑證不同（測試用 CA）

### 進階資源

- [Apple App Attest Receipt 官方文件](https://developer.apple.com/documentation/devicecheck/assessing_fraud_risk)
- [PKCS#7 規格 RFC 2315](https://www.rfc-editor.org/rfc/rfc2315.html)
- [ASN.1 編碼規則](https://www.itu.int/rec/T-REC-X.690/)
