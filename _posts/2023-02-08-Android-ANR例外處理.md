---
title:  "Android 全域例外捕獲與 ANR 問題"
tags:
    - Android
---

在使用 `Thread.setDefaultUncaughtExceptionHandler` 捕獲異常時，若未正確調用 `exitProcess()` 關閉 App，可能會引發 ANR (Application Not Responding)。在處理程序中無法直接操作 UI，但可以利用 `applicationContext` 重新啟動 App 流程。

```kotlin
val oldHandler = Thread.getDefaultUncaughtExceptionHandler()

Thread.setDefaultUncaughtExceptionHandler { paramThread, paramThrowable ->
    // 異常處理邏輯
    val intent = Intent(applicationContext, ErrorActivity::class.java)
    intent.putExtra("errorMsg", errorMsg)
    intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_NEW_TASK)
    startActivity(intent)

    if (oldHandler != null) {
        oldHandler.uncaughtException(paramThread, paramThrowable) 
    } else {
        exitProcess(1)
    }
}
```

## 參考資料
- [Android exception handling best practice?](https://stackoverflow.com/questions/16561692/android-exception-handling-best-practice)
- [Show a dialog in `Thread.setDefaultUncaughtExceptionHandler`](https://stackoverflow.com/questions/13416879/show-a-dialog-in-thread-setdefaultuncaughtexceptionhandler)
- [Android UncaughtExceptionHandler that instantiates an AlertDialog breaks](https://stackoverflow.com/questions/5519347/android-uncaughtexceptionhandler-that-instantiates-an-alertdialog-breaks)
