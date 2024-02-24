# 自建伺服器的簡易心得系列文(一) - 三個提升Server安全性的設定 之防火牆篇

**前言**
===
:::info
之前公司的伺服器都是用圖形化介面，為了怕未來會太依賴圖形化介面，所以租了一台 VPS 來架設。  
這系列會持續更新一些有關 Ubuntu Server 架設相關的文章。應該都是新手向，畢竟小弟也不是什麼高手。
:::

**三個提升Server安全性的設定**
===

### 一，安裝防火牆 (UFW)，保護好 Server 的 port  
--- 
#### **安裝 UFW**  
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
    **開啟指令**  

    ``` bash
    sudo ufw allow ssh # = sudo ufw allow 22
    sudo ufw allow 80  # = sudo ufw allow any to any port 80
    sudo ufw allow 443 # = sudo ufw allow any to any port 443
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

### 二，安裝 Fail2ban，防止不明人士一直嘗試登入

### 三，管理好使用者，避免最高使用者外洩