# 自建網站伺服器的簡易心得系列文(二) - 三個提升Server安全性的設定之Fail2ban篇

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
3. 如果沒有啟用，就把它打開  

    enable 代表未來系統重啟的話，也會自動啟動服務。  
    start 則是立即手動啟動，不用等待系統重啟。

    ``` bash
    sudo systemctl enable fail2ban

    sudo systemctl start fail2ban
    ```

#### **Fail2Ban 設定** 

Fail2Ban 的應用主要是要去根據需求去更改設定檔。  
:::success
設定檔的位置 Ubuntu 系統位於 /etc/fail2ban/jail.conf。
:::  

`fail2ban.conf` 文件包含 Fail2Ban 的基本配置。通常會複製 `jail.conf` 的內容，然後創建一個 `fail2ban.local` 文件，將其用作主要的設定文件。  

---
因為 fail2ban 有很多功能和複雜的設定，這邊今天只介紹一些簡單常用的設定：

1. **設定 IP 白名單**  

    `ignoreip` 的值可以是多個 IP 位址、CIDR 網段、DNS 主機名稱，各 IP 之間以空白分隔。
    
    ``` bash
    ignoreip = 127.0.0.1/8 0.0.0.0
    ```  
2. **設定封鎖時間**  

    `bantime` 參數可設定每一次阻擋攻擊來源的持續時間。
    
    ``` bash
    ibantime  = 7d
    ```  
3. **惡意攻擊判斷標準**  

    `findtime` 參數表示一段觀察時間，而 `maxretry` 參數則是表示錯誤嘗試的次數。  
    當在 `fidntime` 所設定的時間內，達到 `maxretry` 所設定的錯誤次數時，就會被判定為惡意攻擊然後被封鎖。
    
    ``` bash
    findtime  = 10m
    maxretry = 3
    ```  
4. **各種預設服務**  

    預設的狀態只有啟用 sshd 服務的監控，所以設定好上面幾個簡單設定，就可以算是完成了基本的設定。其實在預設的設定檔中，還提供很多服務的預設設定，像是mysql, grafana, nginx-http-auth...。如果有需要可以再去開啟即可，因為大部分已經出現在設定檔案的服務的action, filter 早就已經寫好放在 `/etc/fail2ban/action.d/`, `/etc/fail2ban/filter.d/` 裡面。  
    
#### **進階防護 - Fail2ban 結合 ufw 防火牆**  

其實做好上一篇說的防火牆和設定基本的 Fail2ban 機制，已經可以做到一些基本的防護。 但如果要更近一步，我們可以把防火牆和 Fail2ban 做結合Fail2ban。 預期的效果會是，如果有人刻意訪問伺服器沒有公開的埠號時，也要建立機制把該 IP 封鎖。 

但因為預設並沒有提供 ufw 的服務設定，我們就需要自己手動建立相關的 jail 和 filter 設定檔，那我們就來實做看看吧:  

1. **jail.local 設定部分**  

    這隻檔案就是我們先前提到過的 Fail2ban 的主要設定檔，我們需要在裡面加入我們針對 ufw 的封鎖策略。
    
    ``` bash
    [ufw]
    enabled=true
    filter=ufw.aggressive
    action=iptables-allports
    logpath=/var/log/ufw.log
    maxretry=3
    bantime=7d
    ```  

2. **filter 設定部分**  

    現在要建立剛剛定義的 filter，在 `/etc/fail2ban/filter.d/` 新增一個
    `ufw.aggressive.conf`。  

    ``` bash
    [Definition]
    failregex = [UFW BLOCK].+SRC=<HOST> DST
    ignoreregex =
    ```  

3. **載入新的規則和檢查狀態**  

    剛剛已經把規則都設定好了，這時候需要讓它重新載入規則。  

    ``` bash
    sudo systemctl reload fail2ban
    ```  
    然後可以檢查一下現在新規則的狀態  

    ``` bash
    sudo fail2ban-client status ufw
    ``` 

