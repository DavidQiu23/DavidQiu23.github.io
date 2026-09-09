---
title:  ".NET 並行處理：Task、async-await 與 Parallel"
toc: true
toc_label: "目錄"
tags:
    - .NET
    - 並行處理
---

一支程式要打 100 次 API 抓資料。一次一秒，循序跑完是 100 秒。太慢，希望它們同時做。

.NET 手上有三套東西：`Task`、`async`/`await`、`Parallel`。三者常被混著講，但解決的是不同的問題。選錯不只沒變快，還可能更慢。

## 執行緒：一個做事的人手

後面每一節都繞著這個詞轉，先定義清楚。

**執行緒 (thread) 是程式裡一個做事的人手。** 程式預設只有一個，一行一行從頭做到尾。要快，就得多幾個人手一起做。

人手不是無限的。.NET 備有一個**執行緒池 (thread pool)**，性質接近人力派遣公司：要用時跟它借，用完還回去。池子裡的人數有限，借光了，後面的工作只能排隊。

全篇的差別都圍繞同一個問題：**這個人手是真的在做事，還是站著發呆等？**

## 慢有兩種，藥也不同

程式慢，先分清楚慢在哪裡。

**第一種是慢在等別人。** 打 API、查資料庫、讀寫檔案。程式本身沒在做事，只是把請求送出去，站著等對方回話。這類工作叫 **IO-bound**。

**第二種是慢在自己算。** 壓縮圖片、加解密、大量數學運算。CPU 忙得冒煙，一秒都沒閒著。這類工作叫 **CPU-bound**。

兩種慢，解法完全不同。前者是「等的時候先去做別件」，術語叫**並行 (concurrency)**；後者是「真的多顆核心同時算」，術語叫**平行 (parallelism)**。中文只差一個字，指的是兩件事。

| | 慢在等別人 (IO-bound) | 慢在自己算 (CPU-bound) |
| --- | --- | --- |
| 例子 | HTTP 請求、資料庫查詢、讀檔 | 壓縮、加解密、大量運算 |
| 問題 | 人手站著空等，浪費 | 只有一個人在算，太慢 |
| 要的是 | 等的時候把人手放走，去做別的 | 多幾個人手一起算 |
| 用什麼 | `async`/`await` + `Task.WhenAll` | `Parallel` |

搞混的下場很具體：

- 把「等 API」的工作丟給 `Parallel`：叫來 8 個人手，8 個全部站著等網路。網路不會因此變快
- 把「自己算」的工作寫成 `async`：程式碼裡根本沒有可等待的點，只是多繞一層，一點也沒快

分辨的標準只有一句：這段程式碼慢，是 CPU 在忙，還是它在發呆。

## Task：一件之後會完成的工作

`Task` 是一件工作的收據。工作交出去，拿到一張收據，之後憑收據拿結果。

```csharp
// 交出去：找個人手去算 Calculate(100)
Task<int> task = Task.Run(() => Calculate(100));

// 憑收據拿結果（還沒算完就在這裡等）
int result = await task;
```

### Task 與 async/await 的關係

兩者常被當成同一件事，實際上是兩層：

- **`Task` 是型別**，是那張收據，一個物件
- **`async`/`await` 是語法**，是開收據與拿收據的寫法。編譯器會把標了 `async` 的方法拆成一台**狀態機 (state machine)**：每個 `await` 是一個中斷點，記住跑到哪、局部變數是什麼，之後再從那裡接回去

分開也各自成立。沒有 `async` 一樣能用 Task，早期都寫 `ContinueWith`；`await` 也不是只認 Task，`ValueTask` 同樣可以 await。

真正該分清楚的不是語法，是人手。同樣拿到一張 Task 收據，來源差很多：

| 寫法 | 有沒有人手在忙 |
| --- | --- |
| `Task.Run(() => Calc())` | 有。跟池子借一個人手，真的佔著在算 |
| `await client.GetStringAsync(url)` | **沒有**。請求交給作業系統的 IO，等待期間不佔任何執行緒，回應到了才叫人接手 |

這條線就是上一節那兩種慢的分界。「把打 API 的程式包進 `Task.Run` 讓它變快」之所以是錯的，原因也在這裡——包了只是多借一個人手站著等。

以下先談 Task 這張收據的用法，下一節再談 `async`/`await` 的寫法。

### 多件一起等：WhenAll

先看慢的寫法。一筆一筆做，每次都等它回來才做下一筆：

