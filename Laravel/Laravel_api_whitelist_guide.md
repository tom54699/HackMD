### **Laravel API 白名單與「真實用戶 IP」技術學習筆記**

#### **前言：為何需要這份手冊？**

在為 Laravel 應用程式實作 IP 白名單時，我們經常遇到一個核心挑戰：當服務部署在 Cloudflare 或其他反向代理後方時，應用程式直接取得的 IP 位址會是代理伺服器的，而非客戶端的真實來源 IP。這份手冊將系統性地闡述從網路基礎、伺服器配置到 Laravel 框架層，如何正確取得並驗證「真實用戶 IP」，確保白名單機制的安全與精確。

---

#### **1. 核心問題：IP 白名單的基礎——CIDR**

當合作方提供 IP 位址時，通常會以 CIDR（無類別域間路由）格式提供，例如 `192.0.2.0/28`。理解其含義是設定白名單的第一步。

*   **CIDR 格式**：`位址/前綴長度`。可用的主機數量由 `2^(32 − 前綴長度)` 決定（以 IPv4 為例）。

| CIDR | 涵蓋 IP 數量 | 適用情境與安全考量 |
| :--- | :--- | :--- |
| **/32** | **1 個** | **白名單首選**。最小攻擊面，精確指向單一伺服器。 |
| /28 | 16 個 | 適用於對方擁有多個固定出口 IP 的叢集或 NAT 環境。需確認必要性，否則範圍過寬。 |
| /24 | 256 個 | 常見於企業內部網段，但作為 API 白名單通常**過於危險**。 |

**判斷準則**：**永遠優先要求 `/32` 的單一 IP**。僅在對方能明確證明其所有出口 IP 都在一段連續範圍內且皆為固定 IP 時，才考慮接受如 `/28` 的網段。若 IP 不連續，應要求提供多個 `/32` 位址。

---

#### **2. 原理剖析：CDN 如何影響 IP 獲取？**

要解決問題，必須先理解請求的流向。

*   **入站流量 (Inbound)**：`用戶 → CDN/反向代理 → 你的 Nginx 伺服器`
    *   這是訪客或 API 客戶請求你服務的的路徑。請求會先抵達 Cloudflare，再由 Cloudflare 轉發至你的主機。**這是我們需要還原真實 IP 的場景**。
*   **出站流量 (Outbound)**：`你的伺服器 → 外部 API`
    *   這是你的伺服器主動請求外部服務的路徑。流量會直接從你的主機房 ISP 出口，**不受 CDN 影響**。

**結論**：CDN 只影響「入站」請求的來源 IP。若要檢測自己伺服器的出口 IP，可使用 `curl ifconfig.co` 指令（詳見第 6 節）。

---

#### **3. 關鍵標頭：Cloudflare 與多層代理的 IP 識別**

當請求經過 Cloudflare 時，它會將真實的客戶端 IP 資訊附加在 HTTP 標頭 (Header) 中傳遞給後端伺服器。

*   `CF-Connecting-IP`：Cloudflare 官方推薦使用的標頭。它代表 Cloudflare 視角下的「客戶端真實 IP」，通常是單一值，**應優先採信**。
*   `X-Forwarded-For` (XFF)：一個業界標準，記錄了請求經過的每一層代理 IP 鏈，格式為 `client, proxy1, proxy2, ...`。在多層代理環境下較為複雜。
*   其他輔助標頭：`CF-Ray` (請求 ID), `CF-IPCountry` (來源國別), `cdn-loop` (防止代理迴圈)。

---

#### **4. Nginx 實作：「真實 IP 還原」模組（`ngx_http_realip_module`）**

這是整個流程中最核心的步驟：設定 Nginx，使其能讀懂 CDN 傳來的標頭，並將 `$remote_addr` 變數還原為真實 IP。

##### **4.1 核心指令詳解**

```nginx
# 宣告哪些來源的代理是可信的。只有來自這些 IP 的請求，Nginx 才會處理 real_ip_header。
set_real_ip_from <IP或CIDR>;

# 指定哪個 Header 攜帶了真實 IP 位址。
real_ip_header <Header名稱>;

# 在處理 X-Forwarded-For 這類多值標頭時，是否要遞迴解析，跳過可信代理 IP。
real_ip_recursive on|off;
```

