# 비동기 프로그래밍 베스트 프랙티스

More Effective C#과 ASP.NET Core 가이드라인의 비동기 프로그래밍 원칙입니다.

---

## 1. async void를 피하라

### 원칙

```csharp
// ❌ 나쁨: async void
public async void ProcessOrderAsync(Order order)
{
    await _repository.SaveAsync(order);
    // 예외 발생 시 호출자가 catch 불가능!
    // 프로세스 크래시 위험
}

// ✅ 좋음: async Task
public async Task ProcessOrderAsync(Order order)
{
    await _repository.SaveAsync(order);
    // 예외가 Task에 캡처되어 호출자가 처리 가능
}
```

### 유일한 예외: 이벤트 핸들러

```csharp
// ✅ 이벤트 핸들러는 async void 허용
private async void Button_Click(object sender, EventArgs e)
{
    try
    {
        await ProcessAsync();
    }
    catch (Exception ex)
    {
        // 반드시 예외 처리!
        _logger.LogError(ex, "Button click failed");
        ShowErrorMessage(ex.Message);
    }
}
```

---

## 2. 동기와 비동기를 섞지 마라 (sync-over-async)

### 원칙

```csharp
// ❌ 매우 나쁨: .Result, .Wait() 사용 (데드락 위험!)
public string GetData()
{
    return GetDataAsync().Result;  // 데드락!
}

public void Process()
{
    GetDataAsync().Wait();  // 데드락!
}

// ❌ 나쁨: GetAwaiter().GetResult()도 마찬가지
public string GetData()
{
    return GetDataAsync().GetAwaiter().GetResult();
}

// ✅ 좋음: 비동기로 끝까지
public async Task<string> GetDataAsync()
{
    return await _httpClient.GetStringAsync(url);
}

// ✅ 컨트롤러도 비동기로
[HttpGet]
public async Task<IActionResult> Get()
{
    var data = await GetDataAsync();
    return Ok(data);
}
```

### 데드락이 발생하는 이유

```
┌─────────────────────────────────────────────────────────────────┐
│                    데드락 시나리오                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UI 스레드 / ASP.NET 동기화 컨텍스트:                           │
│                                                                 │
│  1. 호출: var result = GetDataAsync().Result;                  │
│  2. GetDataAsync 시작, await에서 비동기 작업 시작               │
│  3. .Result가 현재 스레드를 블록하고 대기                       │
│  4. 비동기 작업 완료, 원래 컨텍스트로 돌아가려고 함              │
│  5. 하지만 그 스레드는 .Result에서 블록 중!                     │
│  6. 💀 데드락!                                                  │
│                                                                 │
│  ASP.NET Core는 동기화 컨텍스트가 없어서 덜 위험하지만          │
│  스레드 고갈 문제가 발생할 수 있음                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. ConfigureAwait(false)를 라이브러리에서 사용하라

### 원칙

```csharp
// 라이브러리 코드 (애플리케이션이 아닌)
public class MyLibrary
{
    public async Task<string> GetDataAsync()
    {
        // ✅ 라이브러리에서는 ConfigureAwait(false)
        var response = await _httpClient.GetAsync(url)
            .ConfigureAwait(false);

        var content = await response.Content.ReadAsStringAsync()
            .ConfigureAwait(false);

        return content;
    }
}

// ✅ 애플리케이션 코드에서는 생략 가능 (ASP.NET Core)
// ASP.NET Core는 동기화 컨텍스트가 없어서 영향 없음
public async Task<IActionResult> Get()
{
    var data = await _myLibrary.GetDataAsync();  // ConfigureAwait 불필요
    return Ok(data);
}
```

### 왜 사용하는가?

```csharp
// ConfigureAwait(false) 없이:
// 1. await 후 원래 컨텍스트(스레드)로 돌아가려고 시도
// 2. 컨텍스트 스위칭 오버헤드 발생

// ConfigureAwait(false) 사용:
// 1. await 후 아무 스레드에서나 계속 실행 가능
// 2. 성능 향상, 데드락 방지 (라이브러리에서)
```

---

## 4. 취소(Cancellation)를 항상 지원하라

### 원칙

```csharp
// ❌ 나쁨: 취소 불가
public async Task<List<Order>> GetOrdersAsync()
{
    return await _dbContext.Orders.ToListAsync();
}

// ✅ 좋음: CancellationToken 지원
public async Task<List<Order>> GetOrdersAsync(CancellationToken cancellationToken = default)
{
    return await _dbContext.Orders.ToListAsync(cancellationToken);
}

