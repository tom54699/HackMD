# 第 13 篇：PKCS#7 - 簽章資料容器

## 前言：什麼是 PKCS#7？

### 為什麼需要 PKCS#7？

當你需要傳遞**帶有數位簽章的資料**時，會遇到一些問題：

**問題情境**：
```
你要傳送一份重要文件給對方：
├── 文件本身（資料）
├── 你的數位簽章（證明是你簽的）
└── 你的憑證（讓對方能驗證簽章）

怎麼把這三樣東西打包在一起？
```

**PKCS#7 的解決方案**：
- 定義一個**標準化的容器格式**
- 把「資料 + 簽章 + 憑證」打包在一起
- 任何系統都能用相同方式解析和驗證

---

### 實際應用場景

**1. 數位簽章文件**
- PDF 簽章、電子合約
- 保證文件來源可信、內容未被竄改

**2. 電子收據/憑證**
- App Store Receipt（內購收據）
- App Attest Receipt（裝置驗證）
- 郵件加密（S/MIME）

**3. 軟體分發**
- 程式碼簽章
- 軟體更新包

---

### 本文目標

前面學過的格式：
- **X.509**（第10篇）：憑證格式
- **ASN.1**（第11篇）：編碼方式

現在要學的：
- **PKCS#7**：簽章資料的標準容器
- 如何把資料、簽章、憑證打包在一起
- 驗證流程的概念

## 📦 理解 PKCS#7：一個安全的資料信封

### 生活類比：掛號信封的結構

想像你要寄一封重要文件的掛號信：

```
📮 一般信封：
├── 信封內：文件
└── 外觀：寄件人地址、收件人地址

❌ 問題：
   ├── 任何人都能打開
   ├── 內容可能被調包
   └── 無法證明確實是你寄的

📮 PKCS#7 信封（掛號+簽章）：
├── 📄 原始文件（你的購買收據）
├── 🔏 數位簽章（Apple 的簽章）
├── 📜 簽章者憑證（Apple 的 X.509 憑證）
└── 📋 額外憑證鏈（可能包含中繼 CA 憑證）

✅ 保證：
   ├── 只有 Apple 能製作有效簽章
   ├── 內容若被竄改，簽章驗證會失敗
   └── 包含完整的憑證鏈，可以驗證到根 CA
```

### PKCS#7 的核心概念

**PKCS#7**（Public-Key Cryptography Standards #7），也稱為 **CMS**（Cryptographic Message Syntax）

```
PKCS#7 不是一種加密演算法
而是一種「打包格式」

作用：
├── 把「資料」+「簽章」+「憑證」打包在一起
├── 定義標準的結構，所有系統都能解讀
└── 可以包含多層簽章、多個簽章者

類比：
就像 ZIP 檔案是「壓縮打包格式」
PKCS#7 是「簽章打包格式」
```

## 🏗️ PKCS#7 的基本結構

### 完整的層次結構

```
SignedData (簽章資料)
├── version (版本號)
├── digestAlgorithms (摘要演算法集合)
│   └── sha256  （例如：SHA-256）
├── contentInfo (內容資訊)
│   ├── contentType (內容類型)
│   └── content (實際資料)
│       └── 📄 你的購買收據 JSON/ASN.1 資料
├── certificates (憑證集合) ⭐ 關鍵
│   ├── [0] Apple 簽章憑證（直接簽署收據的憑證）
│   ├── [1] 中繼 CA 憑證
│   └── [2] 可能的其他憑證
└── signerInfos (簽章者資訊集合)
    └── SignerInfo
        ├── version (版本)
        ├── issuerAndSerialNumber (簽章者識別)
        │   ├── issuer: Apple CA
        │   └── serialNumber: 憑證序號
        ├── digestAlgorithm (摘要演算法)
        ├── signedAttrs (簽章屬性) - 可選
        │   ├── contentType
        │   ├── signingTime (簽章時間)
        │   └── messageDigest (內容摘要)
        ├── signatureAlgorithm (簽章演算法)
        └── signature (數位簽章) 🔏
            └── Apple 用私鑰簽署的結果
```

### 關鍵欄位說明

#### 1. contentInfo - 實際資料

**用途**：存放要保護的資料

**為什麼需要？**
- PKCS#7 可以簽署任何類型的資料
- 需要明確標示「這是什麼類型的資料」

**App Store Receipt 範例**：
```
contentInfo:
├── contentType: 1.2.840.113549.1.7.1 (pkcs7-data)
└── content:
    └── SET OF Attributes
        ├── 收據類型
        ├── Bundle ID
        ├── 應用程式版本
        ├── 購買時間
        ├── 原始交易 ID
        └── ...其他購買資訊
```

#### 2. certificates - 憑證鏈

**用途**：提供驗證簽章所需的憑證

**為什麼需要？**
```
問題情境：
你的伺服器收到一個 PKCS#7 Receipt
├── 有 Apple 的數位簽章
└── 但你怎麼知道這個簽章對應的公鑰是 Apple 的？

解決方案：
PKCS#7 直接包含憑證！
├── certificates[0]: Apple 簽章憑證（包含公鑰）
├── certificates[1]: Apple 中繼 CA 憑證
└── 你只需要驗證這條憑證鏈到 Apple Root CA
```

