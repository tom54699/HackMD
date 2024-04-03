# 自建網站伺服器的簡易心得系列文(五) - 從零開始：在 Ubuntu 伺服器上部署專案 - Docker 介紹篇（二）

**前言**
===
:::info

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