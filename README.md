# DavidX 撰文速查手冊 🚀

## 1. 建立新文章 (Naming & Path)
所有文章必須放在 `_posts/` 目錄下，且檔名必須符合以下格式：
`YYYY-MM-DD-title.md` (例如：`2023-10-27-my-new-post.md`)

## 2. 標準 Front Matter 模板
每篇文章開頭必須包含以下資訊：
```markdown
---
title:  "文章標題"
toc: true
toc_label: "目錄"
tags: 
    - 標籤1
    - 標籤2
---
```

## 3. 本地預覽 (Local Server)
在終端機輸入以下指令即可啟動預覽：
```bash
bundle exec jekyll serve
```
*   預覽網址：`http://127.0.0.1:4000`
*   修改儲存後會自動重新編譯 (免重啟)
*   **注意**：修改 `_config.yml` 必須重新啟動 server 才會生效。

## 4. 常用語法與組件 (Minimal Mistakes)
*   **圖片路徑：** `/assets/images/filename.jpg`
*   **標註 (Notices)：**
    ```markdown
    {: .notice--info}
    > 這裡放置資訊標註內容。
    ```
    (可選：`notice--info`, `notice--warning`, `notice--danger`, `notice--success`)

## 5. 圖片存放位置
將所有圖片放入 `/assets/images/` 以保持目錄整潔。

---

## 🛠️ 如何自定義網站 (Customization)

若要修改網站的整體外觀或架構，請調整以下檔案：

### 全域設定 (`_config.yml`)
*   **修改網站標題與簡介**：修改 `title`、`subtitle`、`description` 等欄位。
*   **更換主題配色 (Skin)**：
    ```yaml
    minimal_mistakes_skin: "dark" # 可改成 "default", "air", "aqua", "contrast", "dirt", "neon", "plum", "sunrise", "mint" 等
    ```
*   **修改作者資訊與頭像**：在 `author:` 區塊下修改 `name`, `bio`, `avatar` (指向 `/assets/images/` 下的圖片)。
*   **修改社群連結**：在 `author.links:` 中新增或修改 GitHub 等連結。

### 導覽列與選單 (`_data/navigation.yml`)
用來設定網站上方的導覽列 (Header)。
```yaml
main:
  - title: "Home"
    url: /
  - title: "About"
    url: /about/
  # 在此處新增更多選單項目...
```

### 首頁與其他獨立頁面
*   **首頁 (`index.md` 或 `index.html`)**：可修改網站的首頁排版與內容。
*   **關於作者 (`about.md`)**：修改關於您的介紹頁面。

> 💡 **Tip:** 詳細的主題自定義選項，可隨時查閱 [Minimal Mistakes 官方文件](https://mmistakes.github.io/minimal-mistakes/docs/configuration/)。