#### 3. signerInfos - 簽章資訊

**用途**：包含數位簽章本身和相關資訊

**為什麼需要？**
```
要驗證簽章，需要知道：
├── 誰簽的？ → issuerAndSerialNumber
├── 用什麼演算法？ → signatureAlgorithm
├── 簽什麼內容？ → digestAlgorithm
└── 簽章值是多少？ → signature
```

**App Store Receipt 範例**：
```
signerInfo:
├── issuerAndSerialNumber:
│   ├── issuer: "Apple Inc."
│   └── serialNumber: 0x1234567890abcdef
├── digestAlgorithm: SHA-256
├── signedAttrs (簽章前屬性):
│   ├── contentType: pkcs7-data
│   ├── signingTime: 2024-01-15T10:30:00Z
│   └── messageDigest: [content 的 SHA-256 hash]
├── signatureAlgorithm: RSA
└── signature: [Apple 私鑰簽署 signedAttrs 的結果]
```

## 🔐 PKCS#7 的解析與驗證概念

### 如何解析 PKCS#7 結構

PKCS#7 使用 **ASN.1 DER 編碼**，需要專門的解析器：

```
原始資料：二進位的 PKCS#7 檔案
└── 使用 ASN.1 DER 編碼（第11篇）

解析流程：
┌────────────────────────────┐
│ 1. 讀取二進位資料            │
│ 2. 用 ASN.1 解析器解碼       │
│ 3. 提取各欄位                │
└────────────────────────────┘

解析後得到（簡化表示）：
├── contentInfo (實際資料)
├── certificates (憑證鏈)
└── signerInfos (簽章資訊)
```

**關鍵要點**：
- 不能用 JSON 解析器（這是二進位格式）
- 需要 ASN.1 函式庫（phpseclib、pyasn1、node-forge）
- 結構是巢狀的，需要按規範導航

---

### 驗證概念（3 個核心步驟）

```
收到 PKCS#7 後的驗證邏輯：

步驟 1：驗證憑證鏈
   ↓
   確保簽章憑證確實由可信 CA 簽發
   （例如：Apple Root CA）

步驟 2：驗證簽章
   ↓
   使用憑證中的公鑰驗證簽章
   確認資料未被竄改

步驟 3：解析業務資料
   ↓
   提取 content 中的具體欄位
   （例如：Bundle ID、時間戳記）
```

**signedAttrs 的作用**：

PKCS#7 通常不直接簽 content，而是簽 **signedAttrs**（簽章屬性）：

```
為什麼？

直接簽 content:
├── signature = sign(content)
└── ❌ 無法加入簽章時間等額外資訊

簽 signedAttrs: ⭐ PKCS#7 標準做法
├── signedAttrs 包含：
│   ├── contentType (內容類型)
│   ├── messageDigest = SHA-256(content)
│   └── signingTime (簽章時間)
├── signature = sign(signedAttrs)
└── ✅ 優點：
    ├── 可包含時間戳記
    ├── messageDigest 仍保護 content
    └── 防止重放攻擊

驗證時：
1. 驗證 signature 對 signedAttrs 有效
2. 驗證 messageDigest == SHA-256(content)
```

## 🔍 PKCS#7 應用實例

### 實例：電子收據驗證

以 App Store Receipt 為例，說明 PKCS#7 的實際應用：

```
PKCS#7 SignedData (Receipt 檔案)
│
├── 📋 contentInfo
│   ├── contentType: data (1.2.840.113549.1.7.1)
│   └── content: [收據資料，用 ASN.1 編碼]
│
├── 🪪 certificates
│   ├── [0] 簽章憑證
│   └── [1] 中繼 CA 憑證
│
└── ✍️ signerInfo
    ├── digestAlgorithm: SHA-256
    ├── signatureAlgorithm: RSA
    └── signature: [數位簽章]
```

**驗證流程**：
1. **PKCS#7 層**：驗證憑證鏈和簽章
2. **ASN.1 層**：解析 content 內容
3. **業務層**：檢查具體欄位（App ID、時間等）

---

### contentInfo.content 的內部結構

`contentInfo.content` 欄位由發行方自訂，Apple Receipt 使用 **SET OF Attributes** 格式：

```
Content = SET OF Attribute
每個 Attribute = SEQUENCE {
    type: INTEGER     (屬性編號，如 2 = Bundle ID)
    version: INTEGER  (版本號)
    value: OCTET STRING (實際資料)
}
```

**解析後看到的結構：**
```
解析 PKCS#7 → 得到 contentInfo.content
→ 這是一個 SET（集合）
  → 裡面有多個 Attribute (SEQUENCE)
    → 每個 Attribute 有 type、version、value
      → 根據 type 編號判斷這是什麼欄位
        → 例如 type=2 代表 Bundle ID
```

**如何觀測：**
```bash
# 用 OpenSSL 查看 Receipt 結構
openssl asn1parse -inform DER -in receipt -i

# 輸出會顯示：
# SET
#   SEQUENCE (第一個 Attribute)
#     INTEGER: 2        ← type (Bundle ID)
#     INTEGER: 1        ← version
#     OCTET STRING: ... ← value
#   SEQUENCE (第二個 Attribute)
#     INTEGER: 3        ← type (App Version)
#     ...
```

