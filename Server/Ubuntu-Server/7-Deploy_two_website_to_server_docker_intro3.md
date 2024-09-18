# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker Compose 介紹篇（一）

**前言**
===
:::info
本來要繼續介紹 Docker 的一些基本管理的指令和要注意的事項，但因為小弟現在幾乎都是用 Docker Compose 在部署專案，
所以我想先介紹 Docker Compose，然後再繼續介紹其他 Docker 相關的主題。接下來的文章將會詳細介紹 Docker Compose，並示範如何使用它來部署一個專案。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
::: 

**Docker Compose 是什麼?**  
===  

Docker Compose 是一個工具，它讓你可以輕鬆地在你的電腦上管理多個 Docker 容器。你可以使用一個簡單的配置文件（通常是 YAML 格式），指定你想要運行的每個容器的設置，比如映像、環境變量、網絡設置等等。然後，只需運行一個命令，Docker Compose 就會根據你的配置啟動所有容器，並確保它們之間的通信正常。簡而言之，它讓你更輕鬆地管理和運行多個 Docker 容器。  

**Docker Compose 實作**  
===  

今天的範例是使用 Docker Compose 構建一個由 `MongoDB`、`Nginx` 和 `Golang` 組成的專案。  

#### **基本需求**  

1. 準備好一個 Golang 專案
2. 安裝 Docker , Docker-compose。  

#### **基本流程簡述** 

1. 會有三個資料夾，一個是主程式，一個是 Nginx，一個是資料庫，還有一個  docker-compose.yml (設定檔)。專案目錄參考如下:   
  ![Example_Project_Directory](https://i.imgur.com/yW7pGBb.png)  

2. 在 main_app 資料夾內，要新增一個 Dockerfile 和 .dockerignore。  
3. 在 mysql 資料夾內，要新增五隻檔案，包括剛剛提到的 Dockerfile 和一開始準備好的 sql 檔案。 
4. 在 nginx 資料夾內，要新增三隻檔案，除了 Dockerfile 外，還有 nginx 設定檔。
5. 編寫好 docker-compose.yml  
6. 輸入指令 docker compose up 即可完成啟動  

#### **相關設定檔介紹** 

Docker Compose 最重要的就是寫 `docker-compose.yml`，簡化版設定檔參考:  

``` yml
version: "3.7"  # 指定 Docker Compose 文件的版本

services:
    app:
        build: ./main_app  # 指定 Dockerfile 的路徑，用於構建 app 服務的鏡像
        container_name: test_main_app  # 設定容器的名稱為 test_main_app
        restart: always  # 設定容器在退出時自動重啟
        env_file: ./main_app/.env  # 指定環境變量文件的位置
        volumes:
            - ./storage:/var/www/html/storage/  # 將主機的 ./storage 目錄掛載到容器內的 /var/www/html/storage/ 目錄
        depends_on:
            - db  # 設定 app 服務在 db 服務啟動後才啟動
        networks:
            - test_network  # 將 app 服務連接到 test_network 網絡

    nginx:
        build: ./nginx  # 指定 Dockerfile 的路徑，用於構建 nginx 服務的鏡像
        container_name: test_nginx  # 設定容器的名稱為 test_nginx
        restart: always  # 設定容器在退出時自動重啟
        ports:
            - "8087:80"  # 將主機的 8087 端口映射到容器的 80 端口
        depends_on:
            - app  # 設定 nginx 服務在 app 服務啟動後才啟動
        networks:
            - test_network  # 將 nginx 服務連接到 test_network 網絡

    db:
        build: ./mysql  # 指定 Dockerfile 的路徑，用於構建 db 服務的鏡像
        container_name: test_house_db  # 設定容器的名稱為 test_house_db
        restart: always  # 設定容器在退出時自動重啟
        volumes:
            - ./database:/var/lib/mysql  # 將主機的 ./database 目錄掛載到容器內的 /var/lib/mysql 目錄
        ports:
            - "3308:3306"  # 將主機的 3308 端口映射到容器的 3306 端口
        networks:
            - test_network  # 將 db 服務連接到 test_network 網絡

networks:
    test_network:  # 定義 test_network 網絡，所有服務都將連接到這個網絡

```

### **資料來源**  

[Bind mounts | Docker Docs](https://docs.docker.com/storage/bind-mounts/)  
[Volumes | Docker Docs](https://docs.docker.com/storage/volumes/)  
[Docker volumes 教學 - 從不熟到略懂 - MyApollo](https://myapollo.com.tw/blog/docker-volumes/) 


###### tags: `server` `docker`