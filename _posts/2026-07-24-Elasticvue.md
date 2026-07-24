---
title:  "Elasticvue：Elasticsearch 的圖形化介面 (GUI)"
toc: true
toc_label: "目錄"
tags:
    - Elasticsearch
    - 工具
---

[Elasticvue](https://elasticvue.com/)：免費開源的 Elasticsearch 圖形化介面 (GUI)，不用寫 JSON 就能瀏覽、搜尋、管理叢集資料，elasticsearch-head / cerebro 的現代替代品。想輕量看資料、管索引又不想架 [Kibana]({% post_url 2026-07-24-Kibana常用功能與DataView %}) 時很合適。用法有瀏覽器擴充、線上版、Docker、桌面 App，本篇用官方推薦的**桌面 App**。

## 安裝桌面 App

到 [官網下載頁](https://elasticvue.com/) 依系統安裝：

- **Windows**：MSI 安裝檔。
- **macOS**：Homebrew，或下載 Intel / Apple Silicon 版。
- **Linux**：AUR，或直接跑 AppImage。

## 連線到叢集

第一次開啟填入 endpoint（例如 `http://localhost:9200`）與帳密即可。

桌面 App 直接對叢集發請求，**不受瀏覽器跨來源限制 (CORS) 影響**，不必改 `elasticsearch.yml` 的 CORS 設定——這是它比線上版 / Docker 省事的地方。

## 主要功能

- **叢集概覽 (Cluster overview)**：叢集健康 (green / yellow / red)、節點與分片狀態。
- **索引與別名管理 (Index & alias management)**：建立 / 刪除 / 檢視索引與 mapping、管理 alias。
- **分片管理 (Shard management)**：檢視分片分佈與狀態。
- **搜尋與編輯文件**：搜尋、篩選、直接編輯單筆文件。
- **REST 查詢 (Rest queries)**：輸入 REST 請求打 API，類似 Kibana Dev Tools。
- **快照與儲存庫管理 (Snapshot & repository management)**：管理備份快照與 repository。

## 參考資料

- [Elasticvue 官網](https://elasticvue.com/)
- [Elasticvue GitHub (cars10/elasticvue)](https://github.com/cars10/elasticvue)
