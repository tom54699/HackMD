# 第 14 篇：JWT 與 JWS 格式解析 - 呼叫 API 的通行證

## 前言：為什麼需要學 JWT？

在實作 App Attest 或 Play Integrity 驗證時，你會遇到一個共同的問題：**如何呼叫 Apple 或 Google 的 API？**

### 🔑 實際場景

**驗證 App Store Receipt：**
```
你的伺服器：「Apple，請幫我驗證這個 receipt」
Apple API：「你是誰？證明你有權限呼叫這個 API」
你的伺服器：「這是我的 JWT token（auth token）」
Apple API：「驗證通過，這是 receipt 的內容」
```

**呼叫 Google API：**
```
類似流程，也需要 JWT 當作 auth token
```

### ❓ 問題來了

**你需要產生 JWT token，但：**
- JWT 是什麼格式？
- 為什麼是這種格式？
- 怎麼產生 JWT？

**這就是本文要解答的問題。**

目的很單純：讓你理解 JWT 的格式，知道如何產生一個合法的 JWT token，才能順利呼叫 Apple/Google 的 API。

## JWT 是什麼？

### 📜 正式定義

**JWT** = **J**SON **W**eb **T**oken

- 一種開放標準（RFC 7519）
- 用 JSON 格式來表示「聲明」（claims）
- 可以被數位簽章或加密
- 主要用於身份驗證和資訊交換

### 🎫 生活類比：電影票

想像 JWT 就像一張電影票：

```
電影票
──────────────────
電影：復仇者聯盟
場次：2024/10/02 19:30
座位：A15
票價：$350
──────────────────
[防偽浮水印] [QR Code]
```

**電影票的特點：**
- ✅ 自己帶著所有資訊（電影名稱、時間、座位）
- ✅ 有防偽機制（浮水印、QR Code）
- ✅ 一看就懂（不需要查詢系統）

**JWT 的特點：**
- ✅ 自己帶著所有聲明（user ID、權限、過期時間）
- ✅ 有數位簽章（防止竄改）
- ✅ 可以解碼查看（Base64 編碼，不是加密）

## JWT 的結構

### 🧩 三個部分

JWT 由三個部分組成，用點號 `.` 連接：

```
Header.Payload.Signature
```

**實際的 JWT：**
```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.
eyJpc3MiOiJjb20uZXhhbXBsZS5hcHAiLCJleHAiOjE3MDk4NzY1NDN9.
MEUCIQDxF8H3fK9b_L2vX1pN8wQ7R5tY6zM3nJ4kP9sD2fG1wIgE7vH9Kq
```

看起來像亂碼？別擔心，這只是 **Base64URL 編碼**，不是加密！

### 📦 Part 1: Header（標頭）

**用途：** 說明這個 token 的類型和簽章演算法

**編碼前（JSON）：**
```json
{
  "typ": "JWT",
  "alg": "ES256"
}
```

**編碼後（Base64URL）：**
```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9
```

**欄位說明：**

**typ (Type)**：
- 固定值：`"JWT"`
- 表示這是一個 JWT token

**alg (Algorithm)**：
- 說明用什麼演算法簽章
- 常見值：
  - `ES256`：ECDSA + SHA-256（Apple/Google 常用）
  - `RS256`：RSA + SHA-256（傳統方式）
  - `HS256`：HMAC + SHA-256（需妥善金鑰管理，不適合跨實體驗證）

### 📝 Part 2: Payload（載荷）

**用途：** 實際要傳遞的資料（聲明）

**編碼前（JSON）：**
```json
{
  "iss": "com.example.myapp",
  "iat": 1709876543,
  "exp": 1709880143,
  "aud": "appstoreconnect-v1"
}
```

**編碼後（Base64URL）：**
```
eyJpc3MiOiJjb20uZXhhbXBsZS5hcHAiLCJleHAiOjE3MDk4NzY1NDN9
```

**標準欄位（Registered Claims）：**

| 欄位 | 全名 | 說明 | 範例 |
|------|------|------|------|
| `iss` | Issuer | 發行者（誰發的） | `"com.example.app"` |
| `sub` | Subject | 主題（關於誰） | `"user123"` |
| `aud` | Audience | 接收者（給誰用） | `"appstoreconnect-v1"` |
| `exp` | Expiration | 過期時間（Unix timestamp） | `1709880143` |
| `iat` | Issued At | 發行時間（Unix timestamp） | `1709876543` |
| `nbf` | Not Before | 生效時間（Unix timestamp） | `1709876543` |

