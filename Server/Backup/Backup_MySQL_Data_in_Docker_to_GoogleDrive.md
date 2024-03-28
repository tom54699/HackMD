# 定期異地備份 Docker 中的 MySQL 資料至 Google Drive

- 首先最重要的是 Docker 備份 MySQL 指令
    
    ```bash
    docker exec $container_name mysqldump -u$db_user -p$db_password your_database_name > /tmp/db_backup.sql
    ```
    
- 用 rclone 連線到 Google Drive
    - rclone 簡單介紹
        
        **rclone** 是一個用於與多種不同雲端存儲提供商進行數據同步和轉移的命令行工具。它允許您管理和操作不同雲端存儲服務中的文件和數據，並且可以在本地系統和不同雲端存儲之間進行文件傳輸。
        
        以下是 rclone 的一些主要功能和用途：
        
        1. **支援多種雲端存儲服務**：rclone 可以與許多不同的雲端存儲提供商集成，包括 Google Drive、Amazon S3、Dropbox、Microsoft OneDrive、Box、以及許多其他類型的存儲服務。
        2. **數據同步和備份**：您可以使用 rclone 來同步本地文件和文件夾與雲端存儲之間的數據，以進行備份或確保數據同步。
        3. **遠程文件管理**：rclone 允許您在不同雲端存儲服務中進行文件管理，包括上傳、下載、刪除、移動文件等操作。
        4. **加密和壓縮**：rclone 支援數據的加密和壓縮，以保護敏感信息或節省存儲空間。
        5. **配置靈活**：您可以通過 rclone 配置文件進行靈活的配置，以設置不同雲端存儲服務的訪問權限和參數。
        6. **跨平台**：rclone 可在多種不同操作系統上運行，包括 Linux、macOS 和 Windows。
        7. **強大的命令行工具**：rclone 提供了一個豐富的命令行工具集，用於執行各種操作，並支援腳本化和自動化任務。
        
        總之，rclone 是一個強大的工具，可幫助使用者輕鬆管理和操作不同雲端存儲服務中的數據。無論是備份、同步還是文件管理，rclone 都為這些任務提供了便捷的解決方案。
        
    1. Server 和本地都先安裝 rclone
    
         ```bash
         # centos 安裝指令
         curl https://rclone.org/install.sh | bash
         
         # MacOs 安裝指令
         brew install reclone
         ```
    
    2. 開始進行設定
    
         ```bash
         rclone config
         ```
    
    3. 設定細項範例
        - 階段一，新增空間和命名
            
            ```bash
            No remotes found - make a new one
            n) New remote
            s) Set configuration password
            q) Quit config
            n/s/q> n (選 n 新增雲端空間)
            name> google_drive (命名 google_drive)
            ```
            
        - 階段二，選擇你要連接的第三方空間
            
            ```bash
            Option Storage.
            Type of storage to configure.
            Choose a number from below, or type in your own value.
             1 / 1Fichier
               \ (fichier)
             2 / Akamai NetStorage
               \ (netstorage)
             3 / Alias for an existing remote
               \ (alias)
             4 / Amazon Drive
               \ (amazon cloud drive)
             5 / Amazon S3 Compliant Storage Providers including AWS, Alibaba, ArvanCloud, Ceph, China Mobile, Cloudflare, GCS, DigitalOcean, Dreamhost, Huawei OBS, IBM COS, IDrive e2, IONOS Cloud, Leviia, Liara, Lyve Cloud, Minio, Netease, Petabox, RackCorp, Scaleway, SeaweedFS, StackPath, Storj, Synology, Tencent COS, Qiniu and Wasabi
               \ (s3)
             6 / Backblaze B2
               \ (b2)
             7 / Better checksums for other remotes
               \ (hasher)
             8 / Box
               \ (box)
             9 / Cache a remote
               \ (cache)
            10 / Citrix Sharefile
               \ (sharefile)
            11 / Combine several remotes into one
               \ (combine)
            12 / Compress a remote
               \ (compress)
            13 / Dropbox
               \ (dropbox)
            14 / Encrypt/Decrypt a remote
               \ (crypt)
            15 / Enterprise File Fabric
               \ (filefabric)
            16 / FTP
               \ (ftp)
            17 / Google Cloud Storage (this is not Google Drive)
               \ (google cloud storage)
            18 / Google Drive
               \ (drive)
            19 / Google Photos
               \ (google photos)
            20 / HTTP
               \ (http)
            21 / Hadoop distributed file system
               \ (hdfs)
            22 / HiDrive
               \ (hidrive)
            23 / In memory object storage system.
               \ (memory)
            24 / Internet Archive
               \ (internetarchive)
            25 / Jottacloud
               \ (jottacloud)
            26 / Koofr, Digi Storage and other Koofr-compatible storage providers
               \ (koofr)
            27 / Local Disk
               \ (local)
            28 / Mail.ru Cloud
               \ (mailru)
            29 / Mega
               \ (mega)
            30 / Microsoft Azure Blob Storage
               \ (azureblob)
            31 / Microsoft OneDrive
               \ (onedrive)
            32 / OpenDrive
               \ (opendrive)
            33 / OpenStack Swift (Rackspace Cloud Files, Blomp Cloud Storage, Memset Memstore, OVH)
               \ (swift)
            34 / Oracle Cloud Infrastructure Object Storage
               \ (oracleobjectstorage)
            35 / Pcloud
               \ (pcloud)
            36 / PikPak
               \ (pikpak)
            37 / Proton Drive
               \ (protondrive)
            38 / Put.io
               \ (putio)
            39 / QingCloud Object Storage
               \ (qingstor)
            40 / Quatrix by Maytech
               \ (quatrix)
            41 / SMB / CIFS
               \ (smb)
            42 / SSH/SFTP
               \ (sftp)
            43 / Sia Decentralized Cloud
               \ (sia)
            44 / Storj Decentralized Cloud Storage
               \ (storj)
            45 / Sugarsync
               \ (sugarsync)
            46 / Transparently chunk/split large files
               \ (chunker)
            47 / Union merges the contents of several upstream fs
               \ (union)
            48 / Uptobox
               \ (uptobox)
            49 / WebDAV
               \ (webdav)
            50 / Yandex Disk
               \ (yandex)
            51 / Zoho
               \ (zoho)
            52 / premiumize.me
               \ (premiumizeme)
            53 / seafile
               \ (seafile)
            Storage> 18 (打入 Google Drive 對應序號 18)
            ```
            
        - 階段三，一些可以略過的步驟
            
            ```bash
            Google Application Client Id
            Leave blank normally.
            Enter a string value. Press Enter for the default ("").
            client_id> (通常直接按 enter 不需填入，也可以填，但要去google設定)
            Google Application Client Secret
            Leave blank normally.
            Enter a string value. Press Enter for the default ("").
            client_secret> (通常直接按 enter 不需填入，也可以填，但要去google設定 )
            Scope that rclone should use when requesting access from drive.
            Enter a string value. Press Enter for the default ("").
            Choose a number from below, or type in your own value
            1 / Full access all files, excluding Application Data Folder.
             \ "drive"
            2 / Read-only access to file metadata and file contents.
             \ "drive.readonly"
             / Access to files created by rclone only.
            3 | These are visible in the drive website.
             | File authorization is revoked when the user deauthorizes the app.
             \ "drive.file"
             / Allows read and write access to the Application Data folder.
            4 | This is not visible in the drive website.
             \ "drive.appfolder"
             / Allows read-only access to file metadata but
            5 | does not allow any access to read or download file content.
             \ "drive.metadata.readonly"
            scope> 1 (選 1 獲取全部權限)
            Option service_account_file.
            Service Account Credentials JSON file path.
            Leave blank normally.
            Needed only if you want use SA instead of interactive login.
            Leading `~` will be expanded in the file name as will environment variables such as `${RCLONE_CONFIG_DIR}`.
            Enter a value. Press Enter to leave empty.
            service_account_file> (直接按 enter 不需填入)
            Edit advanced config? (y/n)
            y) Yes
            n) No
            y/n> n (選 n 不需要進階設定)
            ```
            
        - 階段四，重要的認證步驟
            
            如果你在server上面設定，你就選n，因為你沒辦法在server上開瀏覽器。
            
            如果在本地就選y。
            
            ```bash
            Use web browser to automatically authenticate rclone with remote?
             * Say Y if the machine running rclone has a web browser you can use
             * Say N if running rclone on a (remote) machine without web browser access
            If not sure try Y. If Y failed, try N.
            
            y) Yes (default)
            n) No
            y/n> n
            Option config_token.
            For this to work, you will need rclone available on a machine that has
            a web browser available.
            For more help and alternate methods see: https://rclone.org/remote_setup/
            Execute the following on the machine with the web browser (same rclone
            version recommended):
            	rclone authorize "drive" "密鑰"
            Then paste the result.
            Enter a value.
            config_token>
            ```
            
            選 n 後，你就需要在本地輸入 上面給的指令
            
            ```bash
            rclone authorize "drive" "密鑰"
            2023/09/22 10:58:27 NOTICE: If your browser doesn't open automatically 
            go to the following link: http://127.0.0.1:53682/auth?state=xxxx
            ```
            
            然後去瀏覽器輸入他給你的網址，主要目的是讓你用你的google帳號給rclone權限。
            
            成功賦予權限後
            
            ![Success Picture](https://i.imgur.com/vc9wXXl.png)
            
            然後在 terminal 就會出現 ，複製他的密鑰，然後貼回去 server。
            
            ```bash
            # 本地 terminal
            2023/09/22 10:58:27 NOTICE: Log in and authorize rclone for access
            2023/09/22 10:58:27 NOTICE: Waiting for code...
            2023/09/22 10:58:32 NOTICE: Got code
            Paste the following into your remote machine --->
            xxxxxx這段就是config_tokenxxxxxxx
            <---End paste
            ```
            
            ```bash
            # server terminal
            config_token> 輸入密鑰
            ```
            
        - 階段五，最後確認資料
            
            ```bash
            Configure this as a Shared Drive (Team Drive)?
            
            y) Yes
            n) No (default)
            y/n> n (選 n，通常沒有用 team drive)
            
            No Shared Drives found in your account
            Configuration complete.
            Options:
            - type: drive
            - scope: drive
            - token: {"access_token":"xxx","token_type":"Bearer","refresh_token":"xxx":"2023-09-22T11:58:31.919923+08:00"}
            Keep this "google_drive" remote?
            y) Yes this is OK (default)
            e) Edit this remote
            d) Delete this remote
            y/e/d> y
            
            Current remotes:
            
            Name                 Type
            ====                 ====
            google_drive  drive
            ```
            
        - 階段六，確認遠端連線
            
            ```bash
            rclone listremotes. # 可以查看已建立的遠端連線
            
            rclone config # 可以重新編輯設定文件
            
            rclone ls 遠端名稱 # 舉例：rclone ls google_drive: 會列出所有檔案
            ```
            
        - 階段七，遠端建立要備份的資料夾
            
            ```bash
            rclone mkdir 遠端連線名稱:遠端路徑/新資料夾名稱。
            
            # 舉例
            rclone mkdir google_drive:Backup_Folder
            ```
            
    
- 知道備份指令後，就可以開始寫腳本（記得給執行權限）
    
    ```bash
    #!/bin/bash
    
    # Docker容器名稱
    project_name="您的專案名稱"
    container_name="您的Docker容器名稱"
    
    # 資料庫使用者名稱和密碼
    db_user="您的資料庫使用者"
    db_password="您的資料庫密碼"
    db_name="您的資料庫名稱"
    
    # 遠端雲端儲存桶/目錄路徑
    remote_name="您rclone設定的遠端連線名稱"
    cloud_storage_path="存放的雲端目錄資料夾"
    
    echo "$project_name start backup project."
    
    # 刪除三個月前的備份文件
    echo "Deleting expired files from: $remote_name:$cloud_storage_path/$project_name"
    rclone delete $remote_name:$cloud_storage_path/$project_name --min-age 3M
    echo "Expired files deleted."
    
    # 刪除空的源目錄
    echo "Removing empty source directories from: $remote_name:$cloud_storage_path/$project_name"
    rclone rmdirs $remote_name:$cloud_storage_path/$project_name
    echo "Empty source directories removed."
    
    # 設定日期格式
    current_date=$(date +"%Y-%m-%d")
    current_time=$(date +"%Y-%m-%d-%H-%M")
    
    # 備份資料庫
    echo "Backing up database to /tmp/${db_name}_backup.sql"
    docker exec $container_name mysqldump -u$db_user -p$db_password $db_name > /tmp/${db_name}_backup.sql
    echo "Database backup completed."
    
    # 上傳備份至遠端雲端儲存
    echo "Uploading backup to: $remote_name:$cloud_storage_path/$project_name/$current_date/$current_time"
    rclone copy /tmp/${db_name}_backup.sql $remote_name:$cloud_storage_path/$project_name/$current_date/$current_time
    echo "Backup uploaded."
    
    # 清理臨時備份檔案
    echo "Cleaning up temporary backup file."
    rm /tmp/${db_name}_backup.sql
    echo "Temporary backup file cleaned up."
    ```
    
- 寫排程，讓 server 定時備份
    - 因為 server 有很多專案，所以寫了一個主腳本來管理，以後有新專案都要把專案目錄加入陣列
    主腳本目錄：/usr/local/bin/backup/main_docker_db_backup.sh
        
        ```bash
        #!/bin/bash
        
        # 定義專案目錄的陣列
        project_directories=("/var/www/vhosts/romantic-cannon.223-27-43-37.plesk.page/httpdocs/")
        
        # 迭代專案目錄
        for project_dir in "${project_directories[@]}"; do
            # 切換到專案目錄
            cd "$project_dir" || exit 1
        
            # 執行專案目錄內的backup.sh
            ./db_backup.sh
            # 切換回主目錄
            cd - || exit 1
        done
        ```
        
    - 改版後，不用自己新增專案目錄，會自動抓取
        
        ```php
        # 搜尋 /home/httpd/vhosts 下的專案目錄，排除指定的資料夾
        project_directories=($(find /var/www/vhosts/* -maxdepth 0 -type d ! -name "system" ! -name "chroot" ! -name "default" ! -name "xxx.com.tw"))
        # 迭代專案目錄
        for project_dir in "${project_directories[@]}"; do
            # 切換到專案目錄
            cd "$project_dir/httpdocs" || exit 1
            # 執行專案目錄內的backup.sh
            ./db_backup.sh
            # 切換回主目錄
            cd - || exit 1
        done
        ```
        
    - 可以使用指令
        
        ```bash
        # 給予主腳本執行權限
        chmod +x /path/to/your/script.sh
        ```
        
        要設定 Linux 伺服器上的排程，以定期執行您的腳本，您可以使用 cron 工具。Cron 允許您在指定的時間和日期執行腳本或命令。以下是如何設定 cron 排程來執行您的腳本：
        
        1. 打開終端機並使用以下命令來編輯 cron 排程：
            
            ```
            crontab -e
            ```
            
        2. 這將打開一個文本編輯器，用於編輯cron排程。如果是第一次編輯 cron，您可能需要選擇文本編輯器（例如 nano，vim等）。
        3. 在文本編輯器中，添加以下行來設定 cron 排程，每天定時執行您的腳本。這個例子將在每天的凌晨2點執行腳本：
            
            ```bash
            0 2 * * * /bin/bash /path/to/your/script.sh
            ```
            
            請確保替換 **`/path/to/your/script.sh`** 為您腳本.sh 的實際路徑。
            
            這個cron表達式的各個字段的含義如下：
            
            - 第1個字段（0）：分鐘（0-59）
            - 第2個字段（2）：小時（0-23）
            - 第3個字段（*）：日期（1-31）
            - 第4個字段（*）：月份（1-12）
            - 第5個字段（*）：星期幾（0-7，其中0和7都代表星期天）
        4. 保存文件並退出文本編輯器。
        5. cron將在指定的時間自動執行您的腳本。您可以使用以下命令查看當前的cron排程：
            
            ```
            crontab -l
            ```
            
            您應該能夠看到您剛剛添加的排程。
            
        
        請注意，確保腳本.sh 的執行權限已經設置，以便cron可以執行它。您可以使用以下命令添加執行權限：
        
        ```bash
        chmod +x /path/to/your/script.sh
        ```
        
        這樣，cron 將在每天的凌晨2點執行您的腳本，執行備份和上傳操作。您可以根據需求調整 cron 排程中的時間。
        
    - 或是 Server 有自帶圖形化介面的排程設定
        
        Plesk 的排程在 → 工具與設定 → 工作排程 ( Cron 工作)
        
- 參考網站
    1. [https://www.wongwonggoods.com/all-posts/linux/linux-file/rclone-mount-google-drive/#透過終端機下載](https://www.wongwonggoods.com/all-posts/linux/linux-file/rclone-mount-google-drive/#%E9%80%8F%E9%81%8E%E7%B5%82%E7%AB%AF%E6%A9%9F%E4%B8%8B%E8%BC%89)
    2. [https://iservice.nchc.org.tw/download_file.php?f=5G7AkAdcLrsExQugX1yjaY6h1__l9FM_rN4sIIhOhaKvd4nSrAmTh9Ifu5_uLRnsvtXStxkPNcGXxFoLkEIJOw](https://iservice.nchc.org.tw/download_file.php?f=5G7AkAdcLrsExQugX1yjaY6h1__l9FM_rN4sIIhOhaKvd4nSrAmTh9Ifu5_uLRnsvtXStxkPNcGXxFoLkEIJOw)
    3. [https://rclone.org/docs/#time-option](https://rclone.org/docs/#time-option)
    4. [https://rclone.cn/docs.html](https://rclone.cn/docs.html)