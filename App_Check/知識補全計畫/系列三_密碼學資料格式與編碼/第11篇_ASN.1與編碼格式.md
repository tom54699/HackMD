# 第 11 篇：ASN.1 與編碼格式 - 為什麼憑證看起來像亂碼？

## 前言：解開憑證「亂碼」的謎團

在第 10 篇，我們了解了 X.509 憑證的邏輯結構：它包含哪些欄位（version、subject、issuer、publicKey 等）、每個欄位的用途、憑證鏈如何透過 subject 和 issuer 配對。

這些概念很清楚。但當你實際查看憑證檔案時，會看到這樣的東西：

```
30 82 02 cc 30 82 02 34 a0 03 02 01 02 02 10 1a 2b 3c 4d...
```

或是這樣：

```
-----BEGIN CERTIFICATE-----
MIICzDCCAbSgAwIBAgIQGis8TTBAQEBAQEBAQEBAQEBAQDANBgkqhkiG9w0BAQUF
ADBOMQswCQYDVQQGEwJVUzEQMA4GA1UEChMHRXF1aWZheDEtMCsGA1UECxMkRXF1
...
-----END CERTIFICATE-----
```

**完全看不懂！這跟我們理解的「version、subject、publicKey」結構有什麼關係？**

這就是本篇要解開的謎題：
- 為什麼憑證要用這種「亂碼」格式儲存？
- ASN.1 和 DER 編碼是什麼？
- DER、BER、PEM 有什麼差異？
- 如何把「邏輯結構」轉換成「二進位格式」？

理解這些後，你就能完全掌握憑證的「外在形式」和「內在結構」之間的關係。

## 從問題開始：如何在電腦間傳遞結構化資料？

### 🏗️ 生活中的類比：傢俱的包裝

想像你在 IKEA 買了一張桌子：

**方法 1：組裝好直接搬**
- ❌ 佔空間
- ❌ 運輸困難
- ❌ 容易損壞

**方法 2：拆解打包**
- ✅ 省空間
- ✅ 運輸方便
- ✅ 有標準的組裝說明書

**重點：需要一套「拆解」和「組裝」的標準規則**

### 💻 數位世界的挑戰

當兩台電腦要交換資料（例如：憑證）時：

**問題 1：資料結構不同**
```
電腦 A（Python）：
cert = {
    'version': 3,
    'subject': 'Apple Inc.',
    'publicKey': bytes([0x04, 0xa1, ...])
}

電腦 B（Java）：
Certificate cert = new Certificate();
cert.setVersion(3);
cert.setSubject("Apple Inc.");
```

資料結構不一樣！如何確保雙方理解相同的資料？

**問題 2：資料型別不同**
- 數字怎麼表示？（int? long? BigInteger?）
- 字串用什麼編碼？（UTF-8? ASCII?）
- 日期時間怎麼表示？

**這就是為什麼需要 ASN.1！**

## ASN.1 是什麼？

### 📜 正式定義

**ASN.1** = **A**bstract **S**yntax **N**otation **O**ne（抽象語法標記法一號）

- 1984 年由 ITU-T 和 ISO 聯合制定
- 一種「描述資料結構」的語言
- 獨立於程式語言和硬體平台

### 🎯 ASN.1 的作用

**ASN.1 做兩件事：**

1. **定義資料結構**（就像建築藍圖）
2. **編碼規則**（如何把資料打包成二進位）

### 📝 實際例子：Person 結構

**用 ASN.1 定義：**
```asn1
Person ::= SEQUENCE {
    name    UTF8String,
    age     INTEGER,
    email   UTF8String OPTIONAL
}
```

**這個定義說：**
- Person 是一個序列（SEQUENCE）
- 包含名字（UTF8 字串）
- 包含年齡（整數）
- 可選的 email（UTF8 字串）

**好處：**
- Python、Java、C++ 看到同樣的定義
- 都知道如何解析這個結構
- 不會產生誤解

## ASN.1 的基本類型

### 🧱 Universal Types（通用型別）

ASN.1 定義了一些基本的資料型別：

| ASN.1 類型 | Tag | 說明 | 例子 |
|-----------|-----|------|------|
| **BOOLEAN** | 0x01 | 布林值 | TRUE / FALSE |
| **INTEGER** | 0x02 | 整數 | 42, -100, 999999 |
| **BIT STRING** | 0x03 | 位元字串 | 01011010 |
| **OCTET STRING** | 0x04 | 位元組字串 | 二進位資料 |
| **NULL** | 0x05 | 空值 | NULL |
| **OBJECT IDENTIFIER** | 0x06 | 物件識別碼 | 1.2.840.10045.3.1.7 |
| **UTF8String** | 0x0C | UTF-8 字串 | "Apple Inc." |
| **SEQUENCE** | 0x30 | 序列（有序集合） | {a, b, c} |
| **SET** | 0x31 | 集合（無序） | {x, y, z} |

