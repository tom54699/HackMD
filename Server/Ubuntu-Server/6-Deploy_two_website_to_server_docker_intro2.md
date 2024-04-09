# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker 介紹篇（二）

**前言**
===
:::info
再上一篇文章中，我們大致了解了 Docker，也成功建立了一個 MySQL 容器，大家應該對 `Dockerfile`、`container`、`image` 有一點概念了。如果還是不太懂，可以多多爬文，一開始接觸有個撞牆期很正常，建議實作可以比較快熟悉上手。 今天主要會簡單介紹 `Volume`，和一些平常基本會用到的 Docker 指令。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
::: 

**Docker Volume 是什麼?**  
===  

Docker Volume 是 Docker 中用於持久性數據存儲的機制之一。在 Docker 容器中，文件系統是暫時性的，這意味著當容器終止時，任何在容器內創建的文件或數據都會丟失。為了解決這個問題，Docker 提供了 Volume，它允許將數據持久化存儲在主機上，即使容器終止也能保留。  

簡單的以上次建立的 MySQL 容器來舉例，當我們成功建立了 MySQL 容器但未建立相對應的 Volume 時，這將導致在容器終止或重新啟動時，所有在容器內部存儲的資料都會丟失。  

這就是為什麼需要使用 Volume 的原因。通過建立 Volume，我們可以將容器內的數據持久化存儲在主機上，從而確保即使容器終止，數據仍然得以保留。此外，當容器內的數據發生更改時，Volume 會自動將這些變化同步到主機上。這意味著你不需要手動同步數據，Volume 會負責確保數據在容器和主機之間的同步性，從而實現數據的持久化和一致性。

接下來，讓我們來實際進行操作，以確保持久化數據的安全性。  

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

2. **更改啟動 Docker Container 指令**  

    需要加上 `-v Volume名稱:容器內部路徑`

    ``` bash
    docker run -d --name mysql-container -p 3307:3306 -v mysql_data_volume:/var/lib/mysql tom54699/mysql_practice:1.0
    ```  
    這樣就算是成功囉，Docker Volume 的默認存儲位置是在主機的 `/var/lib/docker/volumes` 目錄下。 

2. **額外補充: 使用 Bind mount 來指定掛載資料夾** 

    剛剛我們使用的是 volume，Docker 也提供我們用 Bind mount 來掛載，加上 `-v 本地路徑:容器內部路徑`。  

    ``` bash
   docker run -d --name mysql-container -p 3307:3306 -v /your/custom/directory:/var/lib/mysql tom54699/mysql_practice:1.0
    ``` 



<!-- :::success
額外補充： 可以根據 `IMAGE ID` 刪掉不需要的映像檔，尤其是如果當改了 Dockerfile 但 build 的時候，沒有更換映像檔案的 tag 話，就會發生 image 沒有更新的狀況。  
:::  

``` bash
docker image rmi <IMAGE ID>
``` 
IMAGE ID 可以不用全部打出來，Docker 會自動識別並刪除與所提供的部分 IMAGE ID 相符的映像。
    -->