```csharp
var results = new List<string>();

foreach (var id in ids)
{
    string one = await FetchAsync(id);   // 停在這裡等這一筆回來
    results.Add(one);
}
// 100 筆、每筆 1 秒 = 100 秒
```

快的寫法分兩步。**第一步，全部先發出去，先不等**：

```csharp
var tasks = new List<Task<string>>();

foreach (var id in ids)
{
    tasks.Add(FetchAsync(id));   // 沒有 await，所以不會停下來等
}
// 迴圈跑完，100 個請求都已經在路上，手上是 100 張收據
```

關鍵在**沒有 `await`**。呼叫 `FetchAsync` 會立刻回傳一張 Task 收據，工作在背後繼續進行，程式往下跑下一圈迴圈。

**第二步，一次等這 100 張收據**：

```csharp
string[] results = await Task.WhenAll(tasks);
// 100 筆同時在跑，總共大約 1 秒
```

`results` 的順序跟 `tasks` 一樣：`tasks[0]` 的結果就是 `results[0]`，這是官方保證的，不必自己排。

熟了之後，第一步通常用 LINQ 寫成一行：

```csharp
var tasks = ids.Select(id => FetchAsync(id)).ToList();
string[] results = await Task.WhenAll(tasks);
```

`.ToList()` 別省。`Select` 是延遲執行的，不走訪就不會真的去呼叫 `FetchAsync`；`ToList()` 走訪一遍，請求才在這一行全部發出去。

### 只要最快的：WhenAny

```csharp
Task<string> winner = await Task.WhenAny(taskA, taskB);
string result = await winner;   // WhenAny 回傳的是「贏的那張收據」，要再 await 一次才拿到值
```

### WhenAll 的例外只會噴一個

多件工作同時炸掉時，`await` 只重新拋出**第一個**例外，其餘的靜靜消失。要看全部，得從 Task 物件本身取：

```csharp
var all = Task.WhenAll(tasks);
try
{
    await all;
}
catch
{
    // all.Exception 是一個大袋子 (AggregateException)，裡面才裝著全部
    foreach (var ex in all.Exception!.InnerExceptions)
    {
        Console.WriteLine(ex.Message);
    }
}
```

## async / await：等的時候把人手放走

`await` 的重點不是等，是**等的時候把人手還回池子**。

```csharp
public async Task<string> FetchAsync(string url)
{
    using var client = new HttpClient();

    // 請求送出去之後，這個人手就被放走，可以去做別的事
    // 回應到了，才再跟池子要一個人手，接著跑下一行
    string body = await client.GetStringAsync(url);

    return body.Trim();
}
```

1000 個 HTTP 請求只要少少幾個人手就扛得完，因為大部分時間根本沒人在等。

換成 `Parallel.ForEach` 配同步版的 `GetString`，則會叫來一堆人手，每一個都杵著等網路。池子被借光，程式其他地方連人都調不到。

{: .notice--warning}
> `await` 前後不保證是同一個人手。執行到 `await` 時人手被放走，回來時很可能已經換人——`await` 之後那段程式碼叫**續行 (continuation)**，由誰執行不固定。所以任何認人的東西跨過 `await` 就不能用。例如 `Mutex`，官方文件寫明它「enforces thread identity」——鎖只有取得它的那個執行緒能解，這個性質叫**執行緒親和性 (thread affinity)**。細節見[互斥鎖那篇]({% link _posts/2026-08-04-跨行程async互斥鎖.md %})。

### 三個常見的坑

**用 `.Result` 或 `.Wait()` 硬要結果**

```csharp
// 危險
string s = FetchAsync(url).Result;

// 正確：async 一路往上傳，呼叫端也用 await
string s = await FetchAsync(url);
```

`.Result` 的意思是「就站在這裡等到有結果為止」。人手被卡住，白白佔著池子。

更糟的是**死結 (deadlock)**。舊版 ASP.NET、WinForms、WPF 各有一個管事的調度員，規定續行必須回到原本那個執行緒才能繼續跑，這個調度員叫**同步環境 (SynchronizationContext)**。而那個執行緒正被 `.Result` 卡著不動——它在等工作完成，工作在等它讓開，兩邊互相等，程式就此卡死，而且沒有任何錯誤訊息。

ASP.NET Core 與 Console 沒有這種「一次只准一段程式碼跑」的同步環境，不會死結，但浪費人手依然成立——ASP.NET Core 官方效能指南就明列「不要呼叫 `Task.Wait` 或 `Task.Result`」，因為那會導致執行緒池枯竭 (thread pool starvation)。

**寫成 `async void`**