### 🏷️ Tag（標籤）的概念

每種類型都有一個「標籤」（Tag），就像商品的條碼：

```
Tag: 0x02（INTEGER）
     ↓
資料: 42
     ↓
編碼: 02 01 2a
      │  │  └─ 值：42 (0x2a)
      │  └──── 長度：1 byte
      └─────── 類型：INTEGER
```

### 🔢 Object Identifier (OID)

這是 ASN.1 中最重要的概念之一！

**OID 是什麼？**
- 全球唯一的「物件」識別碼
- 用點號分隔的數字序列
- 例如：`1.2.840.10045.3.1.7`

**層級結構：**
```
1 (ISO)
└── 2 (ISO member body)
    └── 840 (美國)
        └── 10045 (ANSI X9.62)
            └── 3 (id-publicKeyType)
                └── 1 (id-ecPublicKey)
                    └── 7 (secp256r1 / P-256)
```

**在憑證中的應用：**
```
OID 1.2.840.113635.100.8.2 = Apple App Attest Extension
    │   │   │      │   │  └─ 2 (包含 nonce 等 attestation 相關資料)
    │   │   │      │   └──── 8 (App Attest)
    │   │   │      └──────── 100 (Apple Extensions)
    │   │   └─────────────── 113635 (Apple Inc.)
    │   └─────────────────── 840 (美國)
    └─────────────────────── 2 (ISO member body)
```

## 編碼規則：DER vs BER

ASN.1 只定義「資料結構」，還需要「編碼規則」來打包成二進位。

### 🎁 BER（Basic Encoding Rules）

**特性：**
- 最基本的編碼規則
- **允許多種編碼方式**（同樣的資料可以有不同的編碼）
- 靈活但不唯一

**結構：**
```
┌─────────┬─────────┬─────────┐
│  Tag    │ Length  │  Value  │
│ (類型)   │ (長度)   │  (值)   │
└─────────┴─────────┴─────────┘
```

**例子：編碼數字 127**

方式 1：
```
02 01 7f
│  │  └─ 值：127
│  └──── 長度：1
└─────── Tag：INTEGER
```

方式 2（長格式長度）：
```
02 81 01 7f
│  │  │  └─ 值：127
│  │  └──── 長度：1
│  └─────── 長度的長度：1
└────────── Tag：INTEGER
```

**問題：** 同樣的資料，兩種編碼！

### 💎 DER（Distinguished Encoding Rules）

**特性：**
- BER 的子集
- **只有一種編碼方式**（確定性）
- 用於需要數位簽章的場合

**為什麼憑證用 DER？**

還記得第 10 篇提到的問題嗎？

```
CA 簽章憑證：
1. 對憑證內容計算 hash
2. 用私鑰簽章 hash

驗證時：
1. 重新計算憑證的 hash
2. 用公鑰驗證簽章
```

**如果編碼不唯一會怎樣？**

```
原始編碼：02 01 7f       → Hash: abc123
修改編碼：02 81 01 7f    → Hash: def456  （不同！）

簽章驗證失敗！（即使內容一樣）
```

**所以：**
- ✅ DER 確保「同樣的資料只有一種編碼」
- ✅ 簽章才能正確驗證
- ✅ X.509 憑證必須用 DER

### 📏 DER 的編碼規則

**長度編碼：**

**短格式（值 ≤ 127）：**
```
長度 = 42
編碼: 2a
```

**長格式（值 > 127）：**
```
長度 = 500 (0x01f4)
編碼: 82 01 f4
      │  └───┴─ 實際長度（2 bytes）
      └──────── 0x82 = 10000010
                       │     └─ 長度佔 2 bytes
                       └──────── 長格式標記
```

**SEQUENCE 編碼例子：**

```asn1
Person ::= SEQUENCE {
    name UTF8String,
    age  INTEGER
}

實際資料：
name = "Alice"
age = 30
```

**DER 編碼：**
```
30 0d                    -- SEQUENCE, 長度 13
   0c 05 41 6c 69 63 65  -- UTF8String "Alice"
   │  │  └────┬────────┘
   │  │       └─ "Alice" (ASCII)
   │  └────────── 長度：5
   └───────────── Tag：UTF8String (0x0C)

   02 01 1e              -- INTEGER 30
   │  │  └─ 值：30 (0x1E)
   │  └──── 長度：1
   └─────── Tag：INTEGER (0x02)
```

## PEM 格式：讓二進位變成文字

### 📧 為什麼需要 PEM？

**問題：**
- DER 是二進位格式
- Email 和一些協定只能傳輸文字
- 怎麼辦？

**解決方案：PEM**

**PEM** = **P**rivacy **E**nhanced **M**ail

