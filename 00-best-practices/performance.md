# 성능 최적화 베스트 프랙티스

Effective C#과 .NET 성능 가이드라인의 핵심 원칙입니다.

---

## 1. 할당을 최소화하라

### 원칙

```csharp
// ❌ 나쁨: 불필요한 할당
public string GetFullName(string first, string last)
{
    return first + " " + last;  // 중간 문자열 할당
}

// ✅ 좋음: 문자열 보간 (단일 할당)
public string GetFullName(string first, string last)
{
    return $"{first} {last}";
}

// ✅ 더 좋음: 반복 호출 시 StringBuilder
public string BuildReport(IEnumerable<Order> orders)
{
    var sb = new StringBuilder();
    foreach (var order in orders)
    {
        sb.AppendLine($"Order {order.Id}: {order.Total:C}");
    }
    return sb.ToString();
}

// ✅ 최고: Span으로 스택 할당
public bool TryParseCoordinate(ReadOnlySpan<char> input, out (int x, int y) result)
{
    Span<Range> ranges = stackalloc Range[2];
    var count = input.Split(ranges, ',');
    // ...
}
```

---

## 2. 박싱을 피하라

### 원칙

```csharp
// ❌ 나쁨: 박싱 발생
int count = 100;
object obj = count;  // 박싱!
string message = $"Count: {count}";  // 박싱! (일부 .NET 버전)

// ✅ 좋음: 제네릭 사용
List<int> numbers = new();  // ArrayList 대신
numbers.Add(42);  // 박싱 없음

// ✅ 좋음: ToString() 직접 호출
string message = $"Count: {count.ToString()}";

// ❌ 나쁨: 인터페이스로 구조체 사용
IComparable comparable = 42;  // 박싱!

// ✅ 좋음: 제네릭 제약 사용
public void Sort<T>(T[] items) where T : IComparable<T>
{
    // T가 구조체여도 박싱 없음
}
```

---

## 3. ArrayPool과 ObjectPool 활용

### 원칙

```csharp
// ❌ 나쁨: 매번 새 배열 할당
public byte[] ReadData()
{
    var buffer = new byte[4096];  // 매번 할당!
    stream.Read(buffer);
    return buffer;
}

// ✅ 좋음: ArrayPool 사용
public void ProcessData()
{
    var buffer = ArrayPool<byte>.Shared.Rent(4096);
    try
    {
        var bytesRead = stream.Read(buffer);
        Process(buffer.AsSpan(0, bytesRead));
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}

// ✅ ObjectPool 사용 (복잡한 객체)
public class ExpensiveObjectPoolPolicy : PooledObjectPolicy<ExpensiveObject>
{
    public override ExpensiveObject Create() => new ExpensiveObject();
    public override bool Return(ExpensiveObject obj)
    {
        obj.Reset();
        return true;
    }
}

// 등록
services.AddSingleton<ObjectPool<ExpensiveObject>>(sp =>
{
    var policy = new ExpensiveObjectPoolPolicy();
    return new DefaultObjectPool<ExpensiveObject>(policy);
});

// 사용
public class MyService(ObjectPool<ExpensiveObject> pool)
{
    public void Process()
    {
        var obj = pool.Get();
        try
        {
            obj.DoWork();
        }
        finally
        {
            pool.Return(obj);
        }
    }
}
```

---

## 4. Span<T>과 Memory<T> 활용

### 원칙

```csharp
// ❌ 나쁨: 부분 문자열 추출 (새 문자열 할당)
public string GetFirstWord(string input)
{
    var spaceIndex = input.IndexOf(' ');
    return spaceIndex > 0 ? input.Substring(0, spaceIndex) : input;
}

// ✅ 좋음: Span으로 할당 없이 처리
public ReadOnlySpan<char> GetFirstWord(ReadOnlySpan<char> input)
{
    var spaceIndex = input.IndexOf(' ');
    return spaceIndex > 0 ? input[..spaceIndex] : input;
}

// ✅ stackalloc 활용
public void ProcessSmallData()
{
    Span<int> buffer = stackalloc int[128];  // 스택 할당
    for (int i = 0; i < buffer.Length; i++)
    {
        buffer[i] = i * 2;
    }
}

// ✅ 조건부 스택/힙 할당
public void ProcessData(int size)
{
    Span<byte> buffer = size <= 256
        ? stackalloc byte[size]
        : new byte[size];

    // buffer 사용
}
```

---

## 5. LINQ 성능 주의사항

### 원칙

