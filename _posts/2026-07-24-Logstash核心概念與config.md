---
title:  "Logstash：Pipeline 概念與最小 config"
toc: true
toc_label: "目錄"
tags:
    - Elasticsearch
    - 工具
---

Logstash：ELK 的「L」，資料收集與處理**管線 (pipeline)**——把資料收進來、轉換、再送出去，最常見用途是把 log 清洗、結構化後送進 Elasticsearch。

## Pipeline 與 config

Logstash 一次處理一筆**事件 (event)**，每筆事件依序流過三段，每段由**外掛 (plugin)** 組成——這三段正好對應 config 的三個最外層區塊：

```
input → filter → output
```

每筆事件是一組**欄位 (field)** 的記錄（類似一個 JSON 物件）。原始輸入預設整行放在 `message` 欄位，另有內建欄位如 `@timestamp`（事件時間）、`tags`（標籤陣列）。filter 的工作就是從 `message` 拆出更多欄位、或修改欄位——這也是為什麼 grok 都寫 `match => { "message" => ... }`（從 `message` 拆）。

### 整體結構

一份 pipeline config（例如 `logstash.conf`）由三個**最外層區塊**組成，順序固定、缺一不可（filter 內容可空）：

```
input  { ... }   # 資料從哪來
filter { ... }   # 怎麼處理
output { ... }   # 送到哪去
```

每個區塊裡放一到多個**外掛區塊**，外掛區塊裡再放該外掛的**設定項**，三層巢狀：

```ruby
區塊 {                    # input / filter / output
  外掛名 {                # file / grok / elasticsearch …
    設定項 => 值           # 這個外掛的參數
    設定項 => 值
  }
}
```

- **外掛名**：決定這一段做什麼；同一區塊可放多個外掛，由上到下依序執行。
- **`設定項 => 值`**：`=>`（hash rocket）是設定的賦值語法（不是 `=`），設定項名稱由外掛定義。
- **值的型別**：字串 `"beginning"`、數字/布林 `5` / `true`、陣列 `[ "a", "b" ]`、雜湊 `{ "k" => "v" }`。
- 註解用 `#`。

### input（輸入）

資料來源，一條 pipeline 可有多個。常用外掛：

- `file`：追蹤 log 檔（類似 `tail -f`），記錄讀取位置，重啟不重讀。
- `beats`：接收 Filebeat / Metricbeat 送來的資料，正式環境最常見。
- `stdin`：從終端機輸入，測試用。
- `kafka` / `redis`：從訊息佇列拉資料，做緩衝與削峰。

可搭配 **codec（編解碼器）** 決定如何解析原始資料，例如 `json`、`multiline`（把多行堆疊成一筆事件，如 Java stack trace）。

### filter（過濾/處理）

轉換與清洗，是 Logstash 的核心，可串接多個外掛，由上到下依序處理同一筆事件。常用外掛與用法：

**`grok`** — 把一行非結構化 log 用樣式拆成欄位，最重要的 filter。語法 `%{樣式:欄位名}`，加 `:int` / `:float` 可順便轉型：

```ruby
grok {
  match => { "message" => "%{IP:client} %{WORD:method} %{NUMBER:status:int}" }
}
# "1.2.3.4 GET 200" → client="1.2.3.4", method="GET", status=200（整數）
```

常用樣式：`IP`、`WORD`、`NUMBER`、`URIPATHPARAM`、`TIMESTAMP_ISO8601`、`GREEDYDATA`（貪婪比對剩餘全部），以及組合樣式 `COMBINEDAPACHELOG`（一次拆好 Nginx/Apache 標準格式）。沒比對到會標上 `_grokparsefailure` tag。

**`date`** — 把 log 內的時間字串設為事件時間 `@timestamp`（Kibana 依它排序與篩選）：

```ruby
date {
  match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]   # [ 來源欄位, 時間格式 ]
}
```

**`mutate`** — 對欄位做改名、轉型、刪除、去空白等：

