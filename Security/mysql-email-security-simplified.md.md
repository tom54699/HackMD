# MySQL Collation 陷阱：Email 大小寫、重音與忘記密碼的安全坑

> 本文靈感源自這篇研究：https://blog.voorivex.team/puny-code-0-click-account-takeover  
> 該文討論 Punycode/IDN 與字串正規化不一致如何造成 0-click Account Takeover。  
> 本文聚焦在許多系統容易忽略的：**MySQL collation 對大小寫與重音的處理**，以及如何正確處理 email 識別。

---

## 問題核心：MySQL 預設 Collation 不分大小寫、不分重音

以 MySQL 8.x 為例，常見預設是 `utf8mb4_0900_ai_ci`（MySQL 5.7 以前多為 `utf8mb4_general_ci`，實務上也可能被 DBA 修改過）：

- `ai` = Accent Insensitive（重音不敏感）：`e` 與 `é`/`è`/`ê` 等同基底字母的不同重音會被視為相等
- `ci` = Case Insensitive（大小寫不敏感）：`A` 與 `a` 視為相等

這代表：

```sql
-- 這些查詢在 utf8mb4_0900_ai_ci 下都會找到同一筆資料
SELECT * FROM users WHERE email = 'user@example.com';
SELECT * FROM users WHERE email = 'User@Example.COM';
SELECT * FROM users WHERE email = 'USER@EXAMPLE.COM';

-- 重音也不分（ai 的效果）
SELECT * FROM users WHERE email = 'user@example.com';
SELECT * FROM users WHERE email = 'usér@example.com';  -- 也會找到
```

### 如果真的要區分大小寫/重音

可以在查詢時強制使用 `COLLATE`：

```sql
-- 強制精準比對
SELECT * FROM users
WHERE email COLLATE utf8mb4_bin = 'user@example.com';
```

或改變欄位的 collation：

```sql
ALTER TABLE users
MODIFY email VARCHAR(320) COLLATE utf8mb4_bin;
```

**注意**：如果欄位 collation 與查詢強制的 collation 不一致，可能導致索引無法如預期使用（在大量資料下效能影響更明顯）。

**但這只是第一步**，單靠這個還不夠，因為還有 email 本身的規範化問題（IDN、大小寫規則等）。

---

## 實際風險場景

### 場景 1：同一個人註冊了兩次

```
星期一:用戶用 john.doe@gmail.com 註冊 → 成功,DB 存入 john.doe@gmail.com
星期三:同一個人用 John.Doe@Gmail.com 再註冊一次

如果你的系統:
- 應用層沒做檢查,直接 INSERT
- email 欄位用 utf8mb4_bin(區分大小寫)
- 沒有 UNIQUE 約束

→ 結果:同一個 Gmail 信箱變成兩個帳號(Gmail 不分大小寫,但你的 DB 分了)
```

### 場景 2：忘記密碼寄到攻擊者信箱

```
DB 裡有：user@example.com（正常用戶）
攻擊者輸入：usér@example.com（é 用了重音符號）去「忘記密碼」

系統處理：
1. 用 ai_ci 查詢 → 匹配到 user@example.com
2. 但寄信目標用攻擊者輸入的 usér@example.com
3. 攻擊者收到重置連結 → 接管帳號
```

**漏洞成立的關鍵條件**：

- 系統把 reset link 寄到「使用者輸入的 email」
- 而不是寄到「DB 內查到的那筆 user 的 email」

**為什麼會這樣？**

- 查詢時：`ai_ci` 認為 `usér` 和 `user` 相同
- 寄信時：應用層沒檢查，直接用使用者輸入當目標

> 註：很多成熟的實作會寄到 DB 內該 user 綁定的 email，因此不會中這個漏洞。但如果實作時沒注意到這點，就會出問題。

### 場景 3：IDN 變體註冊

```
用戶 A 註冊：user@例子.com
用戶 B 註冊：user@xn--fsq270a.com（同一個 domain 的 Punycode）

如果沒做 canonicalization：
→ 系統以為是兩個不同 domain
→ 變成兩個帳號
→ 但其實是同一個信箱
```

---

## 目標

避免「忘記密碼用 email 查詢」時因為大小寫/重音/IDN 表示不一致，導致：

- 找到錯的人寄錯信
- 或同一個人用不同表示法變成兩個帳號

---

## 核心策略：Canonical Key + UNIQUE 強制一致

**不要靠查詢時到處加 `COLLATE`**。  
改用：**Canonical Key（標準化 key）+ DB UNIQUE 強制一致**。

### 什麼是 Canonical Key？

把所有「看起來像同一個 email」的輸入，轉換成同一個標準形式：

**例子 1：大小寫變體**

```
User@Example.COM    → canonical → user@example.com
user@example.com    → canonical → user@example.com
USER@EXAMPLE.COM    → canonical → user@example.com
```

**例子 2：IDN（國際化域名）的不同表示法**

```
user@例子.com         → canonical → user@xn--....com（ASCII/Punycode）
user@例子.COM         → canonical → user@xn--....com
user@xn--....com     → canonical → user@xn--....com
```

> **註**：中文域名 `例子.com` 和 Punycode（如 `xn--....com`）是同一個域名的兩種寫法，canonicalization 會統一轉成 ASCII 形式（A-label），避免同一個信箱因為不同寫法而變成兩個帳號。實際的 Punycode 值需用 IDNA 工具轉換。

這樣所有查詢、唯一性檢查都用同一個 key，不會有不一致。

---

## 資料模型建議

保留原始輸入 + 另存 canonical key：

