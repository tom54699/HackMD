# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker 介紹篇（二）

**前言**
===
:::info
再上一篇文章中，我們大致了解了 Docker，也成功建立了一個 MySQL 容器，大家應該對 `Dockerfile`、`container`、`image` 有一點概念了。如果還是不太懂，可以多多爬文，一開始接觸有個撞牆期很正常，建議實作可以比較快熟悉上手。 今天主要會簡單介紹 `Volume` 和簡單的實作。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
::: 

**Docker Volume 是什麼?**  
===  

Docker Volume 是 Docker 中用於持久性數據存儲的機制之一。在 Docker 容器中，文件系統是暫時性的，這意味著當容器終止時，任何在容器內創建的文件或數據都會丟失。為了解決這個問題，Docker 提供了 Volume，它允許將數據持久化存儲在主機上，即使容器終止也能保留。  

簡單的以上次建立的 MySQL 容器來舉例，當我們成功建立了 MySQL 容器但未建立相對應的 Volume 時，這將導致在容器終止或重新啟動時，所有在容器內部存儲的資料都會丟失。  

這就是為什麼需要使用 Volume 的原因。通過建立 Volume，我們可以將容器內的數據持久化存儲在主機上，從而確保即使容器終止，數據仍然得以保留。此外，當容器內的數據發生更改時，Volume 會自動將這些變化同步到主機上。這意味著你不需要手動同步數據，Volume 會負責確保數據在容器和主機之間的同步性，從而實現數據的持久化和一致性。

#### **Volume 實際操作** 

今天會介紹 `Volume` 和 `Bind mount` 兩種方式來做到持久化，未來在操作 Docker Compose 會比較常用到的是 `Volume` 的做法。

1. **建立一個 Docker Volume**  

    指令建立，create 後面接 Volume 名稱，可以自訂名稱。
    ``` bash
    docker volume create mysql_data_volume
    ```  
    指令查看所有 Volume  

    ``` bash
    docker volume ls
    ```  

    ![Docker Volume command](https://i.imgur.com/D7tzEZp.png)

    查看特定 Volume 資訊  

    ``` bash
    docker inspect mysql_data_volume
    ```   

2. **更改啟動 Docker Container 指令**  

    需要加上 `-v Volume名稱:容器內部路徑`

    ``` bash
    docker run -d --name mysql-container -p 3307:3306 -v mysql_data_volume:/var/lib/mysql tom54699/mysql_practice:1.0
    ```  
    這樣就算是成功囉，Docker Volume 的默認存儲位置是在主機的 `/var/lib/docker/volumes` 目錄下。 

3. **使用 Bind mount 來指定掛載資料夾** 

    除了剛剛使用的 volume，Docker 也提供我們用 Bind mount 來掛載，指令是 `-v 本地路徑:容器內部路徑`。  

    ``` bash
   docker run -d --name mysql-container -p 3307:3306 -v /your/custom/directory:/var/lib/mysql tom54699/mysql_practice:1.0
    ```  
    確認是否掛載成功，可以查看 container 詳細資訊的 `Mounts` 部分：  

    ``` bash
    docker inspect mysql-container
    ```  

#### **額外補充：匯出 MySQL 容器內資料**  

透過指令匯出 MySQL 容器中的 SQL 檔案。  

```bash
docker exec $container_name mysqldump -u$db_user -p$db_password your_database_name > /tmp/db_backup.sql
```  

:::success
更多詳細的資訊，建議參考底下資料來源的官方文件。未來預計會在介紹 Docker Compose 時會再度提到 `Volume` 的使用，後續也會再出一篇有關 volume 更深入的討論。
:::

### **延伸閱讀**  

[使用 Rclone 將 Linux 伺服器中的 MySQL Docker 容器資料異地備份至 Google Drive]()  

### **資料來源**  

[Bind mounts | Docker Docs](https://docs.docker.com/storage/bind-mounts/)  
[Volumes | Docker Docs](https://docs.docker.com/storage/volumes/)  
[Docker volumes 教學 - 從不熟到略懂 - MyApollo](https://myapollo.com.tw/blog/docker-volumes/) 


###### tags: `server` `docker` `volume`