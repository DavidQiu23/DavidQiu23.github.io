---
title:  "一支 Console App 用參數跑多個 Job"
toc: true
toc_label: "目錄"
tags:
    - .NET
    - 架構
---

手上有兩支排程工作：對訂單、清暫存檔。全部包在同一支 exe 裡，第一個參數決定這次跑哪一個：

```
myapp SyncOrder
```

到點叫它的是 Windows 工作排程器或 Linux 的 cron，不是程式自己。跑完就結束。

## 資料夾結構

```
MyApp/
├── MyApp.csproj
├── Program.cs          進入點：建 host、解析參數、派送
├── JobName.cs          job 名字的 enum，也是 DI 的 key
├── IJob.cs             所有 job 的契約
└── Jobs/
    ├── SyncOrderJob.cs
    └── CleanupTempJob.cs
```

新增一個 job 要動的地方：`Jobs/` 底下加一個檔、`JobName` 加一個值、`Program.cs` 加一行註冊。

## 程式碼

### IJob.cs

```csharp
namespace MyApp;

public interface IJob
{
    Task RunAsync();
}
```

### JobName.cs

```csharp
namespace MyApp;

public enum JobName
{
    SyncOrder,
    CleanupTemp,
}
```

### Jobs/SyncOrderJob.cs

```csharp
using Microsoft.Extensions.Logging;

namespace MyApp.Jobs;

public class SyncOrderJob(ILogger<SyncOrderJob> logger) : IJob
{
    public Task RunAsync()
    {
        logger.LogInformation("對訂單");
        return Task.CompletedTask;
    }
}
```

### Jobs/CleanupTempJob.cs

```csharp
using Microsoft.Extensions.Logging;

namespace MyApp.Jobs;

public class CleanupTempJob(ILogger<CleanupTempJob> logger) : IJob
{
    public Task RunAsync()
    {
        logger.LogInformation("清暫存檔");
        return Task.CompletedTask;
    }
}
```

### Program.cs

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using MyApp.Jobs;

namespace MyApp;

public class Program
{
    // Environment.ExitCode 預設就是 0，成功的路徑什麼都不用做
    public static async Task Main(string[] args)
    {
        try
        {
            await RunAsync(args);
        }
        catch (Exception ex)
        {
            // 連建 host、讀設定檔炸掉的都要接住，否則離開碼不是 1
            Console.Error.WriteLine(ex);
            Environment.ExitCode = 1;
        }
    }

    private static async Task RunAsync(string[] args)
    {
        var builder = Host.CreateApplicationBuilder(args);

        // enum 當 key，一個 job 註冊一行。一律 scoped，job 要注入 DbContext 不必改這裡
        builder.Services.AddKeyedScoped<IJob, SyncOrderJob>(JobName.SyncOrder);
        builder.Services.AddKeyedScoped<IJob, CleanupTempJob>(JobName.CleanupTemp);

        using IHost host = builder.Build();

        // 只認跟 enum 一字不差的名字。不用 TryParse，它連 "0" 這種數字都吃
        string input = args.Length > 0 ? args[0] : "";

        if (!Enum.GetNames<JobName>().Contains(input))
        {
            Console.Error.WriteLine($"未知的 job：{(input.Length > 0 ? input : "(空的)")}");
            Console.Error.WriteLine($"可用的 job：{string.Join("、", Enum.GetNames<JobName>())}");
            Environment.ExitCode = 2;   // 參數不對
            return;
        }

        JobName name = Enum.Parse<JobName>(input);

        // scoped 的東西要從 scope 拿，不能從 host.Services 這個 root provider 拿
        using IServiceScope scope = host.Services.CreateScope();

        IJob job = scope.ServiceProvider.GetRequiredKeyedService<IJob>(name);
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();

        try
        {
            await job.RunAsync();
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "job 失敗：{Job}", name);
            Environment.ExitCode = 1;
        }
    }
}
```

執行結果：

```
myapp SyncOrder      對訂單
myapp CleanupTemp    清暫存檔
myapp syncorder      未知的 job：syncorder
                     可用的 job：SyncOrder、CleanupTemp（離開碼 2）
