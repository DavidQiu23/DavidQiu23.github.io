---
title:  "Elasticsearch 操作指令與 API 用法"
toc: true
toc_label: "目錄"
tags:
    - Elasticsearch
---

Elasticsearch：分散式搜尋與分析引擎，資料以 JSON 文件存放，操作幾乎都透過 REST API。這篇整理常用的操作：狀態查詢、Index 與文件 CRUD、批次寫入、Query DSL。對應 7.x / 8.x。

範例統一用 `METHOD /path` + JSON body 的簡寫，等同對 Elasticsearch 發 HTTP 請求，可用 curl 或任何 HTTP 客戶端送出。例如 `GET /products/_search { ... }` 等於：

```bash
curl -X GET "localhost:9200/products/_search" -H "Content-Type: application/json" -d '
{
  "query": { "match_all": {} }
}'
```

## 快速檢視叢集與 index 狀態

`_cat` 系列 API 會回傳精簡的表格輸出，加上 `?v`（verbose）會顯示欄位標題：

```json
GET /_cat/health?v
GET /_cat/indices?v
GET /_cat/nodes?v
```

## Index 與 mapping

### 建立 index 與定義 mapping

```json
PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name":  { "type": "text" },
      "brand": { "type": "keyword" },
      "price": { "type": "integer" },
      "created_at": { "type": "date" }
    }
  }
}
```

- `text`：會被分詞，用於全文搜尋。
- `keyword`：不分詞，用於精確比對、排序與聚合。

### 常用資料型態

| 型態 | 說明 |
| --- | --- |
| `text` | 全文搜尋，寫入時經分析器分詞；不適合排序與聚合 |
| `keyword` | 精確值（標籤、狀態、Email 等），用於過濾、排序、聚合 |
| `long` / `integer` / `short` / `byte` | 整數，儲存範圍由大到小 |
| `double` / `float` / `half_float` | 浮點數 |
| `scaled_float` | 以固定倍數存成整數，適合金額（需搭配 `scaling_factor`）|
| `boolean` | 布林值 `true` / `false` |
| `date` | 日期時間，可用 `format` 指定格式 |
| `object` | JSON 物件（預設，會被展平成 `a.b` 形式）|
| `nested` | 物件陣列，每個物件獨立索引，可正確查詢陣列內欄位的關聯 |
| `ip` | IPv4 / IPv6 位址 |
| `geo_point` | 經緯度座標 |

字串沒有名為 `string` 的型態，`text` 與 `keyword` 是兩種不同用途。實務上常把同一欄位同時建成兩者——`text` 供全文搜尋、底下再放一個 `keyword` 子欄位供聚合/排序，這叫 **multi-field**：

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      }
    }
  }
}
```

之後 `name` 走全文搜尋、`name.keyword` 用於精確比對、排序與聚合（這也是前面 `term` 範例會用 `.keyword` 的原因）。

### 檢視與刪除 index

```json
# 檢視 mapping / settings
GET /products/_mapping
GET /products/_settings

# 刪除整個 index
DELETE /products
```

## 文件 CRUD

```json
# 新增文件（自動產生 _id）
POST /products/_doc
{ "name": "機械鍵盤", "brand": "Ducky", "price": 3200, "created_at": "2026-07-24" }

# 指定 _id 新增 / 覆蓋
PUT /products/_doc/1
{ "name": "無線滑鼠", "brand": "Logitech", "price": 1500, "created_at": "2026-07-24" }

# 取得單筆
GET /products/_doc/1

# 部分更新
POST /products/_update/1
{ "doc": { "price": 1350 } }

# 刪除
DELETE /products/_doc/1
```

## 批次寫入 `_bulk`

`_bulk` 用「動作行 + 資料行」成對出現，每行都是獨立 JSON、**結尾要有換行**：

```json
POST /_bulk
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "螢幕", "brand": "Dell", "price": 6000 }
{ "index": { "_index": "products", "_id": "3" } }
{ "name": "耳機", "brand": "Sony", "price": 4800 }
```

## 搜尋與 Query DSL

搜尋端點是 `GET /index/_search`，查詢條件放在 `query` 裡。

### match（全文）vs term（精確）

```json
# match：對 text 欄位做全文搜尋（會分詞）
GET /products/_search
{
  "query": { "match": { "name": "機械鍵盤" } }
}

# term：精確比對，用在 keyword 欄位；
# 若要對 text 欄位精確比對，需用其 .keyword 子欄位
GET /products/_search
{
  "query": { "term": { "brand": "Ducky" } }
}
```

### bool 組合查詢

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "name": "滑鼠" } } ],
      "filter":   [ { "range": { "price": { "lte": 2000 } } } ],
      "should":   [ { "term":  { "brand": "Logitech" } } ],
      "must_not": [ { "term":  { "brand": "NoName" } } ]
    }
  }
}
```

- `must`：必須符合，會算分數。
- `filter`：必須符合，但不算分數（可被快取，效能較好）。
- `should`：加分項；沒有 `must` 時至少要符合一個。
- `must_not`：必須不符合。

### 分頁與排序

```json
GET /products/_search
{
  "from": 0,
  "size": 10,
  "sort": [ { "price": "desc" } ],
  "query": { "match_all": {} }
}
```

### 聚合 aggregations

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand" }
    },
    "avg_price": {
      "avg": { "field": "price" }
    }
  }
}
```

`"size": 0` 表示只要聚合結果、不回傳文件；`terms` 聚合的欄位要用 `keyword` 型別。

## 叢集健康檢查

```json
GET /_cluster/health
```

回傳的 `status` 有三種：

- **green**：所有主分片與副本分片都正常。
- **yellow**：主分片正常，但有副本未分配（單節點很常見）。
- **red**：有主分片未分配，部分資料無法存取，需要處理。

## 參考資料

- [Elasticsearch REST APIs 官方文件](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)
- [Query DSL 官方文件](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