| 欄位              | 用途                               |
| ----------------- | ---------------------------------- |
| `email`           | 原始輸入（顯示、稽核、**寄信用**） |
| `email_canonical` | 查詢/唯一性用 key（精準比對）      |

### MySQL Schema

```sql
ALTER TABLE users
  ADD COLUMN email_canonical VARCHAR(320)
    CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  ADD UNIQUE KEY uq_users_email_canonical (email_canonical);
```

**為什麼用 `utf8mb4_bin`？**

- Binary collation：按位元組精準比對，不會有「意外等價」
- 比 `VARBINARY` 更好：還是 VARCHAR，可讀、可用字串函數

**為什麼不用 `VARBINARY`？**

- 雖然也能精準比對，但對人類不友善（debug 時看到一堆 hex）
- `VARCHAR + utf8mb4_bin` 已經夠用

---

## Canonicalization 規則

### 必做

1. **`trim`**：去前後空白
2. **Unicode NFC 正規化**：避免「同字不同表示法」（見下方說明）
3. **切分 local / domain**：用最後一個 `@` 分割（本篇以一般產品接受的 email 格式為前提；若要完整支援 RFC 複雜語法如引號等，建議使用成熟的 email parser library）
4. **Domain 轉小寫**
5. **Domain 做 IDN → ASCII（toASCII / Punycode）**
   - 把中文域名等 IDN 統一轉成 ASCII 形式（A-label）
   - **注意**：不會把 `gmaîl.com` 變成 `gmail.com`（它會變成不同的 `xn--...`）

### 視產品政策決定

**Local-part（@ 前面）是否轉小寫？**

- **多數消費者產品**：local 也轉小寫
  - 符合使用者期待（Gmail/Outlook 等都不分大小寫）
  - `Tom@example.com` 和 `tom@example.com` 是同一個帳號
- **嚴格 RFC 合規**：local 不轉小寫
  - RFC 5321 規定 local-part 可分大小寫
  - 但要接受 `Tom@` 與 `tom@` 是不同帳號
  - 適合企業 email 系統

**不建議**：不要把「去重音（á→a）」放進 canonicalization，容易誤傷合法差異。

---

## Unicode 正規化（NFC）是什麼

### 為什麼需要正規化

很多字可以用兩種方式表示：

**例子：`é`**

1. **預組字**（precomposed）：單一字元

   - `é` = U+00E9

2. **分解組合**（decomposed）：兩個字元
   - `e` = U+0065
   - `́` （重音符號）= U+0301
   - 合起來看也是 `é`

人眼看都是 `é`，但對電腦來說是不同的字元序列。

如果用 `utf8mb4_bin` 精準比對：

- 這兩種形式會被視為 **不相等**
- 可能造成同一個輸入變兩筆，或查不到

### NFC 正規化

**NFC（Normalization Form C）**：把可以合併的組合序列盡量合併成預組字。

```
輸入：e + ́  （兩個字元）
NFC：é      （單一字元 U+00E9）
```

**目的**：把「看起來一樣」的字，變成同一種 canonical 表示法。

---

## 寫入與查詢規則

### 註冊

```
1. 計算 email_canonical = canonicalize(input_email)
2. INSERT INTO users (email, email_canonical) VALUES (?, ?)
3. UNIQUE(email_canonical) 自動擋掉重複註冊
```

### 登入 / 忘記密碼 / 邀請（所有 email lookup）

```
1. 先算 canonical = canonicalize(input_email)
2. WHERE email_canonical = ? 查 user
```

### 寄信（忘記密碼/驗證信）

**關鍵**：寄信目標用 DB 裡保存的 `email`，不要用使用者當下輸入的字串。

```sql
SELECT email FROM users WHERE email_canonical = ?
-- 取得 DB 裡的 email（已驗證過的）
-- 寄到這個地址
```

---

## 針對「忘記密碼寄錯」的額外防呆

### 1. 查詢時 LIMIT 2

```sql
SELECT email FROM users
WHERE email_canonical = ?
LIMIT 2
```

檢查結果：

- **0 筆**：回傳成功（避免帳號枚舉）
- **1 筆**：正常寄信
- **≥2 筆**：不要寄信，記錄安全事件/走人工審核

理論上 UNIQUE 不會有多筆，但在 migration/過渡期可作為防守。

### 2. 所有流程統一走 email_canonical

**所有涉及 email 查詢的地方都要一致**：

- 註冊
- 登入
- 忘記密碼
- Email 驗證
- 邀請
- 後台查詢

---

## 總結

1. **不要依賴 MySQL 的 ai_ci 預設行為**

   - 會產生「意外等價」
   - 不同查詢可能行為不一致

2. **用 Canonical Key 統一規則**

   - `email_canonical` 用 `utf8mb4_bin`
   - UNIQUE 索引強制唯一性

3. **Canonicalization 規則要明確且一致**

   - NFC 正規化
   - Domain 轉小寫 + toASCII
   - Local 是否轉小寫視產品決定

4. **所有流程都要走同一套邏輯**

   - 註冊、登入、忘記密碼、驗證、邀請

5. **寄信永遠用 DB 的 email，不用使用者輸入**
   - 避免攻擊者用變體騙取重置連結

---

## 參考資料

- Voorivex Team: Puny-Code, 0-Click Account Takeover  
  https://blog.voorivex.team/puny-code-0-click-account-takeover

- MySQL Collation 文件  
  https://dev.mysql.com/doc/refman/8.0/en/charset-collation-names.html

- Unicode Normalization (UAX #15)  
  https://unicode.org/reports/tr15/