// ✅ 컨트롤러에서 자동 전파
[HttpGet]
public async Task<IActionResult> GetOrders(CancellationToken cancellationToken)
{
    // ASP.NET Core가 요청 취소 시 자동으로 토큰 취소
    var orders = await _orderService.GetOrdersAsync(cancellationToken);
    return Ok(orders);
}

// ✅ 장시간 작업에서 취소 확인
public async Task ProcessLargeDataAsync(CancellationToken cancellationToken)
{
    foreach (var item in items)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await ProcessItemAsync(item, cancellationToken);
    }
}
```

### 취소 연결

```csharp
public async Task ProcessWithTimeoutAsync(CancellationToken cancellationToken)
{
    // 타임아웃과 외부 취소를 결합
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromSeconds(30));

    try
    {
        await LongRunningOperationAsync(cts.Token);
    }
    catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)
    {
        throw new TimeoutException("Operation timed out");
    }
}
```

---

## 5. ValueTask를 적절히 사용하라

### 원칙

```csharp
// ❌ 항상 Task 사용
public async Task<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
    {
        return value;  // Task<int> 할당!
    }
    return await LoadFromDbAsync(key);
}

// ✅ 동기 완료가 빈번하면 ValueTask 사용
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
    {
        return ValueTask.FromResult(value);  // 할당 없음!
    }
    return new ValueTask<int>(LoadFromDbAsync(key));
}
```

### ValueTask 사용 규칙

```csharp
// ⚠️ ValueTask는 한 번만 await 가능!

// ❌ 나쁨: 여러 번 await
var task = GetCachedValueAsync("key");
var result1 = await task;
var result2 = await task;  // 정의되지 않은 동작!

// ❌ 나쁨: Task.WhenAll에 직접 사용
var tasks = new[] { GetCachedValueAsync("a"), GetCachedValueAsync("b") };
await Task.WhenAll(tasks);  // 컴파일 안됨

// ✅ 좋음: AsTask()로 변환 후 사용
var tasks = keys.Select(k => GetCachedValueAsync(k).AsTask());
await Task.WhenAll(tasks);
```

---

## 6. 병렬 처리가 필요하면 Task.WhenAll 사용

### 원칙

```csharp
// ❌ 나쁨: 순차 실행 (느림)
var user = await GetUserAsync(userId);
var orders = await GetOrdersAsync(userId);
var preferences = await GetPreferencesAsync(userId);
// 총 시간 = user + orders + preferences

// ✅ 좋음: 병렬 실행 (빠름)
var userTask = GetUserAsync(userId);
var ordersTask = GetOrdersAsync(userId);
var preferencesTask = GetPreferencesAsync(userId);

await Task.WhenAll(userTask, ordersTask, preferencesTask);

var user = userTask.Result;  // 이미 완료됨, 안전
var orders = ordersTask.Result;
var preferences = preferencesTask.Result;
// 총 시간 = max(user, orders, preferences)

// ✅ 더 깔끔하게
var (user, orders, preferences) = await (
    GetUserAsync(userId),
    GetOrdersAsync(userId),
    GetPreferencesAsync(userId));
```

### 에러 처리

```csharp
// 모든 예외 수집
var tasks = items.Select(ProcessAsync);
try
{
    await Task.WhenAll(tasks);
}
catch
{
    // Task.WhenAll은 첫 번째 예외만 throw
    // 모든 예외를 보려면:
    var exceptions = tasks
        .Where(t => t.IsFaulted)
        .SelectMany(t => t.Exception!.InnerExceptions);

    foreach (var ex in exceptions)
    {
        _logger.LogError(ex, "Task failed");
    }
    throw;
}
```

---

## 7. 비동기 코드에서 lock 대신 SemaphoreSlim 사용

### 원칙

```csharp
// ❌ 나쁨: lock 안에서 await
private readonly object _lock = new();

public async Task ProcessAsync()
{
    lock (_lock)  // lock 안에서 await 불가!
    {
        await DoWorkAsync();  // 컴파일 에러
    }
}

// ✅ 좋음: SemaphoreSlim 사용
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task ProcessAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        await DoWorkAsync();
    }
    finally
    {
        _semaphore.Release();
    }
}

// ✅ 더 안전하게: 확장 메서드
public static class SemaphoreExtensions
{
    public static async Task<IDisposable> LockAsync(this SemaphoreSlim semaphore)
    {
        await semaphore.WaitAsync();
        return new SemaphoreReleaser(semaphore);
    }

    private class SemaphoreReleaser(SemaphoreSlim semaphore) : IDisposable
    {
        public void Dispose() => semaphore.Release();
    }
}