### 🔄 PEM 的轉換過程

```
1. 原始資料（DER 二進位）
   30 82 02 cc 30 82 02 34...

2. Base64 編碼
   MIICzDCCAbSgAwIBAgIQGis8...

3. 加上標頭和標尾
   -----BEGIN CERTIFICATE-----
   MIICzDCCAbSgAwIBAgIQGis8...
   -----END CERTIFICATE-----
```

### 📝 PEM 格式詳解

**結構：**
```
-----BEGIN <TYPE>-----
<Base64 編碼的資料，每行 64 字元>
-----END <TYPE>-----
```

**常見的 TYPE：**
```
CERTIFICATE        → X.509 憑證
PRIVATE KEY        → 私鑰
PUBLIC KEY         → 公鑰
CERTIFICATE REQUEST → CSR (憑證簽章請求)
```

**實際例子：**
```
-----BEGIN CERTIFICATE-----
MIICzDCCAbSgAwIBAgIQGis8TTBAQEBAQEBAQEBAQEBAQDANBgkqhkiG9w0BAQUF
ADBOMQswCQYDVQQGEwJVUzEQMA4GA1UEChMHRXF1aWZheDEtMCsGA1UECxMkRXF1
aWZheCBTZWN1cmUgQ2VydGlmaWNhdGUgQXV0aG9yaXR5MB4XDTA5MDYxMjEwNTAy
...
-----END CERTIFICATE-----
```

## 三種格式的對比與轉換

### 📊 格式對比

| 特性 | BER | DER | PEM |
|------|-----|-----|-----|
| **編碼方式** | 二進位 | 二進位 | Base64 文字 |
| **唯一性** | ❌ 不唯一 | ✅ 唯一 | ✅ 唯一（基於 DER） |
| **用途** | 一般資料交換 | 數位簽章 | Email、文字傳輸 |
| **檔案副檔名** | .ber | .der | .pem, .crt, .cer |
| **可讀性** | ❌ 二進位 | ❌ 二進位 | ✅ 文字（但仍需解碼） |
| **大小** | 小 | 小 | 大（+33%，Base64 的開銷） |

### 🔄 格式轉換

**DER → PEM：**
```bash
openssl x509 -in cert.der -inform DER -out cert.pem -outform PEM
```

**PEM → DER：**
```bash
openssl x509 -in cert.pem -inform PEM -out cert.der -outform DER
```

**程式化轉換（Python）：**
```python
import base64

# DER → PEM
def der_to_pem(der_bytes, label="CERTIFICATE"):
    b64 = base64.b64encode(der_bytes).decode('ascii')
    # 每 64 字元換行
    pem_lines = [b64[i:i+64] for i in range(0, len(b64), 64)]
    pem = f"-----BEGIN {label}-----\n"
    pem += '\n'.join(pem_lines)
    pem += f"\n-----END {label}-----\n"
    return pem

# PEM → DER
def pem_to_der(pem_string):
    # 移除標頭、標尾、換行
    lines = pem_string.strip().split('\n')
    b64 = ''.join([line for line in lines if not line.startswith('-----')])
    return base64.b64decode(b64)
```

## 實際應用：憑證的不同表示

### 🔍 同一張憑證的三種面貌

**1. DER 格式（二進位）：**
```
30 82 02 cc 30 82 02 34 a0 03 02 01 02 02 10 1a...
```
- 最緊湊
- 電腦直接處理
- 用於實際傳輸（App Attest 的 x5c）

**2. PEM 格式（文字）：**
```
-----BEGIN CERTIFICATE-----
MIICzDCCAbSgAwIBAgIQGis8...
-----END CERTIFICATE-----
```
- 可以複製貼上
- Email 友善
- 設定檔常用

**3. 文字解析（人類可讀）：**
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: ...
        Issuer: CN=Apple App Attestation CA
        Subject: CN=6d2ac4845f13...
        Public Key: EC Public Key (256 bit)
```
- 用 OpenSSL 或線上工具產生
- 除錯用
- 不是標準格式

### 📂 檔案副檔名的混亂

**注意：副檔名不可靠！**

| 副檔名 | 可能的格式 |
|--------|-----------|
| .pem | PEM（通常） |
| .der | DER（通常） |
| .crt | PEM 或 DER（都可能） |
| .cer | PEM 或 DER（都可能） |
| .key | PEM 或 DER（都可能） |

**判斷方式：**
```python
def detect_format(data):
    if data.startswith(b'-----BEGIN'):
        return 'PEM'
    elif data[0] == 0x30:  # SEQUENCE tag
        return 'DER (可能是 BER)'
    else:
        return '未知格式'
