# 使用 Rclone 在本地(Windows) 備份 Linux 伺服器上的檔案

**前言**
===
:::info
這篇文章的主要目標是定期在本地 Windows 主機備份雲端 Linux Server 上的專案、SQL 檔案和 Docker Volume 資料夾。在這過程中，我們將會使用一個名為 rclone 的工具，該工具能夠實現數據同步和轉移的功能。透過 rclone，我們能夠有效地將本地和雲端的資料同步，從而保證資料的安全性和可靠性。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
:::  

**Rclone 簡單介紹**  
===  

`rclone` 是一個用於與多種不同雲端存儲提供商進行數據同步和轉移的命令行工具。它允許您管理和操作不同雲端存儲服務中的文件和數據，並且可以在本地系統和不同雲端存儲之間進行文件傳輸。

以下是 rclone 的一些主要功能和用途：

1. **支援多種雲端存儲服務**：rclone 可以與許多不同的雲端存儲提供商集成，包括 Google Drive、Amazon S3、Dropbox、Microsoft OneDrive、Box、以及許多其他類型的存儲服務。
2. **數據同步和備份**：您可以使用 rclone 來同步本地文件和文件夾與雲端存儲之間的數據，以進行備份或確保數據同步。
3. **遠程文件管理**：rclone 允許您在不同雲端存儲服務中進行文件管理，包括上傳、下載、刪除、移動文件等操作。
4. **加密和壓縮**：rclone 支援數據的加密和壓縮，以保護敏感信息或節省存儲空間。
5. **配置靈活**：您可以通過 rclone 配置文件進行靈活的配置，以設置不同雲端存儲服務的訪問權限和參數。
6. **跨平台**：rclone 可在多種不同操作系統上運行，包括 Linux、macOS 和 Windows。
7. **強大的命令行工具**：rclone 提供了一個豐富的命令行工具集，用於執行各種操作，並支援腳本化和自動化任務。

總之，rclone 是一個強大的工具，可幫助使用者輕鬆管理和操作不同雲端存儲服務中的數據。無論是備份、同步還是文件管理，rclone 都為這些任務提供了便捷的解決方案。  

**實作流程**  
===   

