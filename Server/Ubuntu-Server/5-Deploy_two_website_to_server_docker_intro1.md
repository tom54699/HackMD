# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker 介紹篇（一）

**前言**
===
:::info
因為小弟剛接觸 Docker 時，也撞牆了很久，看了很多文章也還是似懂非懂。直到實在操作過幾次後，才慢慢越來越了解。所以我這邊想用超級白話的方式來說明 Docker 可以帶給我們什麼便利。這邊的介紹不講細節，也可能沒有很嚴謹，主要目的是想給完全沒有碰過的人一個很簡單的概念。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
:::  

**Docker 是什麼?**  
===

Docker 是一個容器化的工具。什麼是『 容器化 』?  

如果大家有聽過 ++VirtualBox++ 虛擬機軟體，或是有在打遊戲用過++沙盒++或是電腦上開的 ++手遊模擬器++。這些東西都算是虛擬機，它會在電腦中隔離出一個空間，並在裡面裝不同的系統環境。像是你可以用 VirtualBox 在 windows 電腦開一個虛擬機，然後在裡面安裝 macOS 環境。 或是手遊模擬器通常就是開出一個虛擬機，裡面安裝安卓環境讓我們可以執行 app。  

那還是沒有提到容器是什麼啊？ 容器就是一個更輕量的虛擬化方案，跟虛擬機最大的不同是容器中只安裝應用程式執行時所需要的環境即可，不需要安裝一個全新的系統。所以運行起來會更快速，更節省資源，也更容易部署。

**為什麼要用 Docker ？**  
===

大家可能會想，我不用 Docker 也可以部署專案。如果用 Docker 雖然相比於虛擬機更輕量化，但多少還是會需要消耗更多資源，還加上學習還有維護成本，我為什麼需要用 Docker ?  

我這邊想分享幾個我使用的經驗，來分享為什麼我會愛上用 Docker：  

1. **移轉專案很方便**  
    當需要將專案移動到另一台伺服器時，只需在新的伺服器上安裝 Docker，然後將專案的 Docker 設定檔複製過去即可。不需要重新安裝專案所需的環境，大大簡化了部署流程，並節省了時間和精力。

2. **更好管理多個專案**  
    當一台伺服器需要管理多個不同專案時，往往會面臨環境或資料庫版本不一致的問題，這可能導致管理上的混亂。使用 Docker 技術可以將每個專案都容器化，這樣各專案的環境將被隔離開來，互不干擾。這不僅使得專案管理更加清晰，也更容易進行環境配置和管理。  

3. **方便部署**  
    透過 Docker 和 Github Action 建立一套 CI/CD 機制，我只要 `git push` 就可以成功部署到伺服器，連程式碼都不需要放到伺服器上。  

4. **統一開發環境**  
    現在 Vscode 也有提供虛擬化的開發容器，主要也是搭配 Docker 使用。可以定義好多人的開發環境，確保開發環境一致，從而提高開發效率和減少問題。

5. **安裝各種應用程式**  
    過去有時候在研究新技術的時候，就會安裝不同的應用程式還有需求的環境。後續解除安裝時，也不一定會刪得很乾淨。現在可以的話，我都會 Docker 直接啟動這些程式，不僅快速，也不用擔心污染電腦環境。  

**Docker 要如何使用？**
===

先從簡單的來，我們今天的目標就用 Docker 啟動一個 MySQL 就好。  

### **安裝 Docker (適用於Ubuntu 22.04.3 LTS)**  

1. **更新 ubuntu**  

    ``` bash
    sudo apt-get update
    ```  
2. **安裝 Docker 所需要相關套件** 

    ``` bash
    sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
    ```  
3. **新增 Docker 官方 GPG key**  

    ``` bash
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```  
4. **將 Docker 的官方套件庫添加到 Ubuntu 系統的 apt 軟體包管理器**

    ``` bash
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```  
5. **更新 ubuntu**

    ``` bash
    sudo apt-get update
    ```  
6.  **安装 Docker** 

    ``` bash
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```  
7. **查看 Docker 版本 ＆ 狀態** 

    ``` bash
    sudo docker version #查看版本
    systemctl status docker #查看狀態
    sudo systemctl start docker #啟動
    sudo systemctl enable docker #每次系統重啟時自動啟動
    ```  