```csharp
// ❌ 나쁨: 중복 열거
var list = GetItems();  // IEnumerable<Item>
if (list.Any())
{
    var first = list.First();     // 다시 열거
    var count = list.Count();     // 또 다시 열거
}

// ✅ 좋음: ToList/ToArray로 구체화
var list = GetItems().ToList();
if (list.Count > 0)
{
    var first = list[0];
    var count = list.Count;
}

// ❌ 나쁨: 핫 패스에서 LINQ 체이닝
public bool IsValid(Order order)
{
    return order.Items.Where(i => i.IsActive).Any(i => i.Quantity > 0);
}

// ✅ 좋음: 단순 루프 (핫 패스에서)
public bool IsValid(Order order)
{
    foreach (var item in order.Items)
    {
        if (item.IsActive && item.Quantity > 0)
            return true;
    }
    return false;
}

// ✅ 또는 최적화된 LINQ
public bool IsValid(Order order)
{
    return order.Items.Any(i => i.IsActive && i.Quantity > 0);
}
```

---

## 6. 캐싱 활용

### 원칙

```csharp
// ❌ 나쁨: 매번 정규식 컴파일
public bool IsValidEmail(string email)
{
    return Regex.IsMatch(email, @"^[\w-\.]+@[\w-\.]+\.\w+$");
}

// ✅ 좋음: 컴파일된 정규식 캐싱
private static readonly Regex EmailRegex = new(
    @"^[\w-\.]+@[\w-\.]+\.\w+$",
    RegexOptions.Compiled);

public bool IsValidEmail(string email)
{
    return EmailRegex.IsMatch(email);
}

// ✅ 더 좋음: Source Generator (.NET 7+)
[GeneratedRegex(@"^[\w-\.]+@[\w-\.]+\.\w+$")]
private static partial Regex EmailRegex();

public bool IsValidEmail(string email)
{
    return EmailRegex().IsMatch(email);
}
```

### 메모이제이션

```csharp
// ✅ 비용이 큰 계산 결과 캐싱
public class ExpensiveCalculator
{
    private readonly ConcurrentDictionary<int, int> _cache = new();

    public int Calculate(int input)
    {
        return _cache.GetOrAdd(input, key =>
        {
            // 비용이 큰 계산
            Thread.Sleep(1000);
            return key * key;
        });
    }
}
```

---

## 7. 구조체 vs 클래스

### 원칙

```csharp
// ✅ 구조체가 적합한 경우:
// - 16바이트 이하
// - 불변
// - 자주 할당/해제
// - 박싱 되지 않음

public readonly struct Point(int x, int y)
{
    public int X { get; } = x;
    public int Y { get; } = y;
}

// ❌ 구조체가 부적합한 경우:
// - 16바이트 초과 (복사 비용)
// - 가변
// - 인터페이스로 자주 사용 (박싱)

// ✅ readonly struct로 방어적 복사 방지
public readonly struct ImmutablePoint
{
    public int X { get; init; }
    public int Y { get; init; }
}

// ✅ ref struct로 스택에만 할당
public ref struct SpanWrapper
{
    public Span<byte> Data;
}
```

---

## 8. 문자열 처리 최적화

### 원칙

```csharp
// ❌ 나쁨: 문자열 연결 반복
public string Build(IEnumerable<string> items)
{
    string result = "";
    foreach (var item in items)
    {
        result += item + ", ";  // 매번 새 문자열!
    }
    return result;
}

// ✅ 좋음: StringBuilder
public string Build(IEnumerable<string> items)
{
    var sb = new StringBuilder();
    foreach (var item in items)
    {
        if (sb.Length > 0) sb.Append(", ");
        sb.Append(item);
    }
    return sb.ToString();
}

// ✅ 더 좋음: string.Join
public string Build(IEnumerable<string> items)
{
    return string.Join(", ", items);
}

// ✅ 문자열 비교 최적화
// ❌
if (str.ToLower() == "test") { }

// ✅
if (str.Equals("test", StringComparison.OrdinalIgnoreCase)) { }

// ✅ 빈 문자열 확인
if (string.IsNullOrEmpty(str)) { }
if (string.IsNullOrWhiteSpace(str)) { }
```

---

## 9. 컬렉션 최적화

### 원칙

```csharp
// ✅ 용량 미리 지정
var list = new List<int>(1000);  // 재할당 방지
var dict = new Dictionary<string, int>(1000);

// ✅ 적절한 컬렉션 선택
// - 순서 있는 접근: List<T>
// - 빠른 조회: Dictionary<K,V>, HashSet<T>
// - FIFO: Queue<T>
// - LIFO: Stack<T>
// - 정렬 필요: SortedSet<T>, SortedDictionary<K,V>

// ✅ 읽기 전용 노출
public class OrderService
{
    private readonly List<Order> _orders = new();

    // ❌ List<T> 직접 노출
    public List<Order> Orders => _orders;

    // ✅ IReadOnlyList<T> 노출
    public IReadOnlyList<Order> Orders => _orders;
}

// ✅ FrozenSet/FrozenDictionary (.NET 8+)
// 불변이고 조회가 빈번한 경우
private static readonly FrozenSet<string> ValidCodes =
    new HashSet<string> { "A", "B", "C" }.ToFrozenSet();
```

---

## 10. HttpClient 재사용

### 원칙

