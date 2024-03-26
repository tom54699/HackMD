# 自建網頁伺服器的簡易心得系列文(一) - 三個提升Server安全性的設定之Fail2ban篇

**前言**
===
:::info
即使已經使用防火牆，為什麼還需要 Fail2ban 呢？因為通常會開放一些埠號，如 SSH，以供外部使用。因此，日誌中會記錄到一些嘗試暴力破解的行為。在這種情況下，Fail2ban 可以分析日誌信息，根據設置的規則來封鎖這些惡意登錄者的 IP 地址。這樣可以進一步提高伺服器的安全性。不過，需要注意設置規則時要小心，有時候甚至可能因為自己輸錯密碼而把自己的 IP 地址鎖在小黑屋裡。  
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
:::  

**Fail2ban 簡單介紹**
===  

Fail2ban 是一種用於防範惡意登入嘗試和拒絕服務攻擊的工具，它可以監視系統日誌文件，並在偵測到惡意行為後採取自動化措施來保護伺服器。
<details open>  
<summary>以下是Fail2ban 的基本運作原理和功能：</summary>

1. 監視日誌檔案：Fail2ban 會持續監視系統記錄文件​​，如SSH、Web 伺服器或其他服務的存取日誌。

2. 分析日誌：它會分析日誌檔案中的活動，偵測到惡意行為，例如多次失敗的登入嘗試、經常存取特定頁面等。

3. 動態阻止惡意IP：一旦發現惡意行為，Fail2ban 將根據預先定義的規則自動阻止相關IP 位址的訪問，可以將其新增至防火牆規則。

4. 自動解除封鎖：Fail2ban 也支援自動解除封鎖，即在一段時間後，自動解除封鎖IP 的限制。

5. 靈活的配置：Fail2ban 允許管理員定義自訂的規則和動作，以滿足特定的安全需求。可以設定監視的日誌檔案、偵測到的惡意行為、封鎖策略等。

透過以上機制，Fail2ban 可以大幅減少惡意存取和拒絕服務攻擊對伺服器的影響，並提高系統的安全性和穩定性。
</details>  

**Fail2ban 基本安裝＆設定指南**
===  
#### **安裝 Fail2ban**  
1. 安裝指令  

    ``` bash
    apt-get update && apt-get upgrade
    sudo apt install fail2ban
    ```
2. 查看 Fail2Ban 狀態  

    ``` bash
    sudo systemctl status fail2ban
    ```

#### **Fail2Ban 設定** 

Fail2Ban 的應用主要是要去根據需求去更改設定檔。  
:::success
設定檔的位置 Ubuntu 系統位於 /etc/fail2ban/jail.conf。
:::  

fail2ban.conf 文件包含 Fail2Ban 的基本配置。通常會複製 jail.conf 的內容，然後創建一個 fail2ban.local 文件，將其用作主要的設定文件。  

---
因為 fail2ban 有很多功能和複雜的設定，這邊今天只介紹一些簡單常用的設定：

1. **設定 IP 白名單**  

    ignoreip 的值可以是多個 IP 位址、CIDR 網段、DNS 主機名稱，各 IP 之間以空白分隔。
    
    ``` bash
    ignoreip = 127.0.0.1/8 0.0.0.0
    ```  
2. **設定封鎖時間**  

    bantime 參數可設定每一次阻擋攻擊來源的持續時間。
    
    ``` bash
    ibantime  = 7d
    ```  
3. **惡意攻擊判斷標準**  

    findtime 參數表示一段觀察時間，而 maxretry 參數則是表示錯誤嘗試的次數。  
    當在 fidntime所設定的時間內，達到maxretry所設定的錯誤次數時，就會被判定為惡意攻擊然後被封鎖。
    
    ``` bash
    findtime  = 10m
    maxretry = 3
    ```  
4. **各種預設服務**  

    預設的狀態只有啟用 sshd 服務的監控，所以設定好上面幾個簡單設定，就可以算是完成了基本的設定。其實在預設的設定檔中，還提供很多服務的預設設定，像是mysql, grafana, nginx-http-auth...。如果有需要可以再去開啟即可，因為大部分已經出現在設定檔案的服務的action, filter早就已經寫好放在 /etc/fail2ban/action.d/, /etc/fail2ban/filter.d/ 裡面。
    


### **資料來源**  

[簡易的防火牆 - UFW & GUFW ·  完全用 GNU/Linux 工作](https://chusiang.gitbooks.io/working-on-gnu-linux/content/07.ufw.html)  
[Ubuntu Server 20.04.1 預設 UFW 防火牆 Firewall 設定規則詳解和教學](https://footmark.com.tw/news/linux/ubuntu/ubuntu-server-ufw/)  
[Ubuntu 防火牆設定 - 使用 ufw 指令](https://blog.tarswork.com/post/ubuntu-firewall-setting-using-ufw/)  

<!-- ### 二，安裝 Fail2ban，防止不明人士一直嘗試登入

### 三，管理好使用者，避免最高使用者外洩 -->

###### tags: `server` `fail2ban`