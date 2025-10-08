# 第 16 篇：Apple App Attest - Attestation 註冊流程

## 前言：裝置註冊的關鍵

Apple App Attest 的第一步是 **Attestation（認證註冊）**：裝置向伺服器證明「我是真實的 iOS 裝置」，並建立一組專屬金鑰對。

**本文重點：**
- Attestation Object 的完整結構（CBOR + X.509 + PKCS#7）
- 9 個驗證步驟的詳細解析
- 每個格式在實際驗證中的角色

**前置知識整合：**
- CBOR（第9篇）→ attestationObject 外層包裝
- X.509（第10篇）→ x5c 憑證鏈
- PKCS#7（第13篇）→ 簽章容器格式
- ASN.1（第11篇）→ 所有二進位資料的編碼方式

## 從真實資料開始：這是什麼？

當你的 iPhone 完成 App Attest 後，會回傳一個 attestation object。讓我們看看它長什麼樣子：

### 📦 CBOR 解碼後的結構

```php
array (
  'fmt' => 'apple-appattest',           // 格式標識

  'attStmt' => array (                  // 證明聲明
    'x5c' => array (                    // 憑證鏈
      0 => '0��0�9����MJ...',          // 葉憑證（二進位）
      1 => '0�C0�Ƞ ...',               // 中繼憑證（二進位）
    ),
    'receipt' => '0�    *�H��...',     // App Attest Receipt（可選，二進位）
  ),

  'authData' => 'X@3�P���QL...',        // 認證資料（二進位）
)
```

**看起來很亂？別擔心！** 我們一步步來理解。

## 三個主要部分詳解

### 📋 Part 1: fmt（格式標識）

**這是什麼？**
```
'fmt' => 'apple-appattest'
```

就像郵件上的「掛號」標籤，告訴你：「這是 Apple App Attest 格式的資料」。

**為什麼需要？**
- 因為 CBOR 可以包裝各種資料
- 這個標籤讓伺服器知道如何處理
- 如果不是 'apple-attest'，就知道格式錯了

**驗證方式：**
```php
if ($attestationObject['fmt'] !== 'apple-appattest') {
    throw new Exception('格式錯誤！');
}
```

### 📝 Part 2: attStmt（證明聲明）

這是最重要的部分，包含兩個東西：

#### 🔐 x5c（憑證鏈）

還記得第 7 篇講的憑證鏈嗎？這裡就是實際的憑證！

```php
'x5c' => array (
  0 => '0��0�9����MJ...',  // 葉憑證（你的 iPhone + App）
  1 => '0�C0�Ƞ ...',       // 中繼憑證（Apple App Attestation CA）
)
```

**為什麼是兩張憑證？**

```
Apple Root CA (根憑證 - 要自己下載)
    ↓ 簽發
x5c[1]: Apple App Attestation CA (中繼憑證)
    ↓ 簽發
x5c[0]: 你的 iPhone + App (葉憑證)
```

**葉憑證裡面有什麼？**

用工具解開後會看到：
```
Subject: 你的 iPhone + App
Issuer: Apple App Attestation CA
Extensions（擴展欄位）:
  ├── App ID: com.example.myapp
  ├── Team ID: ABC123DEF4
  └── Public Key: (你的 App Attest 公鑰)
```

**中繼憑證裡面有什麼？**

```
Subject: Apple App Attestation CA
Issuer: Apple Root CA
Public Key: (Apple CA 的公鑰，用來驗證葉憑證)
```

#### 🧾 receipt（App Attest Receipt）

```php
'receipt' => '0�    *�H��...'  // 很長的二進位資料
```

**這是什麼？**
- App Attest Receipt 是 Apple 提供的裝置環境風險評估憑證
- 通常在 Assertion 流程相關場景中取得，用於輔助風險判斷
- 格式與欄位為 App Attest 專用，不同於 App Store IAP 收據

**你需要驗證嗎？**
- 可選！此為進階功能，大部分情況憑證驗證就夠了
- 如果需要額外的裝置風險評估，可參考 Apple 官方文件處理 Receipt 驗證

