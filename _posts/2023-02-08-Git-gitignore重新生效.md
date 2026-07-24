---
title:  "讓 .gitignore 重新生效"
tags:
    - Git
---

當檔案已被追蹤（Tracked）後才加入 `.gitignore` 時，需清除快取以使設定生效：

```bash
git rm -r --cached . # 清除快取
git add .            # 重新追蹤檔案
```
