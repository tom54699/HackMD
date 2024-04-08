# Laravel 從資料庫拿 datetime 格式資料錯誤 - 發生原因＆解決方案

**前言**
===
:::info
前幾天，前端同仁告訴我 API 中的 created_at 時間錯誤，時間都提早了八個小時。本來以為是小問題，但後來發現事情不簡單。所以今天來分享一下，我這次遇到的 Laravel 資料時區問題。
:::  

**事情經過**
===  
如果想跳過直接看解法可以往下滑到解決方法。  

**檢查資料庫**  

我一開始想說先看看資料庫有沒有錯誤，但發現 created_at 時間很正常，`2024-04-02 15:22:15`。  

![Sample - created_at](https://i.imgur.com/NKjwd5I.png)  

**API拿資料**  

但是從 API 拿到的時間則是 `2024-04-02 07:22:15`，差了八小時，八九不離十跟時區應該有關係。  

![Sample - API created_at](https://i.imgur.com/8g3fni0.png)  

**檢查 /config/app.php**  

``` bash
'timezone' => 'Asia/Taipei'
``` 

雖然覺得跟這個無關，因為這邊 `timezone` 主要應該是日期時間資料寫入資料庫時，Laravel 會自動將日期時間轉換為資料庫的時區，應該跟拿這次問題無關。

**檢查 model**  

``` php
protected $casts = [
    'created_at' => 'datetime:Y-m-d H:i:s',
    'updated_at' => 'datetime:Y-m-d H:i:s',
    'deleted_at' => 'datetime:Y-m-d'
];
``` 

也沒有轉換時區的相關程式碼。只有設定 `casts`，定義要拿的日期時間格式，也和時區無關。

但我也先註解掉了 casts 設定，來看看新的結果。  

![Sample - API created_at](https://i.imgur.com/CJ3ZR50.png)  

Z是相對協調世界時(UTC)時間0偏移的代號，所以代表 Laravel 都統一給我零時區的時間。

**直接去 Controller 印出從 Repository 拿到的資料**  

![Sample - Log print created_at](https://i.imgur.com/pVEVeMQ.png) 

有趣的地方來囉，猜看看兩者有何差異，我本來的理解是不該有差別，因為是從資料庫抓出來的同一包資料啊，但結果出乎我意料。 

![Sample - Log print created_at result](https://i.imgur.com/Aiz7C5A.png)  

出來的結果不一樣，代表在這個階段，Laravel 偷偷做了什麼事情，轉了我的時間格式。所以我開始瘋狂爬文，最後被我找到答案了。

**Laravel 幹了什麼？**
===  

在 Laravel 7 之後，官方更新了 `Eloquent` 模型的日期序列化格式，如果有任何 `serialize` 的行為，像是 `toArray` 或 `toJson` ，Laravel都會把資料的時間格式強制轉換成 `ISO-8601`，而 `ISO-8601` 日期始終以 `UTC` 表示。  

像前面的測試可以得知，我們都是拿到 UTC 零時區的時間。

可以參考這兩篇官方文件：  

[Date Serialization](https://laravel.com/docs/7.x/upgrade#date-serialization)  
[Date Casting, Serialization, & Timezones](https://laravel.com/docs/9.x/eloquent-mutators#date-casting-and-timezones)  

**解決方案**  

1. 在 model 加入這一段：

    加入這段主要是使用更新前的設計，不想要強制被轉換時間格式，維持舊的方案。

    ``` php
    use DateTimeInterface;

    public function serializeDate(DateTimeInterface $date){
        {
            return $date->format('Y-m-d H:i:s');
        }
    }
    ```  

2. 讓前端去處理時間格式的轉換  

    這乍聽之下，好像顯得是後端不想處理想丟給前端。但在了解過後，可以明白為什麼 Laravel 想強制轉換成 `ISO-8601` ，也建議轉換時間格式的工作要留給前端處理。

**設計思維**  

在 Laravel 官方文件中可能沒有明確解釋為什麼要強制將日期時間格式轉換為 `ISO-8601`，但通過與同事的討論，我們認為這對於跨越多個時區的應用程序來說可能非常重要。如果後端統一轉換日期時間格式為當地時區再傳送至前端，在國外的使用者查看時，實際上會看到的是服務器所在時區的時間，而不是該地區的本地時間。這可能導致混淆，尤其是在那些遵循夏令時間或時區與地理位置不對應的地區。  

但如果由前端負責轉換，可以先檢測使用者的時區，然後進行相應的轉換，這樣就能確保用戶看到的是正確的時間。因此，我猜想 Laravel 可能認為後端不需要再處理時區的問題，僅需統一將日期時間格式轉換為 `ISO-8601` 提供給前端即可。 

### **資料來源**  

[toArray uses incorrect timezone. · Issue #31722 · laravel/framework](https://github.com/laravel/framework/issues/31722)

[ISO 8601 - 維基百科，自由的百科全書](https://zh.wikipedia.org/zh-tw/ISO_8601)  

[Based on thousands of APIs, what is the best approaches and format for handling timezone, timestamps, and datetime in APIs and Apps | Moesif Blog](https://www.moesif.com/blog/technical/timestamp/manage-datetime-timestamp-timezones-in-api/)  

[Always Use UTC Dates And Times. As soon as you handle dates or times… | by Kyle K | Medium](https://kylekatarnls.medium.com/always-use-utc-dates-and-times-8a8200ca3164)  

[Laravel: Timezones in the database? | by Italo Baeza Cabrera | Medium](https://darkghosthunter.medium.com/laravel-timezones-in-the-database-1905020cc699)


###### tags: `laravel` `datetime`