### 🔑 Part 3: authData（認證資料）

```php
'authData' => 'X@3�P���QL...'
```

這是一串二進位資料，包含了很多重要資訊。讓我們拆解它：

#### authData 的內部結構

```
authData (總長度：可變)
├── [0-31] RP ID Hash (32 bytes)
│   └── SHA-256(App ID Prefix + "." + Bundle ID)
│       例如：SHA-256("ABC123DEF4.com.example.myapp")
│       App ID Prefix 通常是你的 10 位數 Team ID
│
├── [32] Flags (1 byte)
│   └── 0x41 = 01000001
│       ├── Bit 0 (UP): User Present = 1
│       └── Bit 6 (AT): Attested Credential Data = 1
│
├── [33-36] Counter (4 bytes)
│   └── 0x00000000 (Attestation 時總是 0)
│       Assertion 時會遞增
│
└── [37+] Attested Credential Data
    ├── AAGUID (16 bytes)
    │   ├── Development: "appattestdevelop"
    │   └── Production: "appattest" (10 bytes) + 6 個 0x00 = 16 bytes
    │
    ├── Credential ID Length (2 bytes)
    │   └── 通常是 32
    │
    ├── Credential ID (32 bytes)
    │   └── SHA-256(公鑰) - 公鑰的雜湊值
    │
    └── Public Key (CBOR 格式)
        └── COSE Key 格式的公鑰（X, Y 座標）
```

**欄位說明（根據官方文件）：**

- **RP ID Hash**：App ID 的雜湊，格式為 `AppIDPrefix.BundleID`
  - App ID Prefix 通常是 10 位數 Team ID
  - 可在 Apple Developer Account 查詢

- **AAGUID**：環境識別
  - Development：`"appattestdevelop"` (16 bytes ASCII)
  - Production：`"appattest"` (10 bytes) + 6 個 `0x00` = 16 bytes

- **Credential ID**：公鑰的 SHA-256 雜湊
  - 用於唯一識別這個 attested key
  - 等同於 App 傳給後端的 `keyId`

## 完整驗證流程：一步一步來

現在我們知道資料結構了，來看看如何驗證：

### 🔍 完整的 9 個驗證步驟（根據 Apple 官方文件）

```
Step 1: 驗證 x5c 憑證鏈起始於 credential certificate
   ↓ x5c[0] 是葉憑證（credCert）
   ↓ x5c[1] 是中繼憑證
   ↓ 使用 Apple App Attest Root CA 驗證憑證有效性

Step 2: 準備 clientDataHash
   ↓ clientDataHash = 伺服器提供的 32 bytes challenge 原值
   ↓ （本專案約定：直接使用伺服器發給 App 的 32 bytes 隨機值）
   ↓ 將 clientDataHash 附加到 authData 後面

Step 3: 建構 nonce
   ↓ composite = authData || clientDataHash
   ↓ nonce = SHA-256(composite)
   ↓ 這是憑證驗證的關鍵值

Step 4: 從 credCert 提取擴展欄位中的 nonce
   ↓ 找到 OID 1.2.840.113635.100.8.2 擴展
   ↓ 解碼 DER 編碼的 ASN.1 SEQUENCE
   ↓ 提取其中的 OCTET STRING
   ↓ 驗證：提取的值 == Step 3 計算的 nonce

Step 5: 計算 credCert 公鑰的雜湊並驗證 key identifier
   ↓ 提取 credCert 的公鑰（X9.62 uncompressed point format）
   ↓ 計算：SHA-256(publicKey)
   ↓ 驗證：雜湊值 == App 傳來的 keyId

Step 6: 計算 App ID 雜湊並驗證 RP ID
   ↓ 計算：SHA-256(AppIDPrefix + "." + BundleID)
   ↓ 驗證：雜湊值 == authData 的 RP ID Hash

Step 7: 驗證 counter = 0
   ↓ Attestation 時 authenticatorData 的 counter 必須為 0

Step 8: 驗證 aaguid 環境標識
   ↓ Development: "appattestdevelop" (16 bytes)
   ↓ Production: "appattest" (10 bytes) + 6 個 0x00 = 16 bytes

Step 9: 驗證 credentialId
   ↓ authenticatorData 的 credentialId == keyId

✅ 驗證通過！儲存 keyId 和公鑰供 Assertion 使用
```

