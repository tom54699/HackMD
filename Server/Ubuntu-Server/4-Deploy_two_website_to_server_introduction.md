# 自建網站伺服器的簡易心得系列文(四) - 從零開始：在 Ubuntu 伺服器上部署專案 - 前導篇

**前言**
===
:::info
接下來的主題 <從零開始：在 Ubuntu 伺服器上部署專案>，會有一個很明確的目標，就是在伺服器上建立兩個有獨立網域的專案。實際專案內容不重要，主要是想分享架設過程中會用到的一些技術。
會用到 Docker-compose 建立容器來部署我們的專案，並使用 Nginx 來做反向代理和設定 SSL。如果還有餘力，後續也會分享如何使用 Github Action 來建立一套簡易的 CI/CD 流程，方便日後部署。
:::  

:::warning
Server Linux 版本: Ubuntu 22.04.3 LTS
:::  

**簡單介紹**
===

為了部署未來的專案，我們主要會用到的兩大工具就是：  

+ Docker & Docker-compose [官網連結](https://www.docker.com/) 
+ Nginx [官網連結](https://www.nginx.com/)

所以我們的主題會主要圍繞在這兩個工具的介紹和實作教學，所以這個主題預計會提到的內容如下：

1. Docker 的基礎認識和基本操作
2. Server 上 Docker 的管理 & 資安防護
3. Docker-compose 的主要部署教學
4. Nginx 的基礎認識和基本操作
5. Nginx 設定 SSL 和 反向代理到專案的相關教學
6. ... 考慮中  

###### tags: `server` `Docker` `Nginx`