#### **解除 Fail2ban 封鎖方法**  

有時候當我們規則寫得越嚴謹，雖然會提高安全性，但也會增加把自己不小心誤封鎖的窘境。尤其結合防火牆去做封鎖時，小弟曾經因為一直想進去某個服務但忘記開啟該服務埠號，而慘遭封鎖。  

當然如果有固定 IP，你就可以把該 IP永久加入白名單，就比較不會出現該問題。但如果沒有又不小心被封鎖了，可以依照以下步驟解除封鎖:  

1. **先切換別的網路**  

    因為原本的 IP 被封鎖了，所以也無法透過 SSH 連線到 Server處理。 
    
    :::danger
    這邊特別注意在切換網路之前，先確保你剛剛在執行的服務有先關掉，不然你第二個 IP 有可能會馬上又被封鎖。
    :::

2. **如果知道自己被封鎖的 IP**

    ``` bash
    sudo fail2ban-client unban <IP_ADDRESS> 
    ```  
    如果不知道 IP 就執行下個步驟，查看封鎖資訊。  

3. **查看封鎖的紀錄**  

    這邊要透過 Fail2ban 的 Log 來猜測自己可能是哪一個IP，所以最好在封鎖的時候看一下時間判斷一下，不然就要多嘗試幾個了。

    ``` bash
    sudo cat /var/log/fail2ban.log 

    sudo less /var/log/fail2ban.log #查看更多資訊
    ```  
    ![Fail2ban Log - 1](https://i.imgur.com/BqoTbKm.png) 

    範例圖 

    ![Fail2ban Log - 2](https://i.imgur.com/4e4t7O3.png)  

    前兩條資訊代表`fail2ban`的`filter`模組監視日誌文件，當偵測到某個IP 位址頻繁出現並符合特定的規則時，它會記錄相關資訊。

    1. `[ufw] Found 51.178.249.89`表示`fail2ban`
        
        在日誌中發現了IP 位址為 `51.178.249.89` 的活動。
        
    2. `[ufw] Ban 51.178.249.89`
        
        表示`fail2ban`將IP `51.178.249.89`位址為的主機加入了`ufw`規則(jail)的封鎖清單中。 
    
    如果找到了就可以用第一個步驟的指令來解除封鎖。  

4. **小補充 - 如果要解封被特定 Jail 封鎖的 IP**  

    剛剛第一步驟的指令是解除所有 Jail 封鎖的 IP，但如果今天只是要幫特定人員解除某個服務的封鎖，像是只想幫他解除 mysql 服務的封鎖，但 SSH想繼續封鎖他的 IP。就要指定 Jail 來下指令:  

    ``` bash
    sudo fail2ban-client set <JAIL_NAME> unbanip <IP_ADDRESS>
    ```  

### **資料來源**  

[搭配 Fail2ban 來使用 UFW ｜ fernvenue's Blog](https://blog.fernvenue.com/zh/archives/ufw-with-fail2ban/)  
[GitHub - Fail2ban](https://github.com/fail2ban/fail2ban) 
[Linux作業系統如何限制SSH在一段時間內的嘗試登入次數，來避免被暴力破解登入？ | MagicLen](https://magiclen.org/fail2ban-ssh/#google_vignette) 
[Fail2Ban- 使用 Fail2Ban 保護您的 Linux 伺服器 - TAKI官方部落格](https://www.taki.com.tw/blog/%E4%BD%BF%E7%94%A8-fail2ban-%E9%85%8D%E7%BD%AE%E4%BF%9D%E8%AD%B7%E6%82%A8%E7%9A%84-linux-%E4%BC%BA%E6%9C%8D%E5%99%A8/)  
[運用 Fail2ban 自動封鎖攻擊者的 IP | 艦長，你有事嗎？](https://chengweichen.com/2021/12/fail2ban-log4shell-log4j.html) 

###### tags: `server` `fail2ban` `ufw`