**type 編號對應：**（Apple Receipt 範例）
- type=2: Bundle ID
- type=3: App Version
- type=4: Opaque Value
- type=5: SHA-1 Hash
- ...（依規格定義）

---

### 為什麼選擇 PKCS#7？

**與其他方案比較**：

| 方案 | 優點 | 缺點 | 適用場景 |
|------|------|------|---------|
| **PKCS#7** | 包含完整憑證鏈<br>離線可驗證<br>標準成熟 | 格式複雜<br>需 ASN.1 解析器 | 長期保存的憑證<br>電子收據、合約 |
| **JWT** | 輕量、易Debug<br>Web 友善 | 需另外取得公鑰<br>不含憑證鏈 | 短期 token<br>API 認證 |
| **直接簽章** | 最簡單 | 需自行管理憑證<br>無標準格式 | 簡單場景 |

**PKCS#7 的選擇時機**：
- ✅ 需要長期保存（數年）
- ✅ 需要離線驗證
- ✅ 需要攜帶完整憑證鏈
- ✅ 需要跨平台相容

---

## 🔄 格式關係

### PKCS#7 與其他格式的關係

**PKCS#7 使用 X.509**：
- `certificates` 欄位存放的是 X.509 憑證
- 用憑證裡的公鑰來驗證簽章

**PKCS#7 和 X.509 都用 ASN.1**：
- 都使用 ASN.1 DER 編碼
- 需要相同的解析工具
- 可轉換成 PEM 文字格式

**JWT 是現代替代方案**：
- 更輕量（JSON + Base64）
- 更適合 Web 開發
- 但不含憑證鏈（需另外管理公鑰）

## 實務要點

**工具選擇：**
- ASN.1 解析函式庫（phpseclib、pyasn1、node-forge）
- OpenSSL 命令列工具（查看結構）

**查看 PKCS#7 結構：**
```bash
openssl asn1parse -inform DER -in receipt -i
```

**使用建議：**
- 理解結構概念即可，實際驗證使用現成函式庫
- 重點在於知道「PKCS#7 包含哪些部分」
- 詳細驗證步驟見實務應用篇（第18篇）

---

## 🎯 本文小結

✅ **PKCS#7 是什麼**：標準化的「簽章資料容器」格式，打包資料、簽章、憑證  
✅ **核心結構**：SignedData = contentInfo + certificates + signerInfo  
✅ **解析方式**：使用 ASN.1 解析器，需要專門函式庫  
✅ **驗證概念**：憑證鏈驗證 → 簽章驗證 → 業務資料檢查（3步）  
✅ **contentInfo.content**：可觀測 SET OF Attributes 結構（type 編號對應）  
✅ **應用場景**：電子收據、數位簽章文件、軟體分發  
✅ **設計優勢**：離線可驗證、內建憑證鏈、跨平台標準

---

## 💡 常見問題

**Q: PKCS#7 和 CMS 有什麼差別？**
A: CMS（RFC 5652）是 PKCS#7 的後續標準，功能更強大，但基本概念相同。實務上兩者可以互換使用。

**Q: 為什麼不用 JWT 做 Receipt？**
A: JWT 更適合短期 token，PKCS#7 包含完整憑證鏈，更適合長期保存的憑證（訂閱可能數年）。而且 Receipt 格式在 JWT 標準化之前就確立了。

**Q: signedAttrs 一定要有嗎？**
A: 不一定，但建議使用。有 signedAttrs 可以包含簽章時間等額外資訊，提高安全性。Apple 的 Receipt 一定會有 signedAttrs。

**Q: 解析 PKCS#7 為什麼這麼複雜？**
A: PKCS#7 使用 ASN.1 DER 編碼，是巢狀的 TLV（Tag-Length-Value）結構。解析器會保留所有層級，需要根據規範導航到正確的索引位置。這也是為什麼需要專門的 ASN.1 程式庫。

---

## 📚 延伸閱讀

- [RFC 2315 - PKCS #7: Cryptographic Message Syntax](https://tools.ietf.org/html/rfc2315)
- [RFC 5652 - CMS (Cryptographic Message Syntax)](https://tools.ietf.org/html/rfc5652)
- [Apple Receipt Validation Guide](https://developer.apple.com/documentation/appstorereceipts)

---

## 📖 下一篇預告

**第14篇：JWT 與 JWS 格式解析**
- 現代的輕量級簽章格式
- 與 PKCS#7 的對比
- Google Play Integrity 的應用

**系列四：實務應用**

完成格式基礎後，將整合所有知識：

**第16篇：Apple App Attest - Attestation 註冊**
- CBOR（第9篇）+ X.509（第10篇）+ ASN.1（第11篇）+ X9.62（第12篇）

**第18篇：Apple App Attest - Receipt 風險評估**
- **PKCS#7 容器**（本篇）的實際應用
- Receipt Fields 完整解析（type 編號對應）
- 業務整合範例