##### **4.2 Nginx Cloudflare 推薦設定**

將以下設定加入 `nginx.conf` 的 `http {}` 區塊中，以應用於所有站點。

```nginx
# --- Begin Real IP Configuration ---

# 1. 信任所有 Cloudflare 的 IP 來源（v4 和 v6）
#    注意：此列表應定期從 Cloudflare 官方網站更新：https://www.cloudflare.com/ips/
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
# ... (此處省略其他 Cloudflare IP 段)
set_real_ip_from 2a06:98c0::/29;
# ... (此處省略其他 Cloudflare IPv6 段)

# 2. 告知 Nginx 從 CF-Connecting-IP 標頭中讀取真實 IP
real_ip_header CF-Connecting-IP;

# 3. 啟用遞迴解析（可選但推薦）
#    雖然 CF-Connecting-IP 是單一值，但開啟此選項無副作用，且能兼容其他代理場景。
real_ip_recursive on;

# --- End Real IP Configuration ---
```

**安全要點**：`set_real_ip_from` 清單**絕對不能**設定為 `0.0.0.0/0` 或 `::/0`，這等同於信任任何來源的 IP 偽造，會造成嚴重安全漏洞。

##### **4.3 流程對照**

| 狀態 | 請求流程 | Nginx `$remote_addr` 結果 |
| :--- | :--- | :--- |
| **無代理** | `用戶 → Nginx` | ✅ **用戶真實 IP** |
| **經 CDN (未設定 RealIP)** | `用戶 → CF → Nginx` | ❌ **Cloudflare 節點 IP** |
| **經 CDN (已正確設定)** | `用戶 → CF → Nginx` | ✅ **用戶真實 IP** |

---

#### **5. Laravel 層：以最簡潔的方式獲取 IP**

當 Nginx 層面已正確還原 `$remote_addr` 後，Laravel 的工作將變得極其簡單。

```php
// 在你的 Middleware 或 Controller 中
$ip = request()->ip(); // 這就是經過還原的、可信的真實 IP (可能是 v4 或 v6)
```

**關於 `TrustProxies` Middleware**：
Laravel 的 `app/Http/Middleware/TrustProxies.php` 用於處理代理。若 Nginx 已完成上述設定，將 `TrustProxies` 的 `$proxies` 設為 `'*'` 是安全的，因為信任邊界已由 Nginx `set_real_ip_from` 嚴格把關。反之，若未設定 Nginx，直接信任所有代理 (`'*'`) 將允許客戶端偽造 `X-Forwarded-For` 標頭，繞過 IP 限制。

---

#### **6. 自我檢測：如何指導客戶提供正確的 IP？**

為了避免溝通誤差，你可以提供明確的指令，讓客戶在他們的伺服器環境中執行，以獲得其準確的出口 IP。

**指令稿：**

> 您好，為了將您的服務加入我們的 API 白名單，請在您**發起 API 請求的伺服器**上執行以下指令，並將回傳的 `ip` 欄位值提供給我們：
>
> *   **若您使用 IPv4 出口**：
>     ```shell
>     curl -4s ifconfig.co/json
>     ```
> *   **若您使用 IPv6 出口**：
>     ```shell
>     curl -6s ifconfig.co/json
>     ```
> *   **若不確定，請兩者皆執行並提供結果。**

這可以確保你拿到的是對方真實的出口 IP，而不是他們辦公室的網路 IP 或其他無關位址。

---

#### **總結：重點步驟回顧**

| 步驟 | 核心任務 | 關鍵指令/方法 | 安全備註 |
| :--- | :--- | :--- | :--- |
| **1. 需求確認** | 向合作方索取 IP | 要求 `/32` 格式 | 避免接受過寬的 CIDR 網段 |
| **2. Nginx 設定** | 還原真實 IP | `set_real_ip_from` + `real_ip_header` | 僅信任 Cloudflare 官方 IP 列表 |
| **3. Laravel 獲取** | 在程式中取得 IP | `request()->ip()` | 確保 Nginx 已正確配置 |
| **4. 客戶指引** | 協助對方自查 IP | `curl -4/-6 ifconfig.co/json` | 避免因資訊錯誤導致的連線問題 |


