# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker 介紹篇（三）

**前言**
===
:::info
再上一篇文章中，我們大致了解了 Docker，也成功建立了一個 MySQL 容器，大家應該對 `Dockerfile`、`container`、`image` 有一點概念了。如果還是不太懂，可以多多爬文，一開始接觸有個撞牆期很正常，建議實作可以比較快熟悉上手。 今天主要會簡單介紹 `Volume` 和簡單的實作。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
::: 

<!-- :::success
額外補充： 可以根據 `IMAGE ID` 刪掉不需要的映像檔，尤其是如果當改了 Dockerfile 但 build 的時候，沒有更換映像檔案的 tag 話，就會發生 image 沒有更新的狀況。  
:::  

``` bash
docker image rmi <IMAGE ID>
``` 
IMAGE ID 可以不用全部打出來，Docker 會自動識別並刪除與所提供的部分 IMAGE ID 相符的映像。
    -->

### **資料來源**  

[Bind mounts | Docker Docs](https://docs.docker.com/storage/bind-mounts/)  
[Volumes | Docker Docs](https://docs.docker.com/storage/volumes/)  
[Docker volumes 教學 - 從不熟到略懂 - MyApollo](https://myapollo.com.tw/blog/docker-volumes/) 


###### tags: `server` `docker`