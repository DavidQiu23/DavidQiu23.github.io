# Minimal Mistakes 用法指南

本站基於 [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) 主題建置。這份文件說明在撰寫文章與客製化網站時，最常會用到的主題功能與語法，方便日後查閱。

## 撰寫新文章

所有文章都放在 `_posts/` 目錄下，檔名必須符合 `YYYY-MM-DD-title.md` 的格式，例如 `2026-07-24-my-new-post.md`。Jekyll 會依照日期與標題自動產生網址。

每篇文章的開頭都必須包含一段 **front matter**，用來設定該頁面的標題與行為：

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

除了上述基本欄位之外，Minimal Mistakes 還提供許多常用的 front matter 選項：

| 欄位 | 作用 |
| --- | --- |
| `title` | 文章標題。 |
| `toc` / `toc_label` / `toc_icon` | 是否顯示目錄、目錄標題文字、目錄圖示。 |
| `toc_sticky: true` | 讓目錄在捲動時固定於側邊。 |
| `tags` / `categories` | 標籤與分類，會自動產生對應的封存頁。 |
| `excerpt` | 在文章列表頁顯示的摘要。 |
| `classes: wide` | 隱藏側欄，讓內容佔滿整個寬度。 |
| `header` | 設定頁首橫幅（詳見下文）。 |
| `author_profile: false` | 隱藏頁面左側的作者資訊欄。 |
| `layout` | 指定頁面版型，文章預設為 `single`。 |

網站的 `_config.yml` 已在 `defaults` 區塊設定好文章的預設值，因此多數欄位不需要每篇重複填寫。

## 標註方塊 (Notices)

若要在文章中加入彩色的提示方塊，可以在段落後面加上對應的 class：

```markdown
{: .notice--info}
> 這裡放置提示內容。
```

可用的樣式包含 `.notice`（灰色）、`.notice--primary`、`.notice--info`、`.notice--success`、`.notice--warning` 與 `.notice--danger`，分別對應不同的顏色與語意。

## 按鈕 (Buttons)

在一般的 Markdown 連結後方加上 `.btn` 及樣式 class，就會渲染成按鈕：

```markdown
[下載](/download){: .btn .btn--primary}
[了解更多](/about/){: .btn .btn--info .btn--large}
```

顏色可選擇 `btn--primary`、`btn--success`、`btn--warning`、`btn--danger`、`btn--info`、`btn--inverse` 等；大小則有 `btn--small`、`btn--large`、`btn--x-large` 可搭配使用。

## 圖片

為了保持目錄整潔，請將所有圖片放入 `/assets/images/`。插入圖片時可搭配對齊 class：

```markdown
![說明文字](/assets/images/example.jpg)
{: .align-center}
```

對齊方式包含 `.align-left`、`.align-right`、`.align-center`，以及讓圖片跨出內容欄、佔滿版面的 `.full`。若需要在圖片下方加上圖說，可改用 HTML 的 `<figure>` 搭配 `<figcaption>`。

## 頁首橫幅 (Header)

在 front matter 中設定 `header`，即可為頁面加上大圖橫幅：

```yaml
header:
  overlay_image: /assets/images/banner.jpg
  overlay_filter: 0.5              # 疊上 50% 的黑色遮罩，讓標題文字更清楚
  caption: "圖片來源"
  actions:
    - label: "了解更多"
      url: "/about/"
```

若只想在文章列表顯示小縮圖，改用 `header.teaser` 即可；不需要遮罩時，可用 `header.image` 取代 `overlay_image`。

## 版型 (Layouts)

`layout` 欄位決定頁面採用哪一種版型，常見的有：

| Layout | 用途 |
| --- | --- |
| `single` | 一般文章或頁面（預設）。 |
| `home` | 首頁的文章列表。 |
| `archive` | 文章封存列表。 |
| `splash` | 全幅的著陸頁，常用於首頁的大型 hero 區塊。 |
| `categories` / `tags` | 分類與標籤的封存頁。 |

## 客製化網站

若要調整網站的整體外觀與架構，主要會修改下列檔案。

### 全域設定 (`_config.yml`)

- **更換配色**：透過 `minimal_mistakes_skin` 切換主題膚色，例如 `default`、`air`、`contrast`、`dark`、`neon`、`sunrise` 等。

  ```yaml
  minimal_mistakes_skin: "dark"
  ```

- **網站資訊**：修改 `title`、`subtitle`、`description` 等欄位。
- **作者資訊**：在 `author:` 區塊下設定 `name`、`bio` 與 `avatar`（指向 `/assets/images/` 下的圖片）。
- **社群連結**：於 `author.links:` 新增或修改 GitHub 等連結。

修改 `_config.yml` 後，必須重新啟動 Jekyll server 才會生效。

### 導覽列 (`_data/navigation.yml`)

用來設定網站上方的導覽選單：

```yaml
main:
  - title: "Home"
    url: /
  - title: "About"
    url: /about/
  # 在此新增更多選單項目…
```

### 獨立頁面

- **首頁 (`index.md` / `index.html`)**：調整首頁的排版與內容。
- **關於作者 (`about.md`)**：撰寫個人介紹頁面。

---

更完整的設定選項，請參閱 [Minimal Mistakes 官方文件](https://mmistakes.github.io/minimal-mistakes/docs/configuration/)。
