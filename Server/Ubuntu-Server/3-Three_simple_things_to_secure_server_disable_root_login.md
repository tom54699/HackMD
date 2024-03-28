# 自建網頁伺服器的簡易心得系列文(三) - 三個提升Server安全性的設定之禁止Root管理者以SSH登入篇

**前言**
===
:::info
這次要介紹的是我的主管分享給我的一個系統安全性的概念。就是標題說的，禁止在登入 SSH 時使用 root 使用者。通常我們會使用一個普通權限的使用者登入，如果有需要再切換成 root 使用者來操作需要權限的指令。這篇會介紹這樣做的原因，以及如何去設定，最後再提到一些要注意的地方。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
:::  

**禁止 root 使用者原因**
===  
在一般情況下，透過 SSH 登入 Linux 伺服器時，預設使用 root 使用者登入。然而，這種設定存在安全風險。駭客常常針對 root 使用者進行入侵嘗試，因為 root 使用者擁有系統的最高權限。因此，禁止 root 使用者登入可以增加伺服器的安全性。這樣做會為駭客設置更多的障礙，因為他們需要額外的步驟來嘗試猜測其他使用者的名稱。

總結來說，透過禁止 root 使用者透過 SSH 登錄的好處有以下幾點：

1. 降低攻擊面：禁止 root 使用者登入會減少伺服器的攻擊面。駭客必須猜測其他使用者的名稱來嘗試登錄，從而增加了入侵的難度。

2. 減少潛在風險：root 使用者擁有對系統的完全存取權限，如果駭客成功登入為 root 使用者，可能會對伺服器造成嚴重的損害。禁止 root 使用者登入可以降低此類潛在風險。

3. 提高安全性意識：禁止 root 使用者登入可以提高管理員和使用者的安全意識。管理員將被迫使用普通使用者帳戶登錄，並透過提升權限來執行需要 root 權限的操作，這種做法促進了安全最佳實踐的實施。

**設定步驟**
===  

#### **建立普通使用者**  

在禁止 root 使用者 SSH 登入之前，請確保在伺服器上建立了一個普通用戶，用於登入和管理伺服器。這個使用者應該擁有適當的權限來執行系統管理任務。  

建立使用者：

``` bash
adduser <USER_NAME>
```  
![Add User](https://i.imgur.com/A5yN75z.png)

:::danger
建立時記得請使用足夠複雜的密碼，避免常見且太簡單的密碼。
:::

刪除使用者：  

``` bash
userdel <USER_NAME>
``` 
切換使用者： 

``` bash
su - <USER_NAME>
```  

#### **禁止 root SSH 登入**  

1. **修改設定檔**  

    用 root 或是有 sudo 權限的帳號修改 SSH 的設定檔 `/etc/ssh/sshd_config`。  

    ``` bash
    vim /etc/ssh/sshd_config
    ```  
    找到 `PermitRootLogin` 選項，並設定成 `no`：  

    ``` bash
    # 禁止 root 管理者登入
    PermitRootLogin no
    ```  
2. **重新載入設定檔**  

    ``` bash
    # 重新啟動 SSH 服務
    systemctl restart sshd
    ```  
    這樣就設定成功囉，未來就請以之前建立的使用者登入。若要操作需要權限，可以切換成 root 使用者，或是讓普通使用者有 `sudo` 權限。

**額外討論**
===  

在與同事討論後，我們決定在執行需要權限的任務時仍然選擇切換到 root 使用者，而不是提升普通使用者的權限。這項決定的主要原因是，目前我們的伺服器環境中只有兩位成員，使用者情境相對簡單。然而，如果未來需要為其他人員建立額外的使用者帳戶，我們將考慮提升一般使用者的權限。這樣做的目的是為了確保不會將 root 使用者的資訊暴露給其他人員。  

#### **提升權限做法**  

這邊提供兩種最簡單粗暴的做法：  

1. **指令修改**  

    ``` bash
    udo adduser <USER_NAME> sudo
    ```  
2. **設定檔修改**  

    ``` bash
    sudo vim /etc/sudoers
    ```  

    找到以下的選項，把自己的 `USER_NAME` 加上去。

    ``` bash
    # User privilege specification
    root ALL=(ALL:ALL) ALL
    <USER_NAME> ALL=(ALL:ALL) ALL
    ```  
    
### **資料來源**  

[如何禁止 root SSH 用密碼登入 ? 1個關鍵步驟強化 Linux 系統安全性 – Li-Edward](https://liedward.com/linux/root-permitrootlogin/)  
[鳥哥私房菜 - 第十三章、Linux 帳號管理與 ACL 權限設定](https://linux.vbird.org/linux_basic/centos7/0410accountmanager.php#userswitch)  
[[Ubuntu] 解決 xxx is not in the sudoers file - 傑瑞窩在這](https://jerrynest.io/xxx-is-not-in-the-sudoers-file/) 


###### tags: `server` `root` `ssh`