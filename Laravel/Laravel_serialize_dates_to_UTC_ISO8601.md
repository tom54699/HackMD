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

**直接去 Controller 印出從 Repository 拿到的資料**  

![Sample - Log print created_at](https://i.imgur.com/pVEVeMQ.png) 

有趣的地方來囉，猜看看兩者有何差異，我本來的理解是不該有差別，因為是從資料庫抓出來的同一包資料啊，但結果出乎我意料。 

![Sample - Log print created_at result](https://i.imgur.com/Aiz7C5A.png)  

出來的結果不一樣，代表在這個階段，Laravel 偷偷做了什麼事情，轉了我的時間格式。所以我開始瘋狂爬文，最後被我找到答案了。

**Laravel 幹了什麼？**
===  

在 Laravel 7 之後，官方更新了 `Eloquent` 模型的日期序列化格式，如果有任何 `serialize` 的行為，像是 `toArray` 或 `toJson` ，Laravel都會把資料的時間格式強制轉換成 `ISO-8601`，而 `ISO-8601` 日期始終以 ==UTC== 表示。

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

**新的設計思維**  

先說我沒有在 Laravel 官方文件明確找到為什麼要強制把格式轉換爲 `ISO-8601`，但我跟同仁討論後認為，在跨國使用上這點可能會很重要。如果是經由後端統一轉換成當地時區在前端顯示，國外的使用者顯示的時間，其實會是我本地的時間，而不是當地時間。  

但如果經由前端來轉換，就可以先偵測使用者的時區，再去做相對應的時區轉換。
所以我猜測 Laravel 可能認為後端不需要再去處理時區的問題，統一給前端 `ISO-8601` 格式即可。  

### **資料來源**  

[toArray uses incorrect timezone. · Issue #31722 · laravel/framework](https://github.com/laravel/framework/issues/31722)  
[toArray uses incorrect timezone. · Issue #31722 · laravel/framework](https://github.com/laravel/framework/issues/31722)  

###### tags: `laravel` `datetime`





