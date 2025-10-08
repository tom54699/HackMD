# 第 9 篇：資料序列化與 CBOR - 為什麼 Apple 不用 JSON？

## 前言：當你第一次看到 App Attest 的回應資料

經過前面五篇文章，我們已經理解了密碼學的核心概念：
- **ECDSA** 讓 iPhone 可以產生數位簽章
- **信任鏈** 讓我們相信公鑰的真實性
- **數位簽章** 在各種系統中的應用

現在有個實際問題：當 iPhone 要把這些密碼學資料（公鑰、簽章、憑證）傳給伺服器時，要用什麼格式包裝？

你可能會想：「用 JSON 不就好了嗎？」但當你看到 Apple App Attest 的回應時，會發現它用的是一種叫 **CBOR** 的格式。為什麼？

## 從生活例子理解：資料要怎麼「打包」？

### 📦 寄包裹的類比

想像你要寄一個包裹：

**方法一：用紙箱包裝（像 JSON）**
- ✅ **看得懂**：打開就知道裡面是什麼
- ✅ **好處理**：任何人都會用紙箱
- ❌ **占空間**：紙箱本身很大
- ❌ **不適合特殊物品**：易碎品、液體需要特殊處理

**方法二：用真空包裝（像 CBOR）**
- ✅ **省空間**：壓縮後體積小很多
- ✅ **保護性好**：適合精密物品
- ❌ **打開麻煩**：需要專門工具
- ❌ **看不懂**：外觀看不出內容

### 💻 數位世界的「打包」問題

在數位世界裡，把資料打包起來的過程叫做「**序列化**」。

**為什麼需要序列化？**

```
你的 iPhone 記憶體裡：
├── 公鑰（一堆數字）
├── 簽章（一堆數字）
└── 其他資訊（文字、數字）

要傳給伺服器 → 需要「打包」成一串資料 → 伺服器「解包」還原
```

就像寄包裹一樣，你需要：
1. **打包**：把東西裝進箱子
2. **運送**：透過網路傳輸
3. **解包**：對方收到後拆開

## 常見的「打包」方式比較

### 📄 JSON：最常見的格式

**就像用透明塑膠袋包裝，看得清楚**

```json
{
  "name": "John",
  "age": 30,
  "publicKey": "LS0tLS1CRUdJTi..."
}
```

**優點：**
- ✅ 人類可讀：打開就看得懂
- ✅ 到處都支援：所有程式語言都會處理
- ✅ 容易除錯：出問題可以直接看

**缺點：**
- ❌ 體積大：很多重複的引號、括號
- ❌ 二進位資料麻煩：需要先轉成文字（Base64）
- ❌ 傳輸慢：資料大，傳得久

**實際例子：**
```
原始資料：32 bytes 的二進位資料
JSON 表示：需要約 50 bytes（因為要用 Base64 編碼）
```

### 🎁 CBOR：Apple 的選擇

**就像用真空壓縮袋，體積小、保護好**

**優點：**
- ✅ 體積小：同樣的資料，比 JSON 小 20-40%
- ✅ 二進位友善：可以直接放入二進位資料
- ✅ 解析快：電腦處理更快
- ✅ 適合手機：省電、省流量

**缺點：**
- ❌ 人類不可讀：看起來像亂碼
- ❌ 除錯麻煩：需要專門工具

**實際例子：**
```
原始資料：32 bytes 的二進位資料
CBOR 表示：約 35 bytes（直接包裝，幾乎沒有額外負擔）
JSON 表示：約 50 bytes（需要 Base64）

省下約 30% 的空間！
```

## 為什麼 Apple App Attest 選擇 CBOR？

### 🎯 三個關鍵原因

#### 1. 省空間、省流量

App Attest 的回應包含：
- 公鑰（64 bytes）
- 數位憑證（可能幾 KB）
- 簽章（64 bytes）
- 其他資料

