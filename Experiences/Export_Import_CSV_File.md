# CSV 亂碼問題處理的解法和注意事項 

**前言**
===
:::info
這篇文章簡單整理在處理匯出匯入 CSV 檔案遇到的一些小問題，和解決方案。
:::

**匯出後，Windows 系統打開出現亂碼**
===
因為是在 MAC 開發測試，一開始沒有意識到這個問題，直到客戶詢問才發現。

### 造成原因

1. Mac 和 Windows 系統對於 CSV 檔案的編碼默認設置不同，Mac 可能使用 UTF-8，而 Windows 上的 Excel 通常預設使用 ANSI 或其他編碼方式來讀取 CSV 檔案。

### 解決方案  

1. 要在匯出 CSV 檔案的 Header 加上 BOM（Byte Order Mark），這樣 Windows 上的 Excel 就能正確識別為 UTF-8 編碼。以下是簡單的範例：

``` php  
// output BOM
fwrite($output, "\xEF\xBB\xBF");
```  

2. 如果無法修改匯出邏輯，在 Windows 打開記事本 另存檔案時選擇 `UTF-8 BOM`。

### 注意事項

1. 前後端一致性：

    - 問題：如果前端API輸出了正確的具有BOM的CSV，但前端在處理時未保留BOM，最終用戶下載的檔案仍然可能出現亂碼。  

    - 原因：有些前置庫在處理CSV資料時可能會自動刪除BOM。  

    - 解決方案：確保前端程式碼在處理CSV資料時保留BOM。

2. 匯入時要注意 `BOM` 的部分:  

    若是匯入後有要開頭就要讀取 CSV，要注意如果是 BOM 開頭要從文件開頭讀取 3 個字節，這是因為 UTF-8 的 BOM 是由三個字節 `\xEF\xBB\xBF` 組成的，簡單範例如下: 
    ``` php
    $handle = fopen($path, 'r');

    $bom = fread($handle, 3);
    if ($bom != "\xEF\xBB\xBF") {
        rewind($handle);
    }
    ```