8. **測試 Docker 是否運作正常**  

    ``` bash
    sudo docker run hello-world
    ```  
    ![Docker run hello-world](https://i.imgur.com/GEiq465.png)  

    出現這樣就代表成功囉。  

### **準備 MySQL 容器的 Dockerfile**  

Dockerfile 是用來定義容器內部環境和設定檔。透過 Dockerfile，您可以指定容器需要的基礎映像檔、應用程式、和其他依賴庫，並設定運行時的環境變數、指令等。Docker 會根據 Dockerfile 來建立映像檔 (image)。 有了 Docker image 後，就可以根據映像檔來建立容器。接下來我們就來試做看看：  

1. **建立一個 Dockerfile 的檔案，設定檔內容如下：**

    ``` bash
    # 使用官方 MySQL 映像作為基礎映像
    FROM mysql:latest

    # 設置 MySQL root 密碼
    ENV MYSQL_ROOT_PASSWORD=testing

    # 在容器內建立一個新的資料庫
    ENV MYSQL_DATABASE=mydatabase

    # 將 SQL 檔案複製到容器內的 /docker-entrypoint-initdb.d/ 中，以初始化資料庫
    COPY mydatabase.sql /docker-entrypoint-initdb.d/
    ```  
2. **準備一個 sql 檔案，放在 Dockerfile 同一個資料夾底下**

    ![MySQL Docker Practice ](https://i.imgur.com/3j5wsYe.png)  

3. **接下來根據 Dockerfile 來建立 image，請在該資料夾底下執行指令:** 

    ``` bash
    docker build -t tom54699/mysql_practice:1.0 .
    ```  
    `-t` 選項用於指定映像的標籤（tag），通常用來標識映像的版本或其他相關資訊。通常，映像檔的格式為 `<docker_account>/<repository>:<tag>`。

    :::warning
    如果沒有 Docker 帳號建議去辦一個，未來映像檔可以上傳到 DockerHub 的 Repository去做存放和版本控管。未來系列文章中會再提到。
    :::  

4. **查看 Docker image 狀態**

    ``` bash
    docker image ls #查看目前 Docker image
    ```  
    應該會成功看到映像檔的各種資訊。

    ![Docker Image Command](https://i.imgur.com/YmGiVeQ.png)  

5. **使用 Docker image 建立容器**  

    ``` bash
    docker run -d --name mysql-container -p 3307:3306 tom54699/mysql_practice:1.0  
    ```  
    指令解釋：  
    - docker run: 這是執行 Docker 容器的命令。

    - -d: 這是一個選項，表示在後台運行容器。

    - --name mysql-container: 這是指定容器的名稱，這裡命名為 mysql-container。

    - -p 3306:3306: 這是指定容器與主機之間的端口映射。左邊的 3307 是主機的端口，右邊的 3306 是容器的端口。這個指令表示將主機的 33076 端口映射到容器的 3306 端口，這樣主機上的應用程序就可以通過主機的 3307 端口連接到容器中運行的 MySQL 服務。

    - tom54699/mysql_practice:1.0: 這是指定要運行的 Docker 映像。  

7. **查看 Docker 容器狀態**  

    ``` bash
    docker container ls
    ```  
    ![Docker Container ls](https://i.imgur.com/mhSCVtO.png)  

    目前我們已經成功啟動一個 MySQL 的 Docker 容器了。

8. **實際操作 MySQL** 

    目前操作都在 Server 裡面，沒有圖形化介面，要用指令進去。

    ``` bash
    docker exec -it mysql-container mysql -uroot -p
    ``` 
    ![Connect to Mysql Docker Container](https://i.imgur.com/HVVMAfb.png)  

    若是要用圖形化介面，可以用本地的 MySQL Workbench 透過 `Standard TCP/IP over SSH` 或其實用 `Standard(TCP/IP)` 應該也可以連線至資料庫，至於為什麼後面會說到。

    ![MySQL Workbench connect to server Docker MySQL](https://i.imgur.com/Ibp50Rq.png)  

    :::danger
    Docker 容器開放的埠號並不受到主機防火牆的保護，因為容器在獨立的網絡環境中運行，並且可以直接訪問主機上的所有端口。這意味著，即使主機的防火牆沒有開放某個特定的埠號，只要容器內部開放了相應的埠號並且在主機上映射了這個埠號，外部仍然可以通過 IP 地址和埠號直接訪問到容器。  

    未來我會盡快出一篇來討論如何將 Docker 納入防火牆的規則，納入防火牆後就一定要用 `Standard TCP/IP over SSH` 才能從本地連到 Server 的 MySQL 容器了。
    :::

:::success
這篇文章主要就介紹到這邊，下一篇預計會繼續介紹 Docker 相關的主題，會從今天建立好的 MySQL Container 在延伸一些 Docker 其他的概念。
::: 

###### tags: `server` `docker`







