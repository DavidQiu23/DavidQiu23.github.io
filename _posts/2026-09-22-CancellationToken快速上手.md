---
title:  "CancellationToken 快速上手"
toc: true
toc_label: "目錄"
tags:
    - .NET
    - 並行處理
---

一支批次程式正在處理一千個檔案，跑到第三百個的時候，你想叫它停下來。

```csharp
foreach (var file in files)
{
    Process(file);   // 一個檔一兩秒，一千個檔就是半小時
}
```

直覺的想法是把執行它的那個執行緒砍掉。`Thread.Abort` 就是幹這件事的，而它在 .NET Core 之後已經不能用，呼叫它會丟 `PlatformNotSupportedException`。理由很實際：從外面砍執行緒，是在它執行到的任意一行把它打斷，拿著的鎖不會釋放，寫到一半的檔案就停在一半，交易不會 rollback。程式沒有機會收尾。

所以 .NET 的取消是另一套做法：**取消是合作制 (cooperative cancellation)**。外面只負責「發出取消的請求」，真正停下來的動作，由工作本身在它認為安全的時間點自己做。

## 取消是合作制

這套機制有兩個型別，像遙控器跟接收器。`CancellationTokenSource` 是遙控器，握在發起取消的那一方手上，按下去就是 `Cancel()`；`CancellationToken` 是接收器，交給要被停下來的那段程式，它只能讀、不能按。

```csharp
using var cts = new CancellationTokenSource();   // 遙控器，建的人負責 Dispose
CancellationToken token = cts.Token;             // 接收器，從遙控器上拿

Task job = Task.Run(() => Run(files, token), token);   // token 也給 Task.Run

Console.ReadLine();   // 使用者按 Enter
cts.Cancel();         // 按下遙控器

try
{
    await job;
}
catch (OperationCanceledException)
{
    Console.WriteLine("已取消");   // 取消是正常的結束，要接住
}
```

重點是**按的人跟停的人是兩段不同的程式碼**。`cts.Cancel()` 不會讓 `Run` 立刻停止，它只是把 token 上的旗標翻成「已請求取消」，然後就回來了。`Run` 什麼時候真的停，完全看 `Run` 自己寫得怎麼樣。

| | `CancellationTokenSource` | `CancellationToken` |
| --- | --- | --- |
| 角色 | 遙控器 | 接收器 |
| 誰拿著 | 發起取消的一方：呼叫端、host、UI 事件 | 要被停下來的那段工作 |
| 按 `Cancel()` | 有這個方法 | 沒有，讀不到也按不到 |
| 讀 `IsCancellationRequested` | 有，但通常用不到 | 有，工作方靠它判斷 |
| 型別 | `class`，持有計時器與註冊清單 | `struct`，複製成本低，往下傳不心疼 |
| 誰負責 `Dispose` | 建立它的人 | 不需要 |

分工照這條線走：**建立 `CancellationTokenSource` 的人負責 `Dispose` 它**，其他人只拿到 token。token 是 struct，當參數一路往下傳、存進 lambda、複製幾十份都不心疼。

## 怎麼停手

工作方要停，有兩種寫法。

**第一種，`ThrowIfCancellationRequested()`。** 已經請求取消就當場丟 `OperationCanceledException`，整個呼叫堆疊一路往上炸出去：

```csharp
public void Run(IEnumerable<string> files, CancellationToken token)
{
    foreach (var file in files)
    {
        token.ThrowIfCancellationRequested();   // 已取消就在這裡拋例外
        Process(file);
    }
}
```

**第二種，自己看 `IsCancellationRequested` 然後回傳。** 同一個迴圈，沒有例外，方法正常結束：

```csharp
int processed = 0;

foreach (var file in files)
{
    if (token.IsCancellationRequested)
    {
        SaveProgress(processed);   // 先把進度寫回去，下次從這裡接
        return;
    }

    Process(file);
    processed++;
}
```

差別有兩個。一是**呼叫端要不要 catch**：丟例外的版本，上層得接 `OperationCanceledException`；回傳的版本，上層看起來就是跑完了。二是**有沒有收尾要做**：要寫進度、要 flush 的用第二種比較好寫，`return` 之前那幾行就是收尾的位置。沒有收尾需求就用第一種，它短，而且例外的型別本身就把「這是被取消的，不是跑完的」帶給了上層。

坑在**檢查的位置**：

```csharp
token.ThrowIfCancellationRequested();   // 錯：只檢查這一次，進迴圈之後按幾次取消都沒用

foreach (var file in files)
{
    Process(file);
}
```

檢查要放在迴圈裡面，每一圈都做。而且「每一圈檢查一次」的反應速度上限，就是跑一圈要多久——`Process` 一個檔要五分鐘的話，按下取消之後最慢也要五分鐘才停得下來。要更快，token 得再往 `Process` 裡面傳。

## token 要往下傳

.NET 的慣例是 `CancellationToken` 當**最後一個參數**，公開 API 給它預設值 `default`，讓不在意取消的呼叫端可以不傳：