#### 1. **伺服器和本地都先安裝 rclone**  

   - centos 安裝指令：  
   
      ```bash
      curl https://rclone.org/install.sh | bash
      ```

   - window 安裝 rclone 網址：
    
      ```bash
      # 主程式下載
      https://rclone.org/downloads/
      ```
      下載後，要打開要 cmd 到該資料夾 輸入 rclone 指令
      嫌麻煩可以根據底下參考教學去把他加入到環境變數。

      ```bash
      # winfsp安裝 - 一直下一步即可
      https://github.com/winfsp/winfsp/releases
      ```
    
   - 參考文章:  

      [Rclone 簡易操作教學 - HackMD](https://hackmd.io/@Restforlife/ryltomajfi)  

      [win10 Rclone 不思考ㄉ安裝與掛載 - HackMD](https://hackmd.io/@adeliae/H1ZRCUfaU)  


#### 2. **安裝 Git bash** 

   通常電腦有安裝 Git 應該就會找得到。主要是為了省事可以在 window 用 linux 指令，所以才要使用到 Git bash。  

   安裝網址和教學：[本文可協助您下載並安裝使用 Git 和編輯 Markdown 檔案以編輯 Microsoft Learn 上的檔所需的用戶端工具。](https://learn.microsoft.com/zh-tw/contribute/content/get-started-setup-tools?pivots=windows-os-pivot-selection)
        
#### 3. **開始進行 rclone 設定 （本地）**  

   - 這次要建立 ssh/sftp 連線，要在本地先建立 ssh key。  
      
      自行決定要把 key 放到哪個資料夾，這邊我放在 c 槽的 ssh 資料夾。
      ```bash
      ssh-keygen -q -t rsa -b 4096 -C "rclone key" -N "" -f ~/.ssh/rclone
      cd ~/.ssh/
      cat rclone* > rclone-merged
      ```
        
   - 把剛剛生成的 pub key 丟到伺服器。
        
      ```bash
      ssh-copy-id -i ~/.ssh/rclone.pub root@<your_server_ip>
      ```
        
   - 階段一，新增連線和命名  

      ```bash
      No remotes found - make a new one
      n) New remote
      s) Set configuration password
      q) Quit config
      n/s/q> n (選 n 新增連線)
      name> Server_backup (命名 Server_backup)
      ```  

   - 階段二，選擇你要連接的方式
         
      ```bash    
      Option Storage.
      Type of storage to configure.
      Enter a string value. Press Enter for the default ("").
      Choose a number from below, or type in your own value.
      1 / 1Fichier
         \ "fichier"
      2 / Alias for an existing remote
         \ "alias"
      3 / Amazon Drive
         \ "amazon cloud drive"
      4 / Amazon S3 Compliant Storage Providers including AWS, Alibaba, Ceph, Digital Ocean, Dreamhost, IBM COS, Minio, SeaweedFS, and Tencent COS
         \ "s3"
      5 / Backblaze B2
         \ "b2"
      6 / Better checksums for other remotes
         \ "hasher"
      7 / Box
         \ "box"
      8 / Cache a remote
         \ "cache"
      9 / Citrix Sharefile
         \ "sharefile"
      10 / Compress a remote
         \ "compress"
      11 / Dropbox
         \ "dropbox"
      12 / Encrypt/Decrypt a remote
         \ "crypt"
      13 / Enterprise File Fabric
         \ "filefabric"
      14 / FTP Connection
         \ "ftp"
      15 / Google Cloud Storage (this is not Google Drive)
         \ "google cloud storage"
      16 / Google Drive
         \ "drive"
      17 / Google Photos
         \ "google photos"
      18 / Hadoop distributed file system
         \ "hdfs"
      19 / Hubic
         \ "hubic"
      20 / In memory object storage system.
         \ "memory"
      21 / Jottacloud
         \ "jottacloud"
      22 / Koofr
         \ "koofr"
      23 / Local Disk
         \ "local"
      24 / Mail.ru Cloud
         \ "mailru"
      25 / Mega
         \ "mega"
      26 / Microsoft Azure Blob Storage
         \ "azureblob"
      27 / Microsoft OneDrive
         \ "onedrive"
      28 / OpenDrive
         \ "opendrive"
      29 / OpenStack Swift (Rackspace Cloud Files, Memset Memstore, OVH)
         \ "swift"
      30 / Pcloud
         \ "pcloud"
      31 / Put.io
         \ "putio"
      32 / QingCloud Object Storage
         \ "qingstor"
      33 / SSH/SFTP Connection
         \ "sftp"
      34 / Sia Decentralized Cloud
         \ "sia"
      35 / Sugarsync
         \ "sugarsync"
      36 / Tardigrade Decentralized Cloud Storage
         \ "tardigrade"
      37 / Transparently chunk/split large files
         \ "chunker"
      38 / Union merges the contents of several upstream fs
         \ "union"
      39 / Uptobox
         \ "uptobox"
      40 / Webdav
         \ "webdav"
      41 / Yandex Disk
         \ "yandex"
      42 / Zoho
         \ "zoho"
      43 / http Connection
         \ "http"
      44 / premiumize.me
         \ "premiumizeme"
      45 / seafile
         \ "seafile"
      Storage> 33  (輸入 sftp 對應序號 33)
      ```  

   - 階段三，設定 host 相關驗證設定。  

      ```bash  
      Option host.
      SSH host to connect to.
      E.g. "example.com".
      Enter a string value. Press Enter for the default ("").
      host> <your_server_ip>
      Option user.
      SSH username, leave blank for current username, ethan.
      Enter a string value. Press Enter for the default ("").
      user> <your_server_user_name>
      Option port.
      SSH port, leave blank to use default (22).
      Enter a string value. Press Enter for the default ("").
      port> 22
      Option pass.
      SSH password, leave blank to use ssh-agent.
      Choose an alternative below. Press Enter for the default (n).
      y) Yes type in my own password
      g) Generate random password
      n) No leave this optional password blank (default)
      y/g/n> n
      Option key_pem.
      Raw PEM-encoded private key.
      If specified, will override key_file parameter.
      Enter a string value. Press Enter for the default ("").
      key_pem> #回車
      Option key_file.
      Path to PEM-encoded private key file.
      Leave blank or set key-use-agent to use ssh-agent.
      Leading `~` will be expanded in the file name as will environment variables such as `${RCLONE_CONFIG_DIR}`.
      Enter a string value. Press Enter for the default ("").
      key_file> ~/.ssh/rclone-merged （選擇你剛剛建立的key）
      Option key_file_pass.
      The passphrase to decrypt the PEM-encoded private key file.
      Only PEM encrypted key files (old OpenSSH format) are supported. Encrypted keys in the new OpenSSH format cant be used.
      Choose an alternative below. Press Enter for the default (n).
      y) Yes type in my own password
      g) Generate random password
      n) No leave this optional password blank (default)
      y/g/n> n
      Option pubkey_file.
      Optional path to public key file.
      Set this if you have a signed certificate you want to use for authentication.
      Leading `~` will be expanded in the file name as will environment variables such as `${RCLONE_CONFIG_DIR}`.
      Enter a string value. Press Enter for the default ("").
      pubkey_file> #回車
      Option key_use_agent.
      When set forces the usage of the ssh-agent.
      When key-file is also set, the ".pub" file of the specified key-file is read and only the associated key is
      requested from the ssh-agent. This allows to avoid `Too many authentication failures for *username*` errors
      when the ssh-agent contains many keys.
      Enter a boolean value (true or false). Press Enter for the default ("false").
      key_use_agent> #回車
      Option use_insecure_cipher.
      Enable the use of insecure ciphers and key exchange methods.
      This enables the use of the following insecure ciphers and key exchange methods:
      - aes128-cbc
      - aes192-cbc
      - aes256-cbc
      - 3des-cbc
      - diffie-hellman-group-exchange-sha256
      - diffie-hellman-group-exchange-sha1
      Those algorithms are insecure and may allow plaintext data to be recovered by an attacker.
      Enter a boolean value (true or false). Press Enter for the default ("false").
      Choose a number from below, or type in your own value.
      1 / Use default Cipher list.
         \ "false"
      2 / Enables the use of the aes128-cbc cipher and diffie-hellman-group-exchange-sha256, diffie-hellman-group-exchange-sha1 key exchange.
         \ "true"
      use_insecure_cipher> #回車
      Option disable_hashcheck.
      Disable the execution of SSH commands to determine if remote file hashing is available.
      Leave blank or set to false to enable hashing (recommended), set to true to disable hashing.
      Enter a boolean value (true or false). Press Enter for the default ("false").
      disable_hashcheck> #回車
      Edit advanced config?
      y) Yes
      n) No (default)
      y/n> n
      --------------------
      [movie]
      type = sftp
      host = 192.168.32.33
      user = root
      port = 22
      key_file = ~/.ssh/rclone-merged
      --------------------
      y) Yes this is OK (default)
      e) Edit this remote
      d) Delete this remote
      y/e/d> y
      Current remotes:
      
      Name                 Type
      ====                 ====
      Server_backup        sftp
      
      e) Edit existing remote
      n) New remote
      d) Delete remote
      r) Rename remote
      c) Copy remote
      s) Set configuration password
      q) Quit config
      e/n/d/r/c/s/q> q
      ```  

      這樣 rclone 的 連線就建立完成。  


#### 4. **撰寫備份腳本**  
        
   - 建立 `sql_backup.sh`，這個腳本的主要工作是要抓取和定期清理伺服器中的 sql 資料。這邊請根據需求去做調整，這邊僅供參考。
            
      ```bash  
      #!/bin/bash

      # 設定遠端連接名稱
      remote_name="<剛剛設定的 rclone 連線名稱>"

      # 設置本地保存目錄
      local_directory="/c/backup/sql_backup"

      echo "Starting SQL backup process..."

      # 創建以日期命名的目錄
      current_date=$(date +"%Y-%m-%d")
      current_time=$(date +"%Y-%m-%d-%H-%M")
      backup_directory="$local_directory/$remote_name/$current_date"

      # 刪除十天前的備份文件
      echo "Deleting SQL backup files older than ten days in $local_directory..."
      find $local_directory -type d -mtime +10 -exec rm -rf {} \;
      echo "Old SQL backup files deleted."

      # 下載文件到新創建的目錄
      echo "Copying SQL backup file from $remote_name to $backup_directory/$current_time..."
      rclone copy $remote_name:/tmp/sql_backup/ "$backup_directory/$current_time"
      echo "SQL backup file copied successfully."

      echo "Backup process completed successfully."

      read -p "Press Enter to exit"
      ```       
   - 建立 `volume_backup.sh`，這個腳本的主要工作是要抓取和定期清理伺服器中的所有 Docker volume 資料夾。這邊請根據需求去做調整，這邊僅供參考。

      ``` bash
      #!/bin/bash

      # 設定遠端連接名稱
      remote_name="<剛剛設定的 rclone 連線名稱>"

      # 設置本地保存目錄
      local_directory="/c/backup/volume_backup"

      echo "Starting volume backup process..."

      # 創建以日期命名的目錄
      current_date=$(date +"%Y-%m-%d")
      current_time=$(date +"%Y-%m-%d-%H-%M")
      backup_directory="$local_directory/$remote_name/$current_date"

      # 刪除十天前的備份文件
      echo "Deleting volume backup files older than ten days in $local_directory..."
      find $local_directory -type d -mtime +10 -exec rm -rf {} \;
      echo "Old volume backup files deleted."

      # 遍歷所有專案空間
      project_directories=($(ssh -i ~/.ssh/$remote_name/rclone-merged root@<your_server_ip> "find /home/httpd/vhosts/* -maxdepth 0 -type d"))
      echo "Copying volume backup file from $remote_name to $backup_directory/$dir_name/$current_time..."

      # 迭代專案目錄
      for project_dir in "${project_directories[@]}"; do
         dir_name=$(basename "$project_dir")
         # 下載文件到新創建的目錄
         rclone copy --ignore-checksum $remote_name:$project_dir/httpdocs/ "$backup_directory/$dir_name/$current_time"
      done

      echo "volume backup file copied successfully."

      echo "Backup process completed successfully."

      read -p "Press Enter to exit"
      ```  

#### 5. **Windows 排程**  

1. 開啟「控制台」，點選「系統及安全性」。
2. 選擇「排程工作」。
3. 點選「建立基本工作」，就可以根據需求來建立排程了。
   
### **資料來源**  

[rclone的sftp/ssh应用实践 - 知乎](https://zhuanlan.zhihu.com/p/441077229)  

[Rclone Documentation](https://rclone.org/docs/#time-option)  

###### tags: `server` `backup`