# 第 17 篇：Apple App Attest - Assertion 驗證流程

## 前言：每次請求的身份證明

完成 Attestation 註冊後，每次 API 請求都需要附上 **Assertion**：證明「這個請求確實來自註冊過的裝置」。

**本文重點：**
- Assertion 的資料結構（signature + authenticatorData）
- 6 個驗證步驟的詳細解析
- Counter 防重放機制
- 與 Attestation 的差異

**前置知識整合：**
- CBOR（第9篇）→ authenticatorData 結構
- ECDSA（第8篇）→ signature 簽章驗證
- Attestation（第16篇）→ 註冊流程獲得的公鑰

## 從真實資料開始：這是什麼?

當客戶端發送 API 請求時，會同時送出 assertion：

### 📦 API 請求格式

```php
// 客戶端傳送的資料
$assertion = 'omlzaWduYXR1cmVYR...'  // Base64 編碼的 CBOR 資料

// + clientData（與後端協調的資料）
$clientData = $challenge;  // 或其他格式

// + API 請求的資料
$requestBody = '{"action":"transfer","amount":1000}';
```

### 🔓 Assertion CBOR 解碼後的結構

```php
// 後端用 CBOR 解碼 assertion 後得到：
array (
  'signature' => <binary>,           // ECDSA 簽章（二進位）
  'authenticatorData' => <binary>    // 認證資料（二進位）
)
```

**重點：**
- App 傳的是 **CBOR 編碼**的 assertion（類似 Attestation Object）
- 後端需要 **CBOR 解碼**才能拿到 signature 和 authenticatorData
- 相比 Attestation，結構簡單很多（只有兩個欄位）

## 兩個主要部分詳解

### 🔏 Part 1: authenticatorData（認證資料）

```php
'authenticatorData' => 'X@3�P���QL...'  // Base64 編碼的二進位資料
```

**這是什麼？**
- 類似 Attestation 的 authData，但更簡單
- 沒有公鑰資料（因為註冊時已經拿到了）
- 主要包含 counter（防重放攻擊的計數器）

#### authenticatorData 的內部結構

```
authenticatorData (總長度：37 bytes)
├── [0-31] RP ID Hash (32 bytes)
│   └── SHA-256(AppIDPrefix + "." + BundleID)
│       例如：SHA-256("ABC123DEF4.com.example.app")
│
├── [32] Flags (1 byte)
│   └── 0x01 = 00000001
│       └── Bit 0 (UP): User Present = 1
│
└── [33-36] Counter (4 bytes)
    └── 遞增的計數器（每次 +1）
```

### ✍️ Part 2: signature（簽章）

```php
'signature' => 'MEUCIQDj8F...'  // Base64 編碼的 ECDSA 簽章
```

**這是什麼？**
- 使用 Attestation 時建立的私鑰簽署的
- 格式：ECDSA-SHA256 簽章（DER 編碼）
- 簽署對象：`authenticatorData || clientDataHash` 的組合資料

## 完整驗證流程：一步一步來

### 🔍 完整的 6 個驗證步驟（根據 Apple 官方文件）

```
Step 1: 計算 clientDataHash
   ↓ clientDataHash = SHA-256(clientData)
   ↓ clientData 是 App 和後端協調的資料
   ↓ （例如：challenge、requestBody 或其他自訂格式）

Step 2: 串接資料並計算 nonce
   ↓ dataToSign = authenticatorData || clientDataHash
   ↓ nonce = SHA-256(dataToSign)

Step 3: 驗證簽章
   ↓ 使用 Attestation 時儲存的公鑰
   ↓ ECDSA_Verify(publicKey, nonce, signature)

Step 4: 計算 App ID Hash 並驗證 RP ID
   ↓ 計算：SHA-256(AppIDPrefix + "." + BundleID)
   ↓ 比對：是否等於 authenticatorData 的 RP ID Hash

Step 5: 驗證 Counter（防重放攻擊）
   ↓ 從資料庫讀取該 keyId 的上次 counter
   ↓ 比較：新 counter > 舊 counter（首次 > 0）
   ↓ 更新：儲存新 counter 到資料庫

Step 6: 驗證 clientData 中的 challenge
   ↓ 檢查 clientData 包含的 challenge
   ↓ 確認是伺服器先前發送的值

✅ 驗證通過！處理 API 請求
```

**關鍵要點：**
- **Step 1**：clientDataHash = SHA-256(clientData)
- **Step 2**：nonce = SHA-256(authenticatorData || clientDataHash)
- **Step 3**：簽章驗證對象是 nonce
- **Step 5**：Counter 必須嚴格遞增（防止重放攻擊）

### 💡 驗證概念說明

**為什麼要計算 nonce = SHA-256(authenticatorData || clientDataHash)？**

```
dataToSign = authenticatorData || clientDataHash
nonce = SHA-256(dataToSign)

作用：
1. clientDataHash 綁定請求資料（App 和後端協調的內容）
2. authenticatorData 包含：
   - RP ID Hash（綁定 App ID）
   - Counter（防止重放）
   - Flags（確認使用者在場）
3. nonce 確保簽章涵蓋完整的驗證資料

如果只簽 clientData：
❌ 無法防止跨 App 重放（不同 App 可能有相同請求）
❌ 無法防止時間重放（舊請求重新送出）
❌ 無法綁定 App 身份（RP ID 未包含）
```