```csharp
public async Task RunAsync(IEnumerable<string> files, CancellationToken token = default)
{
    foreach (var file in files)
    {
        token.ThrowIfCancellationRequested();

        string content = await File.ReadAllTextAsync(file, token);   // 讀檔中途也能停
        await UploadAsync(content, token);                           // 一路往下傳
        await Task.Delay(TimeSpan.FromSeconds(1), token);            // 節流，等待中途也能停
    }
}
```

`Task.Delay`、`HttpClient` 的各個方法、EF Core 的 `SaveChangesAsync` 與 `ToListAsync`、`Stream` 的 async 方法，全都收 token。這不是多禮，是必要的：**沒傳 token 的那個 `await`，就是一段不能取消的等待**。迴圈檢查得再勤，卡在一個沒傳 token、要等三十秒的 HTTP 請求上，就是得等它三十秒。

最常見的坑是**在某一層把 token 吞掉**。這一層的簽章收了 token，卻沒往下傳：

```csharp
// 錯：這一層收了 token，卻沒往下傳，以下整段都不能取消
await httpClient.PostAsync(url, new StringContent(content));

// 正確
await httpClient.PostAsync(url, new StringContent(content), token);
```

這種錯最難查，因為上下游看起來都對——簽章有 token、迴圈有檢查，就是取消之後停不下來。從最外層一路往下看每一個 `await` 有沒有帶 token，比較快。

## 逾時

逾時就是「時間到了自動按遙控器」，同一套機制，不必另外寫計時器。

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(10));   // 建的時候就設
await RunAsync(files, cts.Token);

using var cts2 = new CancellationTokenSource();
cts2.CancelAfter(TimeSpan.FromMinutes(10));   // 或是之後才決定
```

常見的需求是兩個條件都要：使用者按了取消要停，跑太久也要停。`CancellationTokenSource.CreateLinkedTokenSource` 把數個 token 併成一個，**任何一邊取消，合出來的那個就取消**：

```csharp
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromMinutes(10));
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(userToken, timeoutCts.Token);