**如果用 JSON：**
- 所有二進位資料都要 Base64 編碼
- 增加 33% 體積
- 手機多用流量、多耗電

**如果用 CBOR：**
- 直接包裝二進位資料
- 體積最小
- 省電省流量

#### 2. 更適合密碼學資料

密碼學資料特點：
- ✅ 大部分是二進位（公鑰、簽章、憑證）
- ✅ 精確性要求高（一個 bit 錯都不行）
- ✅ 不需要人類閱讀

CBOR 完全符合這些需求！

#### 3. 業界標準

CBOR 不是 Apple 發明的，而是 **IETF RFC 標準**（就像 HTTP、JSON 一樣）。

使用 CBOR 的系統：
- 🍎 Apple App Attest
- 🌐 WebAuthn（網站登入標準）
- 🔐 FIDO2（身份驗證標準）

## CBOR 基本概念：簡單的編碼結構

### 🏗️ CBOR 的結構：類型 + 資料

CBOR 的核心設計很簡單：**每個資料都帶著自己的「標籤」**

```
CBOR 編碼結構：

┌─────────┬─────────┐
│  類型   │  資料   │
│ (Tag)   │ (Value) │
└─────────┴─────────┘

就像貨物標籤：
📦 「易碎品」標籤 → 裡面是玻璃杯
📦 「冷藏」標籤 → 裡面是食物
📦 「數字」標籤 → 裡面是整數
```

### 🔢 CBOR 的基本類型

| 類型 | Tag 開頭 | 說明 | 例子 |
|------|---------|------|------|
| **整數** | 0x00-0x1B | 正整數 | `42` |
| **文字** | 0x60-0x7B | UTF-8 字串 | `"hello"` |
| **二進位** | 0x40-0x5B | 位元組串 | 公鑰、簽章 |
| **陣列** | 0x80-0x9B | 有序列表 | `[1, 2, 3]` |
| **Map** | 0xA0-0xBB | 鍵值對 | `{"name": "John"}` |

### 📝 簡單範例：編碼一個數字

**編碼數字 42：**
```
原始資料：42
CBOR 編碼：18 2a
           │  └─ 數值：42（十六進位 0x2a）
           └──── 類型：正整數

只需要 2 bytes！
```

**編碼字串 "hello"：**
```
原始資料："hello"
CBOR 編碼：65 68 65 6c 6c 6f
           │  └─────┬────────┘
           │        └─ "hello" 的 UTF-8 編碼
           └──────── 類型：文字（長度 5）

需要 6 bytes（1 byte 類型 + 5 bytes 資料）
```

**編碼 Map（字典）：**
```
原始資料：{"age": 30}
CBOR 編碼：a1 63 61 67 65 18 1e
           │  │  └───┬────┘ │  └─ 數值：30
           │  │      │      └──── 類型：整數
           │  │      └─────────── "age" 的 UTF-8
           │  └────────────────── 類型：文字（長度 3）
           └───────────────────── 類型：Map（1 個鍵值對）

需要 7 bytes
```

### 💡 為什麼這樣設計？

**對比 JSON：**
```
JSON: {"age": 30}
需要：12 bytes（包含引號、冒號、括號）

CBOR: a1 63 61 67 65 18 1e
需要：7 bytes

省下 42% 空間！
```

**關鍵差異：**
- JSON：需要很多「裝飾」字元（`{`, `}`, `"`, `:` 等）
- CBOR：用一個 byte 的標籤取代所有裝飾

### 🎁 CBOR 適合什麼資料？

```
適合：
├── 二進位資料（公鑰、簽章、憑證）
│   └── 直接放入，不需要 Base64
├── 結構化資料（有層次的資訊）
│   └── Map、Array 可以嵌套
└── 需要省空間的場景
    └── 行動裝置、IoT

不適合：
└── 需要人類直接閱讀的資料
    └── 用 JSON 更好
```

### 📊 JSON vs CBOR 實際對比

**場景：iPhone 傳送公鑰給伺服器**