```csharp
async void DoWork()     // 不行：例外外面 catch 不到，通常直接讓行程當掉
async Task DoWork()     // 正確
```

原因是 `async Task` 的例外會被收在那張 Task 收據上，`async void` 沒有收據可收，例外直接丟到當時的同步環境上，呼叫端的 `try`/`catch` 攔不到。

唯一的例外是 UI 的事件處理常式，按鈕點擊之類的簽章只能這樣寫。

**忘記 await**

```csharp
SaveAsync(data);           // 工作丟出去就不管了，出錯也沒人知道
await SaveAsync(data);     // 正確
```

編譯器會給 CS4014 警告，值得認真看待：沒 await，例外會靜靜消失，而且呼叫端不會等它做完就往下跑。真的要「射後不理」，寫成 `_ = SaveAsync(data);` 讓意圖明確。

## Parallel：多幾個人手一起算

`Parallel` 把資料切成幾段，分給多個人手同時算，全部算完才往下走。切段這件事叫**分割 (partitioning)**，而「同一份工作套用到一堆資料上」這種平行方式叫**資料平行 (data parallelism)**。

```csharp
// 0 到 999999 這段，切開分給多個人手跑
Parallel.For(0, 1_000_000, i => Compute(i));

// 集合的每個元素，分給多個人手跑
Parallel.ForEach(files, file => Compress(file));

// 三件不相干的事，同時做（這種叫工作平行 task parallelism）
Parallel.Invoke(
    () => BuildReportA(),
    () => BuildReportB(),
    () => BuildReportC());
```

這是給「自己在算」的工作用的。裡面若是打 API，等於叫一堆人來罰站。

### 限制同時幾個人手

`MaxDegreeOfParallelism` 預設是 `-1`，意思是**不設上限**：`For` 與 `ForEach` 底層排程器給多少執行緒就用多少。要壓低——例如下游服務擋不住那麼多請求——用 `ParallelOptions`：

```csharp
var options = new ParallelOptions { MaxDegreeOfParallelism = 4 };
Parallel.ForEach(files, options, file => Compress(file));
```

`Parallel.ForEachAsync` 是例外：它的 `-1` 代表 `Environment.ProcessorCount`，也就是不指定時最多同時跑核心數個。

`ParallelOptions` 也可以放 `CancellationToken`，中途要取消時用得到。

### Break 與 Stop 不一樣

跑到一半想提早結束，有兩種寫法，一個迴圈只能挑一種：

```csharp
// 「做到這裡為止」：編號比現在小的都還會跑完，比現在大而還沒開始的不會再跑
Parallel.For(0, 100, (i, state) =>
{
    if (found) state.Break();
});

// 「立刻收工」：其他人手能停就停，誰跑過誰沒跑不保證
Parallel.For(0, 100, (i, state) =>
{
    if (found) state.Stop();
});
```

找到東西就閃人用 `Stop`；資料本身有順序、要確保前面的都處理完用 `Break`。

`ForEach` 一樣有，多吃一個 `state` 參數就行：

```csharp
Parallel.ForEach(files, (file, state) =>
{
    if (Scan(file).IsMatch) state.Stop();
});
```

`Parallel.Invoke` 沒有——它收的是一組互不相干的 `Action`，沒有迴圈狀態可傳。`Parallel.ForEachAsync` 也沒有，要中止改用 `CancellationToken`。

三個細節：

- 同一個迴圈裡混用 `Break` 與 `Stop` 會丟 `InvalidOperationException`，一個迴圈只能挑一種
- `Break` 只擋「還沒開始」的，已經在跑的不會被打斷。要讓它們也提早收手，得在迴圈裡自己檢查 `state.ShouldExitCurrentIteration` 與 `state.LowestBreakIteration`
- `Break` 的語意（編號比我小的都要做完）建立在索引順序上，官方定位是給資料本身有順序的搜尋用。`ForEach` 跑一般集合時，通常要的是 `Stop`

### 例外會被裝進袋子

多個人手可能同時出錯，所以例外不會單獨拋出，而是全部裝進一個 `AggregateException`：

```csharp
try
{
    Parallel.ForEach(items, item => Process(item));
}
catch (AggregateException ex)
{
    foreach (var inner in ex.InnerExceptions)   // 袋子裡才是真正的例外
    {
        Console.WriteLine(inner.Message);
    }
}
```

### ForEachAsync：兩邊都要的時候

有一種需求是「一堆 API 要打，但不能一次全衝出去」。.NET 6 之後有 `Parallel.ForEachAsync`：

