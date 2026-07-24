---
title:  "Kibana：建立 Data View 與常用功能"
toc: true
toc_label: "目錄"
tags:
    - Elasticsearch
    - 工具
---

Kibana：Elasticsearch 官方的視覺化與管理平台，跑在瀏覽器的網頁介面，用來探索資料、畫圖表、管理索引、打 API。功能完整但較重；輕量替代見 [Elasticvue]({% post_url 2026-07-24-Elasticvue %})。預設位置 `http://localhost:5601`。

## 建立 Data View

用 Kibana 的第一步。**Data View**（舊稱 Index Pattern）＝告訴 Kibana 要看哪些 index。沒建 → Discover 與圖表一片空白。前提是 Elasticsearch 裡已有資料（可照 [Elasticsearch 操作指令與 API 用法]({% post_url 2026-07-24-Elasticsearch操作指令與API用法 %}) 建索引）。

步驟（Kibana 8.x）：

1. 選單 (☰) → **Management** → **Stack Management**。
2. **Kibana** → **Data Views** → **Create data view**。
3. **Index pattern** 填名稱，例如 `products` 或 `products*`（`*` 萬用字元，可涵蓋多個索引）；右側即時顯示比對結果。
4. 有時間欄位 → **Timestamp field** 選它（如 `@timestamp`）；沒有就選「I don't want to use the time filter」。
5. **Save data view to Kibana**。

建好後就能到 Discover 或圖表選用這個 Data View。

## 常用功能

### Discover（探索資料）

翻 log、查原始文件。先在左上角選好 **Data View**，然後：

- 上方**時間範圍 (time range)**：例如「最近 15 分鐘」，只對有時間欄位的資料有效。
- 左側欄位清單：挑欄位加入表格。
- 搜尋列用 **KQL (Kibana Query Language)** 篩選：

```text
status: "error"                  # 等於
price > 100                      # 比較
status: "error" and tag: "db"    # 多條件
```

### Dashboard 與 Visualize（儀表板與視覺化）

先做圖、再拼儀表板：

1. **Visualize / Lens**：建立單一圖表元件（長條圖、圓餅圖、折線圖、地圖…），選資料來源與統計欄位。
2. **Dashboard**：把多個圖表拉進同一頁集中監看，共用同一組時間範圍與篩選。

### Dev Tools（開發者主控台）

直接輸入 REST 請求打 Elasticsearch API，適合手動查資料或維護索引。

### Stack Management（堆疊管理）

設定中心：管理索引、mapping、Data View、使用者與權限。建立 Data View 也在這裡。

## 參考資料

- [Kibana 官方文件](https://www.elastic.co/guide/en/kibana/current/index.html)
