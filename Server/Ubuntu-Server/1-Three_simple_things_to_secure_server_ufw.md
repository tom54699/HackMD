# 自建網頁伺服器的簡易心得系列文(一) - 三個提升Server安全性的設定之防火牆篇

**前言**
===
:::info
根據小弟的經驗，如果伺服器開放了對外的埠口，很容易在日誌中看到大量的機器人和爬蟲嘗試登入資料庫或 SSH。這些登入嘗試的頻率之高，即使沒有暴力破解成功，也會讓人感到困擾。因此，使用防火牆來設定規則，並謹慎地決定是否真的需要對外開放埠口，是保護資訊安全的第一步。
:::

**防火牆基本安裝＆設定指南**
===  
#### **安裝防火牆 (UFW)，保護好 Server 的 port**  
1. 安裝指令  

    ``` bash
    sudo apt update
    sudo apt install ufw+
    ```
    :::success
    :bulb: Ubuntu 8.04 LTS 之後的版本都會有個內建的防火牆 “UFW”。
    :::  

#### **設定步驟** 

1. **先允許 ssh 連線**  

    ``` bash
    sudo ufw allow OpenSSH
    ```  

2. **啟用防火牆 (預設是關閉的)**  

    ``` bash
    sudo ufw enable
    ```
    *除了啟動，還有如果重啟時防火牆也會自動打開*  

3. **開啟需要的 Port**  

    先開啟會用到的服務相關的 port，未來有新的需求再開啟即可。  

    ``` bash
    ssh             22/tcp          # SSH Remote Login Protocol
    http            80/tcp          # WorldWideWeb HTTP
    https           443/tcp         # http protocol over TLS/SSL
    ```  
    **開啟指令&規則**  

    ``` bash
    sudo ufw default allow                       # 預設全部允許
    sudo ufw default deny                        # 預設全部阻擋
    sudo ufw allow ssh                           # = sudo ufw allow 22
    sudo ufw allow 80                            # = sudo ufw allow any to any port 80
    sudo ufw allow 443                           # = sudo ufw allow any to any port 443
    ufw allow from 192.168.0.1                   #允許來自 192.168.0.1 通過所有連線
    ufw allow from 192.168.0.1 to any port 3306  #允許來自 192.168.0.1 通過 3306 Port
    ```  

    **關閉指令**  

    可直接使用設定 UFW 時的規則，加上 delete 參數即可: 

    ``` bash
    sudo ufw status # 看目前設定過的規則
    ```   

    ``` bash
    sudo ufw delete allow 80
    ```  
    也可以查詢規則編號後刪除:  

    ``` bash
    sudo ufw status numbered # 看目前設定過的規則編號
    ```  
      
    ``` bash
    sudo ufw delete 3
    ```  

#### **常用指令**  
1. 重設 UFW 規則  

    ``` bash
    sudo ufw reset
    ```  
2. 停用 UFW，且開機停用  

    ``` bash
    sudo ufw disable
    ```  
3. 查看 UFW 更多資訊  

    ``` bash
    sudo ufw status verbose
    ```  
4. UFW 應用程式配置的清單 

    ``` bash
    sudo ufw app list
    ``` 
### **資料來源**  

[簡易的防火牆 - UFW & GUFW ·  完全用 GNU/Linux 工作](https://chusiang.gitbooks.io/working-on-gnu-linux/content/07.ufw.html)  
[Ubuntu Server 20.04.1 預設 UFW 防火牆 Firewall 設定規則詳解和教學](https://footmark.com.tw/news/linux/ubuntu/ubuntu-server-ufw/)  
[Ubuntu 防火牆設定 - 使用 ufw 指令](https://blog.tarswork.com/post/ubuntu-firewall-setting-using-ufw/)  

<!-- ### 二，安裝 Fail2ban，防止不明人士一直嘗試登入

### 三，管理好使用者，避免最高使用者外洩 -->

###### tags: `server`