**為什麼需要這些欄位？**

**iss (Issuer)**：
```
用途：標明是誰發的這個 token
例如：你的 App Bundle ID
Apple 會檢查：這個 token 是你發的嗎？
```

**exp (Expiration)**：
```
用途：限制 token 的有效時間
例如：10 分鐘後過期
Apple 會檢查：這個 token 過期了嗎？
```

**aud (Audience)**：
```
用途：說明這個 token 是給誰用的
例如：給 "appstoreconnect-v1" 用
Apple 會檢查：這個 token 是給我的嗎？
```

### ✍️ Part 3: Signature（簽章）

**用途：** 證明 token 沒有被竄改

**如何產生簽章：**
```
1. 把 Header 和 Payload 用點號連接
   data = base64url(header) + "." + base64url(payload)

2. 用私鑰簽章這段資料
   signature = sign(data, private_key, algorithm)

3. 把簽章也做 Base64URL 編碼
   encoded_signature = base64url(signature)
```

**完整的 JWT：**
```
base64url(header) + "." + base64url(payload) + "." + base64url(signature)
```

**為什麼這樣設計？**

```
駭客想竄改 Payload：
1. 解碼 Payload（很簡單，Base64 而已）
2. 修改內容（例如：把過期時間延長）
3. 重新編碼 Payload

但是：
4. 沒有私鑰，無法產生正確的簽章
5. Apple 驗證簽章時會失敗 ❌
```

## Base64URL 編碼：為什麼不是 Base64？

### 🔤 Base64 vs Base64URL

**問題：** 標準 Base64 會產生 `+`、`/`、`=` 這些字元

```
標準 Base64：
SGVsbG8gV29ybGQh+/==

問題：
├── + 在 URL 中有特殊意義（空格）
├── / 在 URL 中是路徑分隔符
└── = 在 URL 中是參數分隔符
```

**解決方案：Base64URL**

```
替換規則：
+ → -（減號）
/ → _（底線）
= → 移除（不要 padding）

結果：
SGVsbG8gV29ybGQh-_
```

**為什麼重要？**

JWT 常常放在 URL 或 HTTP Header 中：
```
Authorization: Bearer eyJhbGc...（JWT token）
```

使用 Base64URL 確保：
- ✅ 可以安全地放在 URL 中
- ✅ 可以安全地放在 HTTP Header 中
- ✅ 不會被誤解為特殊字元

## JWS：有簽章的 JWT

### 📋 什麼是 JWS？

**JWS** = **J**SON **W**eb **S**ignature

- JWT 只是定義格式
- JWS 定義如何簽章

**關係：**
```
JWT：定義「長什麼樣子」
JWS：定義「如何簽章」

實務上：
大部分人說的 JWT 其實是 JWS（有簽章的 JWT）
```

### 🔐 簽章演算法

#### ES256（ECDSA + SHA-256）⭐ 推薦

**特性：**
- 使用橢圓曲線密碼學（第3篇學過的 ECDSA）
- 公鑰小、簽章小、速度快
- Apple 和 Google 都使用

**金鑰：**
```
私鑰：用來簽章（只有你有）
公鑰：用來驗證（可以公開）
```

**產生流程：**
```
1. 準備資料
   data = base64url(header) + "." + base64url(payload)

2. 計算 hash
   hash = SHA256(data)

3. 用私鑰簽章
   signature = ECDSA_sign(hash, private_key)

4. Base64URL 編碼
   encoded = base64url(signature)
```

#### RS256（RSA + SHA-256）

**特性：**
- 使用 RSA 演算法
- 相容性好（老系統都支援）
- 金鑰較大、簽章較大

#### HS256（HMAC + SHA-256）⚠️ 不適合跨實體驗證場景

**特性：**
- 使用共享密鑰（symmetric key）
- 發送方和接收方都有相同的密鑰
- 無法提供「不可否認性」

