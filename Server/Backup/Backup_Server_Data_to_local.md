# 將伺服器檔案備份至本地(Windows)

:::info
目標: 把雲端 Linux Server 的 專案 SQL 檔案 和靜態資源的資料夾備份到本地 Window 主機
:::

- 主要還是依靠 rclone，所以先在 window 安裝 rclone
    
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
    
    參考:
    
    1. [https://hackmd.io/@Restforlife/ryltomajfi](https://hackmd.io/@Restforlife/ryltomajfi)
    2. [https://hackmd.io/@adeliae/H1ZRCUfaU](https://hackmd.io/@adeliae/H1ZRCUfaU)
    3. [https://site.tdccc.com.tw/p202223/](https://site.tdccc.com.tw/p202223/)
- 接下來，為了省事在 window 用 linux 指令，去安裝 Git bash
    - Git bash簡介
        
        Git Bash 是一個為 Windows 用戶提供的 Git 命令行界面的工具。它基於 Bash shell，使得在 Windows 系統上使用 Git 命令變得更加方便。以下是 Git Bash 的一些主要特點和功能：
        
        1. **Bash Shell：** Git Bash 使用 Bash shell，這是一種類 Unix shell，為使用者提供了一個強大的命令行環境，讓用戶能夠通過命令行界面來執行 Git 命令和其他 Unix/Linux 命令。
        2. **Git 命令行工具：** Git Bash 預先安裝了 Git 的命令行工具，這使得用戶可以直接在命令行中使用 Git 命令，如 **`git clone`**、**`git commit`**、**`git push`** 等。
        3. **Unix 工具：** 除了 Git 命令行工具外，Git Bash 還包含了一些常見的 Unix 工具，如 **`ls`**、**`cp`**、**`mv`**、**`rm`** 等，這使得在命令行中執行標準的 Unix 命令變得更加容易。
        4. **支援 SSH：** Git Bash 支援 SSH，這使得你可以使用 SSH 金鑰進行安全的 Git 操作，例如克隆或推送存儲庫。
        5. **環境變數：** Git Bash 允許你在 Bash shell 中設置環境變數，這有助於自定義你的開發環境。
        6. **支援 Linux/Unix 命令：** 由於 Git Bash 基於 Bash shell，你可以使用類似於 Linux/Unix 的命令，這對於熟悉 Unix 系統的開發人員來說會感到更為熟悉。
        7. **文本編輯器：** Git Bash 通常包含一個簡單的文本編輯器，例如 Vim 或 Nano，這使得在命令行中進行一些簡單的文本編輯變得更容易。
    - 安裝參考
        
        [https://learn.microsoft.com/zh-tw/contribute/content/get-started-setup-tools?pivots=windows-os-pivot-selection](https://learn.microsoft.com/zh-tw/contribute/content/get-started-setup-tools?pivots=windows-os-pivot-selection)
        
- 接下來用 Git bash 來做 rclone 的設定，這次要建立 ssh/sftp 連線
    - 先建立 ssh key
        
        ```bash
        ssh-keygen -q -t rsa -b 4096 -C "rclone key" -N "" -f ~/.ssh/rclone
        cd ~/.ssh/
        cat rclone* > rclone-merged
        ```
        
    - 把剛剛生成的 pub key 丟到 server
        
        ```bash
        ssh-copy-id -i ~/.ssh/rclone.pub root@192.168.32.33
        ```
        
    - 進行 rclone 設定
        
        ```bash
        rclone config
        No remotes found - make a new one
        n) New remote
        s) Set configuration password
        q) Quit config
        n/s/q> n
        name> Server_backup
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
        Storage> sftp  # or 33
        Option host.
        SSH host to connect to.
        E.g. "example.com".
        Enter a string value. Press Enter for the default ("").
        host> 192.168.32.33
        Option user.
        SSH username, leave blank for current username, ethan.
        Enter a string value. Press Enter for the default ("").
        user> root
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
        key_pem> #回车
        Option key_file.
        Path to PEM-encoded private key file.
        Leave blank or set key-use-agent to use ssh-agent.
        Leading `~` will be expanded in the file name as will environment variables such as `${RCLONE_CONFIG_DIR}`.
        Enter a string value. Press Enter for the default ("").
        key_file> ~/.ssh/rclone-merged
        Option key_file_pass.
        The passphrase to decrypt the PEM-encoded private key file.
        Only PEM encrypted key files (old OpenSSH format) are supported. Encrypted keys
        in the new OpenSSH format can't be used.
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
        pubkey_file> #回车
        Option key_use_agent.
        When set forces the usage of the ssh-agent.
        When key-file is also set, the ".pub" file of the specified key-file is read and only the associated key is
        requested from the ssh-agent. This allows to avoid `Too many authentication failures for *username*` errors
        when the ssh-agent contains many keys.
        Enter a boolean value (true or false). Press Enter for the default ("false").
        key_use_agent> #回车
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
        use_insecure_cipher> #回车
        Option disable_hashcheck.
        Disable the execution of SSH commands to determine if remote file hashing is available.
        Leave blank or set to false to enable hashing (recommended), set to true to disable hashing.
        Enter a boolean value (true or false). Press Enter for the default ("false").
        disable_hashcheck> #回车
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
        
        這樣 rclone 的 連線就建立完成
        
    - .sh 檔案 編寫
        - shell 檔案
            
            [Plesk_223_27_34_204_sql_backup.sh](%E5%82%99%E4%BB%BDServer%20%E6%AA%94%E6%A1%88%E8%87%B3%E6%9C%AC%E5%9C%B0%20c9c07dd8cb804fe9acdab80442ae18bb/Plesk_223_27_34_204_sql_backup.sh)
            
            [Plesk_223_27_34_204_volume_backup.sh](%E5%82%99%E4%BB%BDServer%20%E6%AA%94%E6%A1%88%E8%87%B3%E6%9C%AC%E5%9C%B0%20c9c07dd8cb804fe9acdab80442ae18bb/Plesk_223_27_34_204_volume_backup.sh)
            
            [Plesk_223_27_43_37_sql_backup.sh](%E5%82%99%E4%BB%BDServer%20%E6%AA%94%E6%A1%88%E8%87%B3%E6%9C%AC%E5%9C%B0%20c9c07dd8cb804fe9acdab80442ae18bb/Plesk_223_27_43_37_sql_backup.sh)
            
            [Plesk_223_27_43_37_volume_backup.sh](%E5%82%99%E4%BB%BDServer%20%E6%AA%94%E6%A1%88%E8%87%B3%E6%9C%AC%E5%9C%B0%20c9c07dd8cb804fe9acdab80442ae18bb/Plesk_223_27_43_37_volume_backup.sh)
            
        - sql
            
            ```bash
            #!/bin/bash
            
            # 設定遠端連接名稱
            remote_name="Plesk_223_27_43_37"
            
            # 設置本地保存目錄
            local_directory="/c/backup/sql_backup"
            
            echo "Starting SQL backup process..."
            
            # 創建以日期命名的目錄
            current_date=$(date +"%Y-%m-%d")
            current_time=$(date +"%Y-%m-%d-%H-%M")
            backup_directory="$local_directory/$remote_name/$current_date"
            #mkdir -p "$backup_directory"
            
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
            
        - docker volume
            
            ```bash
            #!/bin/bash
            
            # 設定遠端連接名稱
            remote_name="Plesk_223_27_43_37"
            
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
            
            # 自動遍歷抓取所有專案空間
            project_directories=($(ssh -i ~/.ssh/$remote_name/rclone-merged root@223.27.43.37 "find /var/www/vhosts/* -maxdepth 0 -type d ! -name 'system' ! -name 'chroot' ! -name 'default' ! -name 'athenabeta.com.tw' ! -name 'demo.athenademo.com.tw'"))
            echo "Copying volume backup file from $remote_name to $backup_directory/$dir_name/$current_time..."
            
            # 迭代專案目錄
            for project_dir in "${project_directories[@]}"; do
                dir_name=$(basename "$project_dir")
                echo "$dir_name start back up."
                # 如果目录名为 jygesportal.com，则使用不同的路径下载文件
                if [ "$dir_name" = "jygesgportal.com" ]; then
                    rclone copy --ignore-checksum "$remote_name:$project_dir/httpdocs/jyg" "$backup_directory/$dir_name/$current_time"
                else
                    # 下載文件到新創建的目錄
                    rclone copy --ignore-checksum "$remote_name:$project_dir/httpdocs/storage" "$backup_directory/$dir_name/$current_time"
                fi
            done
            
            echo "volume backup file copied successfully."
            
            echo "Backup process completed successfully."
            
            read -p "Press Enter to exit"
            ```
            
    
    最後加入 window 排程即可
    
    - 有可能出現的問題
        1. 檔案大小不同步，可能有備份到log 
            
            ```bash
            解決方案: --ignore-checksum
            ```
            
            參考: 
            
            [https://github.com/rclone/rclone/issues/5366](https://github.com/rclone/rclone/issues/5366)
            
    
    參考:
    
    1. [https://zhuanlan.zhihu.com/p/441077229](https://zhuanlan.zhihu.com/p/441077229)