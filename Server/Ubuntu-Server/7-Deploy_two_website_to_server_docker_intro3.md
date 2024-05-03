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

今天的範例是使用 Docker Compose 構建一個由 `MySQL`、`Nginx` 和 `Laravel` 組成的網站。Docker Compose 最重要的就是寫 `docker-compose.yml`，簡化版設定檔參考:  

``` yml
version: "3.7"

services:
    app:
        build: ./main_app
        container_name: test_main_app
        restart: always
        env_file: ./main_app/.env
        volumes:
            - ./storage:/var/www/html/storage/
        depends_on:
            db
        networks:
            - test_network
    nginx:
        build: ./nginx
        container_name: test_nginx
        restart: always
        ports:
            - "8087:80"
        depends_on:
            - app
        networks:
            - test_network

    db:
        build: ./mysql
        container_name: test_house_db
        restart: always
        volumes:
            - ./database:/var/lib/mysql
        ports:
            - "3308:3306"
        networks:
            - test_network
networks:
    test_network:
```

### **資料來源**  

[Bind mounts | Docker Docs](https://docs.docker.com/storage/bind-mounts/)  
[Volumes | Docker Docs](https://docs.docker.com/storage/volumes/)  
[Docker volumes 教學 - 從不熟到略懂 - MyApollo](https://myapollo.com.tw/blog/docker-volumes/) 


###### tags: `server` `docker`