myapp 0              未知的 job：0（離開碼 2）
myapp 99             未知的 job：99（離開碼 2）
```

## 三個要件

**一、`IJob` 介面。** `Program.cs` 不認識任何一個具體的 job，只認識這個介面。`Jobs/` 底下的檔案互相也不認識。

**二、`enum` 當 key 註冊。** `AddKeyedScoped<IJob, SyncOrderJob>(JobName.SyncOrder)`。值錢的不是「編譯期擋打錯字」——把 `CleanupTempJob` 註冊到 `JobName.SyncOrder` 上照樣編譯過，enum 擋不了接錯線。值錢的是重新命名一次搬完、find-all-references 一次看完一支 job 的所有接線，字串 key 只能靠 grep。

代價是 `JobName` 與容器可能各說各話：`Enum.GetNames` 印出來的可用清單來自 enum，不是來自註冊。加了 enum 值卻忘了註冊，那個名字會出現在清單上，一打就是離開碼 1。

**三、名字要一字不差。** 先用 `Enum.GetNames` 比對，對不上就是名字打錯，印清單設離開碼 2。不要用 `Enum.TryParse` 當守門員——它連 `"0"`、`"SyncOrder,CleanupTemp"` 都收，`myapp 0` 會默默跑起第一支 job。使用者打什麼、`JobName` 寫什麼，兩邊完全一樣。

## keyed service

同一個介面註冊很多次，`GetService<IJob>()` 只會給最後註冊的那一個，沒辦法指定要哪一個。.NET 8 的 **keyed service** 在註冊時多帶一把鑰匙 (key)，取的時候用鑰匙指定：

```csharp
services.AddKeyedScoped<IJob, SyncOrderJob>(JobName.SyncOrder);

IJob? job = sp.GetKeyedService<IJob>(JobName.SyncOrder);          // 查不到回 null
IJob must = sp.GetRequiredKeyedService<IJob>(JobName.SyncOrder);  // 查不到拋例外
```

三種生命週期都有 keyed 版：`AddKeyedSingleton`、`AddKeyedScoped`、`AddKeyedTransient`。

**job 一律註冊成 scoped。** 一個行程只跑一支 job 就結束，singleton 省不到任何東西，卻換來一個陷阱：singleton 的 job 注入 `DbContext` 會變成俘虜相依，Development 有 `ValidateScopes` 會擋，Production 預設不擋——本機測得過，上線才爆。預設就 scoped，這整類錯誤不會發生。

對應地，`Program.cs` 從 `host.Services.CreateScope()` 解析 job。直接從 `host.Services` 這個 root provider 拿 scoped 的東西，在 Development 會丟 `Cannot resolve scoped service ... from root provider`，在 Production 不會丟——它會安靜地成功，然後那個 `DbContext` 活到行程結束。執行階段不會幫你擋，得自己記得開 scope。

兩個會踩的地方：

| 事 | 細節 |
| --- | --- |
| key 比對走 `Equals` | enum 比的是值，安全。改用字串 key 就要注意 `"Sync-Order"` 查不到 `"sync-order"` |
| **keyed 與非 keyed 是兩個世界** | `GetServices<IJob>()` 拿不到任何 keyed 註冊的實作，反過來也一樣 |

## 離開碼

排程器看不到程式印了什麼，只看離開碼：0 是成功，非 0 是失敗。Windows 工作排程器顯示在工作的「上次執行結果」，cron 則是非 0 才把輸出寄給 `MAILTO`。這是這支程式跟排程器之間唯一的介面，所以它跟「參數決定跑哪個 job」一樣是架構的一部分。

這篇用三個碼：0 成功、2 參數不對（名字打錯）、1 其他失敗。

**未捕捉的例外不會變成 1。** 它會變成執行階段自己的碼——Windows 上是 `0xE0434352`，排程器面板顯示的就是這串。所以 `Main` 最外層一定要有 `try/catch`，連建 host 跟讀設定檔都要包在裡面。

**`Environment.ExitCode` 不是 `Environment.Exit()`。** 前者只是設一個值，行程照常跑完再自然結束；後者當場砍掉行程，`finally` 不會跑、緩衝的日誌全部丟掉。job 裡兩個都不要碰，離開碼只在 `Program` 決定。

## 參考資料

- [.NET Generic Host](https://learn.microsoft.com/dotnet/core/extensions/generic-host)
- [Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)（含 keyed services 一節）
- [ServiceCollectionServiceExtensions.AddKeyedScoped 方法](https://learn.microsoft.com/dotnet/api/microsoft.extensions.dependencyinjection.servicecollectionserviceextensions.addkeyedscoped)
- [Enum.TryParse 方法](https://learn.microsoft.com/dotnet/api/system.enum.tryparse)
