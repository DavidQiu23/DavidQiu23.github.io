---
title:  "EF Core Migration 多人協作：衝突成因與官方解決作法"
toc: true
toc_label: "目錄"
tags:
    - .NET
    - EF Core
---

在 Code First 開發中，Migration 不只是「一段 SQL 腳本」，它同時內含**當下整個模型的完整快照 (ModelSnapshot)**。這個設計在單人開發時毫無問題，但一旦多人同時建立 Migration，就會在快照檔上撞出合併衝突；若處理方式錯誤，還會讓後續的 Migration 悄悄損壞。這篇筆記整理衝突的成因、預防流程，以及微軟官方文件建議的正規解法。

## Migration 的組成：衝突從何而來

每次執行 `dotnet ef migrations add` 都會產生三類產物：

| 產物 | 內容 |
| --- | --- |
| `<timestamp>_<Name>.cs` | 本次 schema 變更的 `Up()` / `Down()` 方法 |
| `<timestamp>_<Name>.Designer.cs` | 這個 Migration 對應時間點的模型 metadata |
| `<Context>ModelSnapshot.cs` | **整個模型「目前狀態」的完整快照**，每次 add 都會被覆寫 |

關鍵就在第三個檔案。`ModelSnapshot` 是**單一檔案**，內容是「所有 Migration 累積後的完整模型」。因此只要兩個人各自 `add` 一個 Migration，兩邊都會改到同一個快照檔的同一區塊 —— 合併時自然衝突。

官方文件把場景講得很清楚：開發者 A 和 B 同時從 main 拉出分支，各自建立一個 Migration。若 A 先合併、B 後合併，**B 的那個 Migration 所帶的模型快照並不包含 A 的變更**。這種「快照落後」會讓之後產生的 Migration 出現各種難以察覺的損壞（context snapshot corruption）。

{: .notice--info}
> **EF Core 11（preview-3 起）的新機制**：ModelSnapshot 會額外記錄「最新一個 Migration 的 ID」。因此兩個分支各自建立 Migration 後合併時，**一定**會在快照檔產生原始碼衝突。這其實是刻意設計的訊號 —— 它在告訴你：Migration 樹已經分岔 (diverged)，其中一個必須先被丟棄並重建，而不是硬合。

## 多人協作流程（預防勝於治療）

官方對這題的第一句建議就是**事前協調**（highly recommended to coordinate in advance），盡量避免在多個分支上同時開發 Migration。實務上可以遵循幾個原則：

- **一個 PR 一個 Migration**：Migration 保持小而聚焦，命名有意義（例如 `AddOrderStatusColumn` 而非 `Update1`）。
- **各自使用本機資料庫**開發 Migration，不要多人共用同一個開發 DB 去做 add / apply，否則 `__EFMigrationsHistory` 會互相干擾。
- **頻繁 `pull` / `rebase` main**，縮短分支存活時間，從源頭降低分岔機率。
- **留意 Migration 的排序**：Migration 是依檔名 timestamp 排序套用的。若別人的 Migration 時間較晚卻先被合併進 main，你那個較早 timestamp 的 Migration 順序就會錯亂 —— 這正是後面「必須重建自己 Migration」的根本原因。

## 官方正規作法：解決分岔的 Migration 樹

當合併時偵測到 Migration 樹分岔（快照檔衝突），官方的正解**不是手動去合併快照**，而是「重建自己的 Migration」。文件標題為 *Resolving diverged migration trees*，四個步驟如下：

**步驟 1 — 中止合併**，把工作目錄還原到合併前的狀態。

```bash
git merge --abort   # 或用 git reset --hard 回到合併前
```

**步驟 2 — 移除自己的 Migration，但保留自己的模型變更**（entity 類別的 C# 改動要留著）。

```bash
dotnet ef migrations remove
```

`migrations remove` 會刪掉最後一個「尚未套用」的 Migration 檔並把 ModelSnapshot 回復到上一版；你在 entity 類別上寫的 C# 屬性 / 設定不會被動到。

**步驟 3 — 合併隊友的變更**進工作目錄。

```bash
git merge main   # 或 git pull origin main
```

**步驟 4 — 重新建立自己的 Migration。**

```bash
dotnet ef migrations add <YourMigrationName>
```

這樣一來，你的 Migration 會**乾淨地疊在隊友的 Migration 之上**，它所帶的快照包含了先前所有變更，可以安全地分享給團隊。

{: .notice--warning}
> **不要手動合併 `ModelSnapshot.cs` 的衝突。** 它是自動產生、格式冗長且彼此關聯的檔案，手動 merge 幾乎一定會留下與 Migration 不一致的狀態。正解永遠是「中止 → 移除 → 合併 → 重建」，讓工具重新產生正確的快照。

## 常見坑與範例

| 情境 | 問題 | 解法 |
| --- | --- | --- |
| 兩人同時 `add` Migration 後合併 | ModelSnapshot 衝突、後續 Migration 損壞 | 依官方四步驟重建自己的 Migration |
| 手動合併 ModelSnapshot | 快照與 Migration 不一致、apply 出錯 | 改用「移除 → 合併 → 重建」 |
| 修改已合併 / 已套用的 Migration | 別人或正式環境已套用，DB 與程式碼不同步 | 不要改舊 Migration，改新增一個「修正用」的 Migration |
| 忘記 commit ModelSnapshot | 隊友 `add` 時的基準錯誤 | Migration 的三個檔案務必一起進版控 |
| 用 `Database.Migrate()` 在正式環境自動套用 | 難以掌控、多實例部署時競態 | 產生 SQL script 或 migration bundle，於部署階段套用 |
| 刪欄位觸發資料遺失警告 | 產生的 `Up()` / `Down()` 可能遺失資料 | 檢視產生的 Migration，必要時手動補上資料搬移 |

{: .notice--danger}
> **已經被套用（別人的 DB 或正式環境）過的 Migration，絕對不要回頭去改它的內容。** 因為那些環境的 `__EFMigrationsHistory` 已記錄它套用過、不會再跑一次，你的修改永遠不會生效，反而造成程式碼與資料庫不一致。要修正就**新增一個 Migration**。

## 部署：用 Script / Bundle 取代執行期自動 Migrate

在團隊 / 正式環境，比起在服務啟動時呼叫 `context.Database.Migrate()`，官方更建議把「產生變更」與「套用變更」拆開：

- **產生冪等 SQL 腳本**，供 DBA 審核後套用：

  ```bash
  dotnet ef migrations script --idempotent -o migrate.sql
  ```

  `--idempotent` 產生的腳本會自行判斷哪些 Migration 尚未套用，可重複執行不出錯。

- **產生 Migration Bundle**（EF Core 6+），一個自足的執行檔，適合放進 CI/CD：

  ```bash
  dotnet ef migrations bundle
  ```

為什麼不建議在多實例服務啟動時直接 `Database.Migrate()`？因為多個實例同時啟動會**競態**去改 schema、應用程式帳號通常也不該擁有改 DDL 的權限，而且這種做法**無法事先審核**要跑哪些變更。把 schema 變更當成一個獨立、可審核的部署步驟，才是可控的作法。

## 參考資料

- [Microsoft Learn — Migrations in Team Environments](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/teams)
- [Microsoft Learn — Managing Migrations（add / remove / script / bundle 指令）](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/managing)
- [Microsoft Learn — Applying Migrations（script / bundle / Database.Migrate 差異）](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)