```

## 為什麼密碼學系統選擇 ASN.1？

### ✅ 優點

**1. 標準化**
- 1984 年就有的國際標準
- 所有系統都支援
- 不會有相容性問題

**2. 類型安全**
- 明確定義每個欄位的類型
- 不會把字串當數字解析

**3. 擴展性**
- OID 系統提供無限擴展可能
- 新增欄位不會破壞舊版本

**4. 效率**
- DER 編碼緊湊
- 二進位格式，解析快

### ❌ 缺點

**1. 學習曲線陡**
- 概念抽象
- 文件難懂
- 新手不友善

**2. 除錯困難**
- 二進位格式看不懂
- 需要專門工具

**3. 靈活性不足**
- 結構一旦定義就難更改
- 不像 JSON 那樣彈性

### 🆚 與現代格式的對比

| 特性 | ASN.1/DER | JSON | CBOR |
|------|-----------|------|------|
| **標準化時間** | 1984 | 2001 | 2013 |
| **可讀性** | ❌ 低 | ✅ 高 | ❌ 低 |
| **體積** | 小 | 大 | 很小 |
| **類型安全** | ✅ 強 | ❌ 弱 | ✅ 強 |
| **擴展性** | ✅ OID | ✅ 彈性 | ✅ 彈性 |
| **學習曲線** | 陡 | 平緩 | 中等 |
| **適用場景** | 密碼學、電信 | Web API | IoT、密碼學 |

## 本文小結：理解編碼的重要性

✅ **ASN.1**：定義資料結構的標準語言  
✅ **DER**：唯一確定的編碼方式（用於簽章）  
✅ **BER**：靈活的編碼方式（允許多種編碼）  
✅ **PEM**：DER 的 Base64 文字版本（方便傳輸）  
✅ **OID**：全球唯一的物件識別系統

### 🎓 與前面文章的連結

**第 9 篇（CBOR）→ 現代的編碼方式**
- CBOR 和 DER 都是二進位編碼
- CBOR 更簡單、更現代
- DER 更古老、更複雜

**第 10 篇（X.509）→ 為什麼憑證用 DER**
- 憑證需要數位簽章
- 簽章需要唯一編碼
- 所以選擇 DER 而不是 BER

### 📊 編碼方式的演進

```
1984: ASN.1/DER → 密碼學標準（X.509、PKCS）
      ↓
2001: JSON      → Web API 革命（簡單、可讀）
      ↓
2013: CBOR      → 結合優點（二進位 + 簡單）
```

**為什麼還在用 ASN.1/DER？**
- 歷史遺留（X.509 憑證從 1988 年就用）
- 標準成熟（所有系統都支援）
- 更換成本高（整個 PKI 體系都要改）

## 下一步學習

下一篇《第 12 篇：X9.62 橢圓曲線公鑰格式》會介紹：
- ECC 公鑰如何表示成二進位（X9.62 格式）
- 未壓縮格式（0x04 || X || Y）與壓縮格式
- 如何將 X9.62 raw point 包裝成 SPKI 格式

這會連結系列二的 ECC 數學知識與實際的公鑰格式！

---

## 💡 補充資料

### 常見問題

**Q: 我需要深入學習 ASN.1 嗎？**
A: 不用！理解基本概念就夠了。實際開發用現成函式庫（OpenSSL、cryptography）就好。

**Q: 如何判斷檔案是 DER 還是 PEM？**
A:
- PEM：文字檔，開頭是 `-----BEGIN`
- DER：二進位檔，開頭通常是 `0x30`（SEQUENCE）

**Q: 為什麼叫「抽象」語法？**
A: 因為 ASN.1 只定義「資料結構」，不管實際如何編碼。編碼由 DER/BER 等規則決定。

**Q: OID 會用完嗎？**
A: 不會！OID 是階層式的，每個組織可以在自己的分支下無限分配。

### 實用工具

**線上 ASN.1 解析器：**
- https://lapo.it/asn1js/ - 視覺化 ASN.1 結構
- https://certlogik.com/decoder/ - 憑證解碼器

**命令列工具：**
```bash
# 查看 DER 檔案結構
openssl asn1parse -in file.der -inform DER

# 查看 PEM 檔案結構
openssl asn1parse -in file.pem -inform PEM

# 查看憑證詳細資訊
openssl x509 -in cert.pem -text -noout
```

**Python 函式庫：**
- `pyasn1` - ASN.1 編碼/解碼
- `cryptography` - 密碼學操作（內建 ASN.1 支援）

### 進階閱讀（選讀）

- [ITU-T X.680 - ASN.1 規格](https://www.itu.int/rec/T-REC-X.680/)
- [ITU-T X.690 - DER/BER 編碼規則](https://www.itu.int/rec/T-REC-X.690/)
- [RFC 5280 - X.509 in ASN.1](https://tools.ietf.org/html/rfc5280)
- [A Layman's Guide to ASN.1, BER, and DER](http://luca.ntop.org/Teaching/Appunti/asn1.html)