**為什麼不適合 API 認證？**
```
問題：
├── 伺服器需要知道你的密鑰
├── 如果密鑰洩漏，伺服器也能偽造你的 token
└── 無法證明「確實是你發的」

適用場景：
└── 自己的系統內部使用（前後端共享密鑰）
```

## 產生 JWT 的基本流程

### 📝 步驟總覽

```
步驟 1：準備私鑰
└── 從 Apple Developer 下載 .p8 私鑰檔案

步驟 2：建立 Header
└── { "alg": "ES256", "typ": "JWT" }

步驟 3：建立 Payload
└── { "iss": "你的 issuer ID", "exp": 過期時間, ... }

步驟 4：簽章
└── 用私鑰簽章 Header 和 Payload

步驟 5：組合
└── header.payload.signature
```

### 🔑 關鍵：私鑰格式

Apple 給你的私鑰是 **PKCS#8 PEM 格式**：

```
-----BEGIN PRIVATE KEY-----
MIGTAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBHkwdwIBAQQg...
-----END PRIVATE KEY-----
```

**這是什麼？**
- PKCS#8：私鑰的標準格式
- PEM：Base64 編碼的文字格式（第8篇講過）
- 內容：EC 私鑰（P-256 曲線）

### 🛠️ 使用程式庫

**Python 範例（概念）：**
```python
import jwt
import time

# 讀取私鑰
with open('AuthKey_XXXXXXXXXX.p8', 'r') as f:
    private_key = f.read()

# 建立 payload
payload = {
    'iss': 'your-issuer-id',
    'iat': int(time.time()),
    'exp': int(time.time()) + 600,  # 10 分鐘後過期
    'aud': 'appstoreconnect-v1'
}

# 產生 JWT
token = jwt.encode(
    payload,
    private_key,
    algorithm='ES256',
    headers={'kid': 'your-key-id'}  # Key ID
)
```

**重點概念：**
- 使用現成函式庫（`PyJWT`、`jose`等）
- 不需要手動處理 Base64URL、簽章等細節
- 理解格式比實作細節重要

## 驗證 JWT 的流程

### ✅ 接收方如何驗證？

```
收到 JWT：header.payload.signature

步驟 1：分離三個部分
├── header_b64 = "eyJhbGc..."
├── payload_b64 = "eyJpc3M..."
└── signature_b64 = "MEUCIQDx..."

步驟 2：解碼 Header
├── 得知使用的演算法（例如：ES256）
└── 得知 Key ID（如果有）

步驟 3：解碼 Payload
├── 檢查 exp（過期時間）
├── 檢查 iss（發行者）
└── 檢查 aud（接收者）

步驟 4：驗證簽章
├── 重新計算：data = header_b64 + "." + payload_b64
├── 用公鑰驗證簽章
└── 驗證通過 ✅ 或失敗 ❌
```

### 🔍 常見的驗證失敗原因

**1. 簽章驗證失敗**
```
原因：
├── Payload 被竄改
├── 使用錯誤的私鑰簽章
└── 使用錯誤的公鑰驗證
```

**2. Token 過期**
```
原因：
├── exp 時間已過
└── 需要重新產生 token
```

**3. Issuer 不符**
```
原因：
├── iss 欄位不是預期的值
└── 可能是錯誤的 token
```

## JWT 與其他格式的對比

### 📊 為什麼 Apple/Google API 選擇 JWT？

| 特性 | JWT | X.509 憑證 | CBOR |
|------|-----|-----------|------|
| **可讀性** | ✅ 高（Base64 解碼即可） | ❌ 低（需要工具） | ❌ 低（二進位） |
| **自包含** | ✅ 帶著所有資訊 | ✅ 帶著公鑰和身份 | ✅ 帶著所有資訊 |
| **體積** | 中等 | 大 | 小 |
| **標準化** | ✅ RFC 7519 | ✅ X.509 | ✅ RFC 8949 |
| **Web 友善** | ✅ 很友善 | ⚠️ 需要轉換 | ❌ 需要轉換 |
| **適用場景** | API 認證、會話管理 | 身份憑證、信任鏈 | 資料傳輸 |

**為什麼 API 認證用 JWT？**

1. **簡單易用**：
   - JSON 格式，開發者熟悉
   - 不需要複雜的解析工具