**使用 JSON：**
```json
{
  "publicKey": {
    "x": "LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0=",
    "y": "TUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FR"
  }
}
```
- 大小：約 120 bytes
- 可讀性：✅ 人類可讀
- 效率：❌ Base64 編碼增加 33% 體積

**使用 CBOR：**
```
a1 66 70 75 62 6c 69 63 4b 65 79 a2 61 78 58 20 [32 bytes 原始資料] 61 79 58 20 [32 bytes 原始資料]
```
- 大小：約 80 bytes
- 可讀性：❌ 人類不可讀
- 效率：✅ 直接儲存，省 40 bytes

**省下 33% 的空間！**

## App Attest 中的 CBOR：實際應用

### 🍎 Attestation Object 的結構

當 iPhone 完成 attestation，會回傳一個 CBOR 物件：

```
Attestation Object（CBOR 格式）- 三個主要欄位
├── fmt（格式）: "apple-appattest"
├── attStmt（證明聲明）
│   ├── x5c: [憑證1, 憑證2]（都是二進位）
│   └── receipt: App Store 收據（二進位）
└── authData（認證資料）: 二進位資料（包含公鑰、簽章計數等）
```

**實際解碼後的樣子：**
```javascript
{
  fmt: 'apple-appattest',
  attStmt: {
    x5c: [
      <Buffer 30 82 02 cc ...>,  // 憑證1（二進位）
      <Buffer 30 82 02 36 ...>   // 憑證2（二進位）
    ],
    receipt: <Buffer 30 80 06 09 ...>  // App Store 收據（二進位）
  },
  authData: <Buffer 21 c9 9e 00 ...>  // 認證資料（二進位）
}
```

**為什麼用 CBOR？**
- ✅ 憑證是二進位（DER 格式）→ CBOR 直接放
- ✅ 收據是二進位 → CBOR 直接放
- ✅ authData 包含公鑰等資料，也是二進位 → CBOR 直接放

**如果用 JSON 會怎樣？**
```json
{
  "fmt": "apple-appattest",
  "attStmt": {
    "x5c": [
      "MIICzDCCAbSgAwIBAgI...",  // Base64 編碼，變長了！
      "MIIC2jCCAcKgAwIBAgI..."   // Base64 編碼，變長了！
    ],
    "receipt": "MIAGCSqGSIb3DQE..."  // Base64 編碼，變長了！
  },
  "authData": "IcmeAEgk8y2+6..."  // Base64 編碼，變長了！
}
```
所有二進位資料都要 Base64 編碼，體積增加 33%！

### 🔍 你不需要手動解析 CBOR

**重點：你通常不需要自己寫 CBOR 解析器！**

各種程式語言都有現成的函式庫：

**Python：**
```python
import cbor2

# 解碼 CBOR
data = cbor2.loads(cbor_bytes)
print(data['fmt'])  # 輸出：apple-appattest
```

**PHP：**
```php
use CBOR\Decoder;
use CBOR\StringStream;

$decoder = new Decoder();
// 解析 CBOR
$parsed = $decoder->decode(new StringStream($cborBytes));
// 轉成 PHP array
$data = $parsed->normalize();
echo $data['fmt'];  // 輸出：apple-appattest
```

**JavaScript：**
```javascript
const cbor = require('cbor');

const data = cbor.decodeFirstSync(cborBuffer);
console.log(data.fmt);  // 輸出：apple-appattest
```

## 與其他格式的比較

### 📊 三種格式的應用場景

| 格式 | 適用場景 | 優勢 | Apple 生態應用 |
|------|---------|------|---------------|
| **JSON** | API 回應、設定檔 | 人類可讀、除錯容易 | 一般 API、設定 |
| **CBOR** | 密碼學資料、二進位資料 | 體積小、效率高 | **App Attest** |
| **XML** | 舊系統、文件格式 | 擴展性好、驗證嚴格 | 較少使用 |

### 🎯 什麼時候該用哪種格式？