```ruby
mutate {
  rename       => { "host" => "server" }     # 改名
  convert      => { "bytes" => "integer" }   # 轉型
  remove_field => [ "message" ]              # 刪除
}
```

**`json`** — 把某欄位的 JSON 字串解析成結構化子欄位：

```ruby
json {
  source => "payload"   # payload 是一段 JSON 字串 → 展開成欄位
}
```

**`geoip`** — 由 IP 推算地理位置（國家、經緯度），供地圖視覺化：

```ruby
geoip {
  source => "client"    # 產生 geoip.country_name、geoip.location 等欄位
}
```

**條件式** — 用 `if / else if / else` 只對部分事件套用；判斷可用 `==`、`>=`、`=~`（正則比對）、`in` 等。條件裡的 `[status]`、`[request]` 是**事件欄位**，必須是同一個 filter 區塊中、前面的 filter 先產生的（下例由 grok 拆出）：

```ruby
filter {
  grok {
    match => { "message" => "%{WORD:method} %{URIPATHPARAM:request} %{NUMBER:status:int}" }
  }
  # 上面 grok 拆出 status、request 後，才能在條件式引用
  if [status] >= 500 {
    mutate { add_tag => ["server_error"] }
  } else if [request] =~ /^\/api\// {
    mutate { add_tag => ["api"] }
  }
}
```

filter 可省略（純轉送），但 input 與 output 一定要有。

### output（輸出）

送到哪裡，可同時送多個目的地，也能用條件式分流。常用外掛：

- `elasticsearch`：送進 ES，最常見。
- `stdout { codec => rubydebug }`：印到終端機，除錯必備。
- `file`：寫入檔案。

```ruby
output {
  if "server_error" in [tags] {
    elasticsearch { hosts => ["http://localhost:9200"]; index => "errors-%{+YYYY.MM.dd}" }
  } else {
    elasticsearch { hosts => ["http://localhost:9200"]; index => "logs-%{+YYYY.MM.dd}" }
  }
}
```

### 範例：處理 Nginx access log

```ruby
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"   # 首次讀取從頭開始（預設 end，只讀新增行）
  }
}

filter {
  grok {
    # 內建樣式 COMBINEDAPACHELOG 直接對應 Nginx/Apache 標準格式
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]   # 設為 @timestamp
  }
  geoip {
    source => "clientip"            # 由來源 IP 補上地理欄位
  }
  mutate {
    remove_field => [ "message", "timestamp" ]   # 拆完就移除原始欄位
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

### 其他重點

- **自訂 grok 樣式**：內建樣式不夠用時，可用 `(?<欄位名>正則)` 直接寫正則，或用 Kibana Dev Tools 內建的 **Grok Debugger** 貼樣本即時測試。

- **欄位參照**：config 裡用 `[欄位名]` 取值，巢狀用 `[a][b]`（如 `[geoip][country_name]`）；字串內插值用 `%{欄位名}`。

- **三種長得像的 `%{}`**：grok 樣式 `%{IP:client}`（比對並存成欄位）、欄位內插 `%{欄位名}`（取欄位值組成字串，如 index 名稱）、時間格式 `%{+YYYY.MM.dd}`（用事件時間格式化）——用途完全不同。

- **`@timestamp` vs 事件時間**：沒有 `date` filter 時，`@timestamp` 是「Logstash 收到的時間」而非「log 發生的時間」，通常要用 `date` 校正。

- **驗證與除錯**：`bin/logstash -f logstash.conf --config.test_and_exit` 只檢查語法不啟動；跑起來時保留 `stdout { codec => rubydebug }` 確認欄位是否正確拆解。送進 ES 後即可照 [Elasticsearch 操作指令與 API 用法]({% post_url 2026-07-24-Elasticsearch操作指令與API用法 %}) 查詢、用 [Kibana]({% post_url 2026-07-24-Kibana常用功能與DataView %}) 檢視。

## 參考資料

- [Logstash 官方文件](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Grok filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
- [Logstash 內建 grok patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/main/patterns)