// 사용
public async Task ProcessAsync()
{
    using (await _semaphore.LockAsync())
    {
        await DoWorkAsync();
    }
}
```

---

## 8. Task.Run을 올바르게 사용하라

### 원칙

```csharp
// ❌ 나쁨: IO 바운드 작업에 Task.Run 사용
public async Task<string> GetDataAsync()
{
    return await Task.Run(async () =>
    {
        return await _httpClient.GetStringAsync(url);  // 불필요!
    });
}

// ✅ 좋음: IO 바운드는 직접 await
public async Task<string> GetDataAsync()
{
    return await _httpClient.GetStringAsync(url);
}

// ✅ 좋음: CPU 바운드 작업에 Task.Run 사용
public async Task<int> CalculateAsync()
{
    return await Task.Run(() =>
    {
        // CPU 집약적 작업
        return HeavyCalculation();
    });
}

// ✅ ASP.NET Core에서는 Task.Run 거의 불필요
// (요청마다 스레드 풀에서 실행됨)
```

---

## 9. 비동기 Dispose 패턴

### 원칙

```csharp
public class AsyncResource : IAsyncDisposable
{
    private readonly Stream _stream;
    private bool _disposed;

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;

        await _stream.DisposeAsync();

        GC.SuppressFinalize(this);
    }
}

// ✅ await using 사용
await using var resource = new AsyncResource();
await resource.UseAsync();
// 자동으로 DisposeAsync 호출
```

---

## 10. 비동기 스트림 활용 (IAsyncEnumerable)

### 원칙

```csharp
// ❌ 나쁨: 모든 데이터를 메모리에 로드
public async Task<List<Order>> GetOrdersAsync()
{
    return await _dbContext.Orders.ToListAsync();
}

// ✅ 좋음: 스트리밍으로 메모리 절약
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    await foreach (var order in _dbContext.Orders.AsAsyncEnumerable()
        .WithCancellation(cancellationToken))
    {
        yield return order;
    }
}

// 사용
await foreach (var order in GetOrdersAsync(cancellationToken))
{
    await ProcessOrderAsync(order, cancellationToken);
}
```

---

## 안티패턴 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                    비동기 안티패턴                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. async void (이벤트 핸들러 제외)                             │
│  2. .Result, .Wait() 사용                                      │
│  3. Task.Run으로 IO 작업 래핑                                   │
│  4. CancellationToken 무시                                     │
│  5. lock 안에서 await 시도                                     │
│  6. 모든 곳에서 ConfigureAwait(false) (앱 코드)                │
│  7. ValueTask를 여러 번 await                                  │
│  8. 불필요한 async/await (pass-through)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 불필요한 async/await

```csharp
// ❌ 불필요한 async/await
public async Task<int> GetCountAsync()
{
    return await _repository.GetCountAsync();
}

// ✅ 직접 반환 (단, 예외 처리 동작이 달라질 수 있음)
public Task<int> GetCountAsync()
{
    return _repository.GetCountAsync();
}

// ⚠️ using이 있으면 async 필요!
public async Task<string> ReadFileAsync()
{
    using var reader = new StreamReader(path);
    return await reader.ReadToEndAsync();
    // async 없으면 reader가 너무 일찍 Dispose됨!
}
```

---

## 면접 예상 질문

### Q1: async void를 사용하면 안 되는 이유는?

**A:** 예외가 호출자에게 전파되지 않고, 예외 발생 시 프로세스가 크래시될 수 있습니다. Task를 반환해야 호출자가 예외를 처리하고 작업 완료를 기다릴 수 있습니다.

### Q2: ConfigureAwait(false)는 언제 사용하나요?

**A:** 라이브러리 코드에서 사용하여 원래 동기화 컨텍스트로 돌아가지 않게 합니다. 성능 향상과 데드락 방지에 도움됩니다. ASP.NET Core 애플리케이션 코드에서는 불필요합니다.

### Q3: ValueTask와 Task의 차이점은?

**A:** ValueTask는 동기적으로 완료되는 경우가 많을 때 힙 할당을 피할 수 있습니다. 단, 한 번만 await 가능하고 동시에 여러 번 대기할 수 없습니다.

---

## 참고 자료

- More Effective C# (Bill Wagner)
- [비동기 프로그래밍 모범 사례](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/)
- [ASP.NET Core 성능 모범 사례](https://learn.microsoft.com/aspnet/core/performance/performance-best-practices)

---

## 다음 문서

→ [performance.md](./performance.md) - 성능 최적화 원칙