```csharp
var options = new ParallelOptions { MaxDegreeOfParallelism = 10 };

await Parallel.ForEachAsync(urls, options, async (url, ct) =>
{
    var body = await client.GetStringAsync(url, ct);
    await SaveAsync(body, ct);
});
```

與 `Task.WhenAll` 的差別在**節流 (throttling)**：`WhenAll` 全部一起衝，1000 筆就 1000 筆同時出去；`ForEachAsync` 可以設上限，一次只放 10 筆。對方有速率限制時用這個。

## 共用同一份資料會壞

這是三者共同的坑，而且錯得很安靜。

```csharp
var results = new List<int>();

Parallel.ForEach(items, item =>
{
    results.Add(Process(item));   // 有時候少幾筆，有時候直接爆例外
});
```

`List<T>` 沒有考慮多人同時使用，也就是**不是執行緒安全的 (thread-safe)**。兩個人手同時 `Add`，內部狀態就亂了。

`i++` 同樣不安全。它看起來是一個動作，實際上是三個：讀出來、加一、寫回去。兩個人手交錯執行就會少算——結果取決於誰先誰後，這種錯誤叫**競爭條件 (race condition)**，難重現也難查。

三種解法：

```csharp
// 1. 換成本來就允許多人同時使用的集合（執行緒安全的集合）
var results = new ConcurrentBag<int>();
Parallel.ForEach(items, item => results.Add(Process(item)));

// 2. 純粹的加減計數，用 Interlocked（讀、加、寫一次做完，中間插不進來，這叫原子操作 atomic）
int count = 0;
Parallel.ForEach(items, item => Interlocked.Increment(ref count));

// 3. 真的要保護一段程式碼，才用 lock（同時只准一個人手進去）
var locker = new object();
Parallel.ForEach(items, item =>
{
    var r = Process(item);          // 重活留在鎖外面，否則大家排隊等，等於沒平行
    lock (locker) { results.Add(r); }
});
```

{: .notice--warning}
> `lock` 區塊裡**不能**寫 `await`，編譯器會直接擋下來（CS1996: *Cannot await in the body of a lock statement*）。理由就是上面說的：鎖認執行緒，而 `await` 之後可能換人。async 情境要互斥得改用 `SemaphoreSlim`，完整做法見[互斥鎖那篇]({% link _posts/2026-08-04-跨行程async互斥鎖.md %})。

最省事的做法是根本不共用。每個工作各自回傳結果，最後再合併：

```csharp
var results = await Task.WhenAll(items.Select(ProcessAsync));   // 沒有共用的東西，就沒有這些問題
```

## 怎麼選

| 情境 | 用什麼 |
| --- | --- |
| 一堆 API 或查詢，全部一起衝 | `await Task.WhenAll(...)` |
| 一堆 API 或查詢，要限制同時幾筆 | `await Parallel.ForEachAsync(...)` |
| 一堆計算工作 | `Parallel.ForEach` / `Parallel.For` |
| 幾件不相干的事同時做 | `Parallel.Invoke` |
| 一件很重的計算，不想卡住畫面 | `await Task.Run(() => ...)` |
| 只有一筆 IO 工作 | 直接 `await`，不必包東西 |

兩個補充：

- **量少的時候不要平行。** 切資料、調人手、合結果都有成本。官方的說法是：迴圈次數少、每次做的事又快，平行化幾乎快不了，某些情況甚至更慢——要不要用，實測過再決定
- **ASP.NET Core 裡要克制。** 每個請求本來就已經佔著一個人手。在請求處理中再用 `Parallel` 搶核心，同時湧入 100 個請求時會直接拖垮整台機器

## 參考資料

- [Task-based asynchronous programming](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-based-asynchronous-programming)
- [Asynchronous programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- [Data Parallelism (Task Parallel Library)](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/data-parallelism-task-parallel-library)
- [Thread-Safe Collections](https://learn.microsoft.com/en-us/dotnet/standard/collections/thread-safe/)
- [Exception handling (Task Parallel Library)](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/exception-handling-task-parallel-library)
- [ParallelOptions.MaxDegreeOfParallelism](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.paralleloptions.maxdegreeofparallelism)
- [ParallelLoopState.Break](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.parallelloopstate.break)
- [Interlocked Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked)
- [Mutex Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex)
- [Async/Await - Best Practices in Asynchronous Programming](https://learn.microsoft.com/en-us/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming)
- [ASP.NET Core Best Practices](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices)
- [Potential Pitfalls in Data and Task Parallelism](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism)
- [Resolve errors and warnings that involve async, await](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-messages/async-await-errors)