**使用 JSON 的時機：**
- ✅ 一般的 Web API（大部分情況）
- ✅ 需要人類閱讀的設定檔
- ✅ 除錯方便很重要

**使用 CBOR 的時機：**
- ✅ 傳輸二進位資料（憑證、簽章）
- ✅ 行動裝置（省電、省流量）
- ✅ 密碼學應用（App Attest、WebAuthn）

## 實務開發建議

### 💡 你需要知道的重點

#### 1. 理解概念就夠了

你不需要：
- ❌ 記憶 CBOR 的每個位元組格式
- ❌ 手動計算 CBOR 編碼
- ❌ 深入研究 CBOR 規格

你只需要：
- ✅ 知道 CBOR 是什麼、為什麼用
- ✅ 會使用函式庫解碼 CBOR
- ✅ 理解 App Attest 用 CBOR 的原因

#### 2. 使用現成工具

**除錯 CBOR：**
- 線上工具：[cbor.me](http://cbor.me)
- 命令列工具：`cbor2diag`
- 程式庫自帶的除錯功能

**範例：除錯 CBOR 資料**
```python
import cbor2
import json

# 讀取 CBOR
with open('attestation.cbor', 'rb') as f:
    data = cbor2.load(f)

# 轉成 JSON 方便查看（二進位資料會變成 Base64）
print(json.dumps(data, indent=2, default=str))
```

#### 3. 常見錯誤處理

**問題：CBOR 解碼失敗**

```python
try:
    data = cbor2.loads(cbor_bytes)
except Exception as e:
    print(f"CBOR 解碼失敗：{e}")
    # 檢查資料是否完整
    print(f"資料長度：{len(cbor_bytes)} bytes")
    # 檢查前幾個位元組
    print(f"開頭：{cbor_bytes[:10].hex()}")
```

**常見原因：**
- ❌ 資料不完整（網路傳輸中斷）
- ❌ Base64 解碼錯誤（如果先經過 Base64）
- ❌ 格式不是 CBOR（可能是其他格式）

## 本文小結：資料格式的選擇哲學

✅ **JSON 的價值**：人類可讀、除錯容易、適合一般用途  
✅ **CBOR 的價值**：體積小、效率高、適合密碼學資料  
✅ **格式選擇**：根據使用場景選擇最適合的格式  
✅ **實務開發**：使用現成函式庫，理解概念比細節重要

### 🎓 與前面文章的連結

- **第 3 篇（ECDSA）**：公鑰和簽章都是二進位資料 → 用 CBOR 最適合
- **第 4 篇（憑證鏈）**：憑證也是二進位（DER 格式）→ 用 CBOR 直接包裝
- **第 5 篇（數位簽章應用）**：App Attest 的所有密碼學資料都用 CBOR 傳輸

## 下一步學習

下一篇《第10篇：X.509 憑證格式》會帶你理解：
- 憑證裡面裝了什麼？
- 為什麼憑證看起來像亂碼？
- App Attest 的憑證有什麼特別的地方？

你會發現，憑證本身也是一種「打包」格式，就像 CBOR 一樣！

---

## 💡 補充資料

### 常見問題

**Q: 我一定要學會 CBOR 嗎？**
A: 不用！使用現成函式庫就可以了。理解「為什麼用 CBOR」比「怎麼編碼 CBOR」更重要。

**Q: CBOR 比 JSON 好嗎？**
A: 沒有絕對好壞，看使用場景。JSON 適合人類閱讀，CBOR 適合機器處理。

**Q: 如果我想看 CBOR 的內容怎麼辦？**
A: 用程式庫解碼成 JSON 或 Python dict，就可以看懂了。

### 進階資源
- [CBOR 官方網站](https://cbor.io)
- [RFC 8949 - CBOR 規格](https://www.rfc-editor.org/rfc/rfc8949.html)（給有興趣的人）
- [cbor.me](http://cbor.me) - 線上 CBOR 除錯工具