try
{
    await RunAsync(files, linkedCts.Token);   // 往下傳的是合出來的那個
}
catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested)
{
    logger.LogWarning("批次逾時");   // 只有計時器觸發時 timeoutCts 才會是已取消
}
```

要分辨是哪一邊取消的，就看原本那顆 source：使用者按取消時 `timeoutCts` 不會被連動，連動的是 `linkedCts`，來源 token 本身不受影響。真的沒有東西可傳的時候用 `CancellationToken.None`，不要為此多建一顆永遠不取消的 source。

坑在 `Dispose`。`CancellationTokenSource` 帶著計時器與註冊清單，**linked source 尤其一定要 `using`**：它會在來源 token 上註冊回呼，來源 token 活得久（例如整個應用程式的 stopping token）而 linked source 沒被釋放，那些註冊會一直掛在上面，一輪迴圈累積一筆，最後就是記憶體往上爬。設了 `CancelAfter` 的 source 同理，沒 `Dispose` 就是留著一個計時器。

## 例外怎麼接

`OperationCanceledException` 代表的是**一個被要求的、正常的結束**，不是失敗。`TaskCanceledException` 是它的子類別，所以接父類別就兩個都接到了。

順序很要緊：

```csharp
try
{
    await RunAsync(files, token);
    logger.LogInformation("批次完成");
}
catch (OperationCanceledException) when (token.IsCancellationRequested)
{
    logger.LogInformation("批次已取消");   // 這是預期中的結束，用 Information
}
catch (Exception ex)
{
    logger.LogError(ex, "批次失敗");
}
```

`catch (Exception)` 若放在前面，取消會被它吞掉，然後這支 job 會以「失敗」的身分被記錄下來，或者更糟——被當成成功跑完。半夜被叫起來查一個根本不存在的錯誤，通常就是這樣來的。**取消不要記 Error**，否則監控上全是假警報。

兩個相關的細節：

- `await` 一個被取消的 `Task`，它的最終狀態是 `Canceled` 而不是 `Faulted`。`async` 方法一律如此，裡面漏出來的 `OperationCanceledException` 都會讓 Task 變成 `Canceled`。`Task.Run` 與 `Task.Factory.StartNew` 例外：它們要求例外帶的 token 與當初傳給 `Task.Run` 的那顆相符，不相符就算一般錯誤，變成 `Faulted`。所以兩件事要做——`Task.Run(..., token)` 把 token 傳進去，以及用 `ThrowIfCancellationRequested()` 而不是自己 `throw new OperationCanceledException()`，它會把 token 一起帶上
- `when (token.IsCancellationRequested)` 這個條件不是裝飾。`HttpClient` 自己的逾時也是丟 `TaskCanceledException`，沒有這個條件就分不出「使用者按了取消」跟「這個請求逾時了」

## 背景服務

前面的批次工作放進常駐服務裡，token 就不用自己建了，host 會給。`BackgroundService.ExecuteAsync(CancellationToken stoppingToken)` 的那個參數，就是應用程式的停止訊號。Ctrl+C（SIGINT）與外面叫它關機時送進來的 SIGTERM，都由 host 的 lifetime 接住，翻譯成這顆 token 的取消。你不必自己處理訊號，只要把 `stoppingToken` 一路往下傳。

```csharp
public class FileBatchService(
    ILogger<FileBatchService> logger,
    IFileSource source,
    IUploader uploader) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await RunBatchAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;   // 關機，正常離開迴圈
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "這一輪失敗，下一輪再試");   // 單輪失敗不該讓整個服務死掉
            }

            try
            {
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
            catch (OperationCanceledException) { break; }   // 等待期間收到關機
        }
    }

    private async Task RunBatchAsync(CancellationToken token)
    {
        foreach (var file in source.GetFiles())
        {
            token.ThrowIfCancellationRequested();

            string content = await File.ReadAllTextAsync(file, token);
            await uploader.UploadAsync(content, token);
        }
    }
}
```

要注意的是 host **不會無限期等你收尾**。送出取消之後它只等 `HostOptions.ShutdownTimeout`，時間到就不等了。這個值預設 30 秒（.NET 6 以前是 5 秒，.NET 7 起改為 30 秒）。收尾要寫進度、要 flush 日誌、要等當前交易做完的，不夠就自己調：

```csharp
// .NET 7 起預設 30 秒
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromMinutes(2));
builder.Services.AddHostedService<FileBatchService>();
```

調大之前先想清楚：關機被拖住的那段時間，外面叫停的人多半也在等。比較耐用的做法是讓收尾本身不需要時間——每處理完一筆就把進度寫回去，關機時什麼都不用補，下次啟動直接從斷點接。

ASP.NET Core 裡是同一套機制，token 換個來源而已：`HttpContext.RequestAborted` 在客戶端斷線時取消。使用者關掉頁面、按了重新整理，這顆 token 就會被按下。把它往下傳給資料庫查詢與 HTTP 呼叫，沒人在等的工作就不會繼續佔著資源做完。

## 速查表

| 情境 | 怎麼寫 |
| --- | --- |
| 要逾時 | `new CancellationTokenSource(TimeSpan.FromSeconds(30))` |
| 要能被使用者按鈕取消 | 自己存一顆 `CancellationTokenSource`，按鈕呼叫 `Cancel()` |
| 迴圈中途要停 | 每一圈開頭 `token.ThrowIfCancellationRequested()` |
| 要在取消時做收尾 | 改判 `if (token.IsCancellationRequested)`，收尾完 `return` |
| 逾時與使用者取消都要 | `CancellationTokenSource.CreateLinkedTokenSource(userToken, timeoutCts.Token)` |
| 常駐服務要能被 Ctrl+C 停 | `BackgroundService` 的 `stoppingToken` 往下傳，別另外建 source |
| 呼叫的 API 不收 token | 迴圈裡自己檢查；真的不能中斷就等它做完，別硬砍執行緒 |

| API | 用途與注意事項 |
| --- | --- |
| `CancellationTokenSource` | 遙控器。建立它的人負責 `Dispose`，其他人只拿 `.Token` |
| `CancelAfter(TimeSpan)` | 時間到自動取消。用了就一定要 `Dispose`，否則留著一個計時器 |
| `CreateLinkedTokenSource(...)` | 把數個 token 併成一個，任一取消就取消。**一定要 `using`**，否則回呼會掛在來源 token 上累積 |
| `token.ThrowIfCancellationRequested()` | 已取消就丟 `OperationCanceledException`，並帶上 token。不需收尾時的首選 |
| `token.IsCancellationRequested` | 只讀旗標，不丟例外。要先做收尾再離開時用 |
| `token.Register(callback)` | 取消時執行一段程式（關連線、通知別人）。回傳值是 `CancellationTokenRegistration`，要解除註冊就 `Dispose` 它；token 已取消時 callback 會當場同步執行 |
| `CancellationToken.None` | 明確表示「這裡不會被取消」，取代自己建一顆永不取消的 source |
| `OperationCanceledException` | 正常的結束訊號。`TaskCanceledException` 是它的子類別；`catch` 要放在 `catch (Exception)` 前面，記 Information 不記 Error |

## 參考資料

- [Cancellation in Managed Threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads)
- [CancellationToken Struct](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken)
- [CancellationTokenSource Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource)
- [CancellationTokenSource.CreateLinkedTokenSource](https://learn.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource.createlinkedtokensource)
- [OperationCanceledException Class](https://learn.microsoft.com/en-us/dotnet/api/system.operationcanceledexception)
- [Implement background tasks with hosted services](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [BackgroundService Class](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.backgroundservice)
- [HostOptions.ShutdownTimeout](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.hosting.hostoptions.shutdowntimeout)
- [HttpContext.RequestAborted](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.httpcontext.requestaborted)