**clientData 是什麼？**

Apple 官方沒有強制規定格式，由 App 和後端協調：

```
常見做法 1：直接用 challenge
clientData = challenge  // 伺服器事先發的隨機值

常見做法 2：包含更多資訊
clientData = JSON.stringify({
  challenge: "abc123",
  action: "transfer",
  amount: 1000
})

重點：
- App 呼叫 DCAppAttestService.generateAssertion(keyId, clientData)
- 後端接收 clientData 並驗證
- 確保 clientData 包含防重放機制（如 challenge、timestamp）
```

**Counter 如何防重放攻擊？**

```
場景：攻擊者攔截了一個合法請求

請求 1 (合法):
  authenticatorData: counter = 5
  signature: 有效
  ✅ 伺服器驗證通過，更新 counter = 5

攻擊者重放請求 1:
  authenticatorData: counter = 5 (重複！)
  signature: 有效（簽章本身沒問題）
  ❌ 伺服器拒絕：counter 必須 > 5

請求 2 (合法):
  authenticatorData: counter = 6
  ✅ 通過（6 > 5）
```

**實務建議：**
- 必須持久化儲存 counter（資料庫）
- 不可重置 counter（否則重放攻擊可行）
- Counter 異常時應警告（可能遭受攻擊）

## 與 Attestation 的對比

### 📊 流程對比

| 特性 | Attestation（註冊） | Assertion（驗證） |
|------|-------------------|-----------------|
| **時機** | 初次註冊 | 每次 API 請求 |
| **頻率** | 一次或極少 | 高頻（每次請求） |
| **資料大小** | 大（~5KB） | 小（~100 bytes） |
| **包含憑證** | ✅ x5c 憑證鏈 | ❌ 無 |
| **公鑰資料** | ✅ 在 authData 中 | ❌ 已儲存於伺服器 |
| **Counter** | 0 | 遞增（1, 2, 3...） |
| **驗證步驟** | 9 步 | 6 步 |

### 🔄 生命週期

```
1️⃣ Attestation（首次註冊）
   ↓
   伺服器儲存：
   - keyId
   - publicKey
   - counter = 0
   ↓
2️⃣ Assertion（第一次請求）
   ↓
   伺服器驗證 → 更新 counter = 1
   ↓
3️⃣ Assertion（第二次請求）
   ↓
   伺服器驗證 → 更新 counter = 2
   ↓
   ... (持續使用)
```

## 本文小結

✅ **Assertion 結構**：authenticatorData + signature（簡潔高效）  
✅ **6 個驗證步驟**（官方）：clientDataHash → nonce → 簽章 → RP ID → Counter → challenge  
✅ **clientData 彈性**：App 和後端協調格式（challenge、JSON 或其他）  
✅ **Counter 機制**：嚴格遞增，防止重放攻擊  
✅ **nonce 設計**：綁定 authenticatorData 和 clientData，確保完整性  
✅ **性能優勢**：資料小（~100 bytes）、離線驗證、適合高頻請求

### 🎓 格式知識應用

**本文用到的所有格式：**
- **CBOR**（第9篇）→ authenticatorData 結構（精簡版）
- **ECDSA**（第8篇）→ 簽章演算法
- **DER**（第11篇）→ 簽章編碼格式
- **SHA-256**（第8篇）→ clientDataHash 和 RP ID Hash

## 下一篇預告

了解如何驗證每次請求後，最後來看看可選的 **Receipt 風險評估**：

**第18篇：Apple App Attest - Receipt 風險評估**
- PKCS#7 Receipt 完整解析
- Receipt Fields 的 type 編號對應
- Risk Metrics（風險分數）解讀
- 實務應用：何時需要檢查 Receipt？

---

## 💡 補充資料

### 常見問題

**Q: Assertion 可以快取嗎？**
A: ❌ 不行！每次請求都必須產生新的 Assertion（counter 會遞增）。快取會導致 counter 驗證失敗。

**Q: Counter 上限是多少？**
A: 2^32 - 1（約 42 億）。實務上不太可能達到。如果真的到上限，需要重新 Attestation。

**Q: 可以跳過 Counter 驗證嗎？**
A: ❌ 強烈不建議！Counter 是防止重放攻擊的關鍵機制。跳過會嚴重削弱安全性。

**Q: 驗證失敗怎麼辦？**
A:
1. 檢查是否 counter 重複（重放攻擊）
2. 檢查簽章是否有效（可能公鑰錯誤）
3. 檢查 clientDataHash 計算是否正確
4. 記錄詳細錯誤資訊供除錯

### 進階資源

- [Apple App Attest - Assertion 官方文件](https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server)
- [ECDSA 簽章演算法](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
- [防重放攻擊設計模式](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html)

### 實用工具

- **Base64 解碼器**: https://www.base64decode.org/
- **ECDSA 簽章驗證**: OpenSSL command line
- **Counter 監控**: 建議使用 Prometheus + Grafana