```csharp
// ❌ 매우 나쁨: 매번 HttpClient 생성
public async Task<string> GetDataAsync()
{
    using var client = new HttpClient();  // 소켓 고갈!
    return await client.GetStringAsync(url);
}

// ✅ 좋음: IHttpClientFactory 사용
builder.Services.AddHttpClient<IMyApiClient, MyApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
});

public class MyApiClient(HttpClient httpClient) : IMyApiClient
{
    public async Task<string> GetDataAsync()
    {
        return await httpClient.GetStringAsync("/data");
    }
}

// ✅ Named 클라이언트
builder.Services.AddHttpClient("github", client =>
{
    client.BaseAddress = new Uri("https://api.github.com");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
});

// 사용
public class GitHubService(IHttpClientFactory factory)
{
    public async Task<string> GetRepoAsync(string repo)
    {
        var client = factory.CreateClient("github");
        return await client.GetStringAsync($"/repos/{repo}");
    }
}
```

---

## 11. 지연 초기화

### 원칙

```csharp
// ✅ Lazy<T> 활용
public class ExpensiveService
{
    private readonly Lazy<ExpensiveObject> _expensive;

    public ExpensiveService()
    {
        _expensive = new Lazy<ExpensiveObject>(() =>
        {
            // 처음 접근할 때만 생성
            return new ExpensiveObject();
        });
    }

    public void DoWork()
    {
        var obj = _expensive.Value;  // 여기서 생성
        obj.Process();
    }
}

// ✅ 널 병합 할당 연산자
private ExpensiveObject? _cached;

public ExpensiveObject GetObject()
{
    return _cached ??= CreateExpensiveObject();
}

// ✅ LazyInitializer (스레드 안전)
private ExpensiveObject? _cached;

public ExpensiveObject GetObject()
{
    return LazyInitializer.EnsureInitialized(ref _cached, CreateExpensiveObject);
}
```

---

## 12. 고성능 로깅

### 원칙

```csharp
// ❌ 나쁨: 로그 레벨 무시하고 문자열 생성
_logger.LogDebug($"Processing order {order.Id} with {order.Items.Count} items");
// Debug 비활성화여도 문자열 보간 실행!

// ✅ 좋음: 구조화된 로깅
_logger.LogDebug("Processing order {OrderId} with {ItemCount} items",
    order.Id, order.Items.Count);
// Debug 비활성화면 문자열 생성 안함

// ✅ 더 좋음: LoggerMessage Source Generator
public static partial class LoggerExtensions
{
    [LoggerMessage(Level = LogLevel.Debug,
        Message = "Processing order {OrderId} with {ItemCount} items")]
    public static partial void LogOrderProcessing(
        this ILogger logger, int orderId, int itemCount);
}

// 사용
_logger.LogOrderProcessing(order.Id, order.Items.Count);
// 최고 성능, 박싱 없음
```

---

## 안티패턴 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                    성능 안티패턴                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 루프 안에서 문자열 연결 (+)                                 │
│  2. 매번 HttpClient 생성                                        │
│  3. 핫 패스에서 LINQ 체이닝                                     │
│  4. 불필요한 ToList()/ToArray()                                │
│  5. 용량 미지정 컬렉션 생성                                     │
│  6. 매번 정규식 컴파일                                          │
│  7. 큰 구조체 복사                                              │
│  8. 박싱 발생하는 코드                                          │
│  9. 로그 레벨 무시한 문자열 보간                                │
│  10. 동기화 컨텍스트에서 .Result/.Wait()                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 면접 예상 질문

### Q1: ArrayPool을 사용해야 하는 이유는?

**A:** 빈번한 배열 할당/해제로 인한 GC 압력을 줄입니다. 풀에서 빌려 쓰고 반환하므로 힙 할당이 줄어들고, 특히 대용량 객체 힙(LOH) 단편화를 방지합니다.

### Q2: Span<T>의 장점은?

**A:** 힙, 스택, 네이티브 메모리에 대한 통합된 뷰를 제공하며, 슬라이싱 시 복사 없이 원본 메모리를 참조합니다. stackalloc과 함께 사용하면 힙 할당 없이 작업 가능합니다.

### Q3: HttpClient를 매번 생성하면 안 되는 이유는?

**A:** HttpClient는 Dispose해도 소켓이 즉시 해제되지 않습니다 (TIME_WAIT 상태). 빈번한 생성/해제는 소켓 고갈과 성능 저하를 유발합니다. IHttpClientFactory가 연결을 재사용하고 DNS 변경도 처리합니다.

---

## 참고 자료

- Effective C# (Bill Wagner)
- [.NET 성능 팁](https://learn.microsoft.com/dotnet/framework/performance/performance-tips)
- [ASP.NET Core 성능 모범 사례](https://learn.microsoft.com/aspnet/core/performance/performance-best-practices)
- [High-Performance .NET Code](https://www.writinghighperf.net/)

---

## 다음 문서

→ [aspnetcore.md](./aspnetcore.md) - ASP.NET Core 가이드라인