2. **自包含**：
   - Token 本身帶著所有資訊
   - 不需要查詢資料庫

3. **跨平台**：
   - 所有程式語言都有 JWT 函式庫
   - 不需要處理 ASN.1 編碼

4. **HTTP 友善**：
   - 可以放在 Header、URL、Cookie
   - Base64URL 確保相容性

### 🔄 實際使用場景對比

**App Attest（裝置認證）：**
```
格式：CBOR + X.509
原因：
├── 體積小（行動網路友善）
├── 安全性高（標準密碼學格式）
└── 包含完整憑證鏈
```

**API 認證（Apple/Google）：**
```
格式：JWT
原因：
├── Web 標準
├── 容易實作
└── 自包含（不需要額外查詢）
```

## 本文小結

✅ **JWT 是什麼**：JSON Web Token，用 JSON + Base64URL + 簽章的格式  
✅ **三個部分**：Header（類型和演算法）+ Payload（資料）+ Signature（簽章）  
✅ **為什麼用 Base64URL**：確保可以安全放在 URL 和 HTTP Header  
✅ **JWS**：有簽章的 JWT，實務上最常用  
✅ **用途**：呼叫 Apple/Google API 時的 auth token

### 🎓 與前面文章的連結

**系列二第6篇（ECDSA）→ JWT 的簽章演算法**
- JWT 的 ES256 就是使用 ECDSA
- 還記得橢圓曲線和數位簽章嗎？JWT 在用同樣的技術

**第10篇（X.509）→ 不同的應用場景**
- X.509：用於身份憑證和信任鏈
- JWT：用於 API 認證和會話管理
- 兩者解決不同的問題

**第11篇（ASN.1）→ 不同的編碼方式**
- X.509 用 ASN.1 DER（二進位）
- JWT 用 JSON + Base64URL（文字）
- JWT 更簡單、更 Web 友善

### 📊 格式選擇的考量

```
選擇 X.509 憑證（ASN.1 DER）：
├── 需要建立信任鏈
├── 需要長期有效的身份證明
└── 密碼學標準要求

選擇 JWT：
├── API 認證
├── 短期會話管理
└── Web/HTTP 友善
```

## 下一篇預告

第15篇《WebAuthn與裝置信任模型》會探討現代身份驗證的基礎協定，而第16篇《Apple App Attest 資料格式實例解析》會整合所有學到的格式知識：
- 如何解析完整的 attestation object？
- CBOR、X.509、ASN.1、PKCS#7 如何協同運作？
- 實際開發時如何處理這些格式？

你會發現，Apple 在不同場景使用不同的格式：
- App Attest：CBOR + X.509
- API 認證：JWT
- Receipt：PKCS#7

每種格式都有它的設計考量！

---

## 💡 補充資料

### 常見問題

**Q: JWT 是加密的嗎？**
A: 不是！JWT 只是 Base64URL 編碼，任何人都能解碼看到內容。簽章只是確保沒有被竄改，不是加密。

**Q: JWT 可以存敏感資訊嗎？**
A: 不建議！因為內容可以被解碼。只放不敏感但需要驗證的資訊（例如 user ID、權限）。

**Q: 為什麼不用對稱金鑰（HS256）？**
A: 對稱金鑰雙方都有相同密鑰，伺服器也能偽造你的 token。非對稱金鑰（ES256/RS256）只有你能簽章，更安全。

**Q: Token 過期了怎麼辦？**
A: 重新產生一個新的 token。這是設計上的安全機制，限制 token 的有效時間。

### 實用資源

**線上工具：**
- [jwt.io](https://jwt.io) - JWT 解碼和驗證工具

**程式庫：**
- **Python**: `PyJWT`, `python-jose`
- **Node.js**: `jsonwebtoken`, `jose`
- **PHP**: `firebase/php-jwt`
- **Java**: `jjwt`

**官方規格：**
- [RFC 7519 - JWT](https://tools.ietf.org/html/rfc7519)
- [RFC 7515 - JWS](https://tools.ietf.org/html/rfc7515)

### Apple API 相關文件

- [App Store Connect API - Authentication](https://developer.apple.com/documentation/appstoreconnectapi/generating_tokens_for_api_requests)
- [App Store Server API](https://developer.apple.com/documentation/appstoreserverapi)