**關鍵要點：**
- **Step 3**：nonce 綁定 challenge 和 authData（防止重放攻擊）
- **Step 4**：憑證擴展的 nonce 必須匹配（證明憑證是為此次請求簽發）
- **Step 5**：公鑰雜湊必須等於 keyId（確保公鑰對應）
- **Step 6**：RP ID 格式為 `AppIDPrefix.BundleID`（注意有點號）

### 💡 驗證概念說明

**nonce 機制為什麼重要？（Step 3-4）**

```
nonce = SHA-256(authData || clientDataHash)

作用：
1. clientDataHash 綁定伺服器的 challenge（防止重放攻擊）
2. authData 包含公鑰、RP ID、counter 等關鍵資訊
3. 憑證擴展中的 nonce 必須匹配（證明憑證是為這次請求簽發的）

安全保證：
✅ 每次 attestation 都需要新的 challenge
✅ 憑證中的 nonce 綁定了 authData 和 challenge
✅ 攻擊者無法重複使用舊的 attestation
```

**實務建議：**
- 使用現成函式庫處理 CBOR/ASN.1 解析
- 重點理解驗證邏輯（為什麼需要每個步驟）
- 詳細實作範例見「實作文件化」系列


## 本文小結

✅ **Attestation Object 結構**：CBOR 包裝（fmt + attStmt + authData）  
✅ **9 個驗證步驟**：從解碼 CBOR 到驗證 nonce，缺一不可  
✅ **nonce 機制**：綁定 challenge 和 authData，防止重放攻擊  

### 🎓 格式知識應用

**本文用到的所有格式：**
- **CBOR**（第9篇）→ attestationObject 外層
- **X.509**（第10篇）→ x5c 憑證鏈
- **ASN.1**（第11篇）→ 憑證和 Receipt 編碼
- **X9.62**（第12篇）→ COSE Key 內部的公鑰點
- **PKCS#7**（第13篇）→ receipt 容器（可選）
- **WebAuthn**（第15篇）→ authData 結構來源

## 下一篇預告

完成 Attestation 註冊後，接下來每次 API 呼叫都需要 **Assertion 驗證**：

**第17篇：Apple App Attest - Assertion 驗證流程**
- Assertion 的資料結構（signature + authenticatorData）
- 如何驗證每次 API 請求的真實性
- Counter 防重放機制
- 與 Attestation 的差異

**第18篇：Apple App Attest - Receipt 風險評估**
- PKCS#7 Receipt 完整解析
- Receipt Fields 的 type 編號對應
- Risk Metrics 風險分數
- 實務應用建議

---

## 💡 補充資料

### 常見問題

**Q: 為什麼 attestation object 這麼複雜？**
A: 因為要同時滿足：安全性（多層驗證）、效能（二進位格式）、標準化（WebAuthn 相容）。

**Q: 我一定要理解所有細節嗎？**
A: 不用！使用現成函式庫即可。但理解原理能幫你：
   - 除錯問題
   - 選擇合適的函式庫
   - 評估安全性

### 進階資源

- [Apple App Attest 官方文件](https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server)
- [WebAuthn 規格](https://www.w3.org/TR/webauthn-2/) - App Attest 基於此標準
- [CBOR 規格 RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html)
- [X.509 規格 RFC 5280](https://www.rfc-editor.org/rfc/rfc5280.html)

### 實用工具

- **CBOR 解碼器**: http://cbor.me
- **ASN.1 解碼器**: https://lapo.it/asn1js/
- **JWT 解碼器**: https://jwt.io
- **OpenSSL**: 命令列工具，用於憑證分析
