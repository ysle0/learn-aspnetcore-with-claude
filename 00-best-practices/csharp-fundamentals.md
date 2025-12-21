# C# 기초 베스트 프랙티스

Effective C#에서 제시하는 핵심 원칙들입니다.

---

## 1. 지역 변수의 타입은 var를 사용하라

### 원칙

컴파일러가 타입을 추론할 수 있으면 var를 사용합니다.

```csharp
// ✅ 좋음: var 사용
var customers = new List<Customer>();
var query = customers.Where(c => c.Age > 18);
var customer = await GetCustomerAsync(id);

// ✅ 좋음: 타입이 명확하지 않을 때는 명시
IEnumerable<Customer> customers = GetCustomers();  // 반환 타입이 List인데 IEnumerable로 사용

// ❌ 나쁨: 불필요한 타입 명시
List<Customer> customers = new List<Customer>();
Dictionary<string, List<int>> mapping = new Dictionary<string, List<int>>();
```

### 예외: 타입이 중요할 때

```csharp
// 정수 리터럴의 타입이 중요할 때
long bigNumber = 10;  // int가 아닌 long임을 명시
decimal price = 19.99m;  // double이 아닌 decimal

// 인터페이스로 제한할 때
IList<Customer> list = new List<Customer>();  // IList로 제한
```

---

## 2. const보다 readonly를 선호하라

### 원칙

`const`는 컴파일 타임 상수, `readonly`는 런타임 상수입니다.

```csharp
public class Configuration
{
    // ❌ const: 값이 호출 어셈블리에 인라인됨
    // DLL 업데이트해도 호출 측은 이전 값 사용!
    public const int MaxRetries = 3;

    // ✅ readonly: 런타임에 값을 읽음
    public static readonly int MaxConnections = 100;

    // ✅ readonly: 생성자에서 초기화 가능
    public readonly string ConnectionString;

    public Configuration(string connectionString)
    {
        ConnectionString = connectionString;
    }
}
```

### const가 적절한 경우

```csharp
// 절대 변경되지 않는 값
public const double Pi = 3.14159265358979;
public const int BitsPerByte = 8;

// 성능 크리티컬한 경우 (switch 문의 case)
public const int StatusOk = 200;
public const int StatusNotFound = 404;
```

---

## 3. 캐스트보다 is, as 연산자를 선호하라

### 원칙

```csharp
// ❌ 나쁨: 캐스트 사용 (예외 발생 가능)
void Process(object obj)
{
    var customer = (Customer)obj;  // InvalidCastException 위험
    customer.DoSomething();
}

// ✅ 좋음: as 사용 (null 반환)
void Process(object obj)
{
    var customer = obj as Customer;
    if (customer != null)
    {
        customer.DoSomething();
    }
}

// ✅ 더 좋음: 패턴 매칭 (C# 7+)
void Process(object obj)
{
    if (obj is Customer customer)
    {
        customer.DoSomething();
    }
}

// ✅ switch 표현식 (C# 8+)
string GetDescription(object obj) => obj switch
{
    Customer c => $"Customer: {c.Name}",
    Order o => $"Order: {o.Id}",
    null => "null",
    _ => obj.GetType().Name
};
```

---

## 4. 문자열 보간을 사용하라

### 원칙

```csharp
var name = "John";
var age = 30;

// ❌ 나쁨: string.Format
var message = string.Format("Name: {0}, Age: {1}", name, age);

// ❌ 나쁨: 문자열 연결
var message = "Name: " + name + ", Age: " + age;

// ✅ 좋음: 문자열 보간
var message = $"Name: {name}, Age: {age}";

// ✅ 좋음: 포맷 지정
var price = 1234.567m;
var formatted = $"Price: {price:C2}";  // Price: $1,234.57
var date = DateTime.Now;
var dateStr = $"Date: {date:yyyy-MM-dd}";
```

### 문화권 고려

```csharp
// 문화권에 따라 결과가 달라질 수 있음
var price = 1234.56m;

// 현재 문화권 사용
var local = $"Price: {price:C}";

// 특정 문화권 명시 (FormattableString 활용)
FormattableString fs = $"Price: {price:C}";
var invariant = FormattableString.Invariant(fs);
var korean = fs.ToString(new CultureInfo("ko-KR"));
```

---

## 5. 문화권 정보를 명시하라

### 원칙

```csharp
// ❌ 나쁨: 문화권 미지정 (현재 스레드 문화권 사용)
var dateString = date.ToString();
var number = double.Parse(input);

// ✅ 좋음: 문화권 명시
var dateString = date.ToString("yyyy-MM-dd", CultureInfo.InvariantCulture);
var number = double.Parse(input, CultureInfo.InvariantCulture);

// 또는 명시적으로 현재 문화권 사용
var localDate = date.ToString("d", CultureInfo.CurrentCulture);
```

### StringComparison 사용

```csharp
// ❌ 나쁨: 기본 비교
if (str1 == str2) { }
if (str.Contains("text")) { }

// ✅ 좋음: 비교 방법 명시
if (str1.Equals(str2, StringComparison.OrdinalIgnoreCase)) { }
if (str.Contains("text", StringComparison.OrdinalIgnoreCase)) { }

// 정렬이 필요한 경우
if (str1.Equals(str2, StringComparison.CurrentCulture)) { }
```

---

## 6. 델리게이트로 콜백을 표현하라

### 원칙

```csharp
// ✅ 좋음: Action, Func 사용
public void Process(
    IEnumerable<Order> orders,
    Func<Order, bool> filter,
    Action<Order> process)
{
    foreach (var order in orders.Where(filter))
    {
        process(order);
    }
}

// 사용
Process(
    orders,
    o => o.Status == OrderStatus.Pending,
    o => Console.WriteLine(o.Id));

// ✅ 비동기 콜백
public async Task ProcessAsync(
    IEnumerable<Order> orders,
    Func<Order, Task> processAsync)
{
    foreach (var order in orders)
    {
        await processAsync(order);
    }
}
```

---

## 7. 이벤트 호출 시 null 조건 연산자 사용

### 원칙

```csharp
public class OrderProcessor
{
    public event EventHandler<OrderEventArgs>? OrderProcessed;

    // ❌ 나쁨: 레이스 컨디션 위험
    protected virtual void OnOrderProcessed(OrderEventArgs e)
    {
        if (OrderProcessed != null)
        {
            OrderProcessed(this, e);  // null 체크 후 null이 될 수 있음!
        }
    }

    // ❌ 나쁨: 임시 변수 (장황함)
    protected virtual void OnOrderProcessed(OrderEventArgs e)
    {
        var handler = OrderProcessed;
        if (handler != null)
        {
            handler(this, e);
        }
    }

    // ✅ 좋음: null 조건 연산자
    protected virtual void OnOrderProcessed(OrderEventArgs e)
    {
        OrderProcessed?.Invoke(this, e);
    }
}
```

---

## 8. 박싱과 언박싱을 최소화하라

### 원칙

```csharp
// ❌ 나쁨: 불필요한 박싱
int value = 42;
object obj = value;  // 박싱
int unboxed = (int)obj;  // 언박싱

// ❌ 나쁨: 컬렉션에서 박싱
ArrayList list = new ArrayList();
list.Add(42);  // 박싱!

// ✅ 좋음: 제네릭 사용
List<int> list = new List<int>();
list.Add(42);  // 박싱 없음

// ❌ 나쁨: 문자열 보간에서 박싱
int count = 100;
string message = $"Count: {count}";  // 박싱!

// ✅ 좋음: ToString() 호출
string message = $"Count: {count.ToString()}";  // 박싱 없음
// 또는 .NET 6+에서는 자동 최적화됨
```

---

## 9. IDisposable을 올바르게 구현하라

### 기본 패턴

```csharp
public class ResourceHolder : IDisposable
{
    private Stream? _stream;
    private bool _disposed;

    public ResourceHolder(string path)
    {
        _stream = File.OpenRead(path);
    }

    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        // 작업 수행
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // 관리 리소스 해제
            _stream?.Dispose();
            _stream = null;
        }

        // 비관리 리소스 해제 (있다면)

        _disposed = true;
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

### IAsyncDisposable (C# 8+)

```csharp
public class AsyncResourceHolder : IAsyncDisposable, IDisposable
{
    private HttpClient? _client;
    private bool _disposed;

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;

        if (_client != null)
        {
            // 비동기 정리 작업
            await Task.Delay(1);  // 예시
            _client.Dispose();
            _client = null;
        }

        _disposed = true;
        GC.SuppressFinalize(this);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _client?.Dispose();
        _client = null;
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}

// 사용
await using var holder = new AsyncResourceHolder();
```

---

## 10. nameof 연산자를 활용하라

### 원칙

```csharp
public class Customer
{
    private string _name = "";

    public string Name
    {
        get => _name;
        set
        {
            // ❌ 나쁨: 하드코딩된 문자열
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("Name cannot be empty", "name");

            // ✅ 좋음: nameof 사용
            ArgumentException.ThrowIfNullOrEmpty(value, nameof(value));

            _name = value;
        }
    }
}

// INotifyPropertyChanged에서 활용
public class ViewModel : INotifyPropertyChanged
{
    private string _title = "";

    public string Title
    {
        get => _title;
        set
        {
            _title = value;
            // ✅ nameof로 리팩토링에 안전
            OnPropertyChanged(nameof(Title));
        }
    }
}

// 로깅에서 활용
public void Process(Order order)
{
    _logger.LogInformation("Entering {MethodName}", nameof(Process));
}
```

---

## 11. 튜플과 분해를 활용하라

### 원칙

```csharp
// ✅ 여러 값 반환
public (bool Success, string Message, Order? Order) TryCreateOrder(OrderRequest request)
{
    if (!IsValid(request))
        return (false, "Invalid request", null);

    var order = new Order(request);
    return (true, "Success", order);
}

// ✅ 분해 사용
var (success, message, order) = TryCreateOrder(request);
if (success)
{
    Console.WriteLine($"Created order: {order!.Id}");
}

// ✅ 일부만 필요할 때
var (success, _, order) = TryCreateOrder(request);

// ✅ 분해 지원 추가
public class Point
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) => (X, Y) = (x, y);

    public void Deconstruct(out int x, out int y)
    {
        x = X;
        y = Y;
    }
}

var point = new Point(10, 20);
var (x, y) = point;
```

---

## 12. null 관련 베스트 프랙티스

### Nullable Reference Types 활용

```csharp
#nullable enable

public class CustomerService
{
    private readonly ICustomerRepository _repository;

    // ✅ null 가능성 명시
    public Customer? GetById(int id)
    {
        return _repository.Find(id);
    }

    // ✅ null 불가 보장
    public Customer GetByIdOrThrow(int id)
    {
        return _repository.Find(id)
            ?? throw new CustomerNotFoundException(id);
    }

    // ✅ null 매개변수 검증
    public void Update(Customer customer)
    {
        ArgumentNullException.ThrowIfNull(customer);
        _repository.Update(customer);
    }
}

// null 병합 연산자 활용
string name = customer?.Name ?? "Unknown";

// null 병합 할당 연산자
_cache ??= LoadCache();

// null 조건 연산자 체이닝
var city = customer?.Address?.City;
```

---

## 13. record 타입 활용 (C# 9+)

### 원칙

```csharp
// ✅ 불변 데이터에 record 사용
public record Customer(int Id, string Name, string Email);

// 자동으로 제공되는 기능:
// - Equals, GetHashCode (값 기반)
// - ToString
// - Deconstruct
// - with 표현식

var customer = new Customer(1, "John", "john@example.com");
var updated = customer with { Email = "newemail@example.com" };

// ✅ record struct (C# 10+)
public readonly record struct Point(int X, int Y);

// ✅ 위치 매개변수 없이도 가능
public record Person
{
    public required string Name { get; init; }
    public required int Age { get; init; }
}
```

---

## 14. 패턴 매칭 활용

### 원칙

```csharp
// ✅ switch 표현식
public decimal CalculateDiscount(Customer customer) => customer switch
{
    { Tier: CustomerTier.Gold, YearsActive: > 5 } => 0.20m,
    { Tier: CustomerTier.Gold } => 0.15m,
    { Tier: CustomerTier.Silver } => 0.10m,
    { YearsActive: > 10 } => 0.05m,
    _ => 0m
};

// ✅ 리스트 패턴 (C# 11+)
int[] numbers = { 1, 2, 3, 4, 5 };

var result = numbers switch
{
    [] => "empty",
    [var single] => $"single: {single}",
    [var first, .., var last] => $"first: {first}, last: {last}",
};

// ✅ 관계 패턴
string GetCategory(int age) => age switch
{
    < 13 => "Child",
    < 20 => "Teen",
    < 65 => "Adult",
    _ => "Senior"
};
```

---

## 면접 예상 질문

### Q1: var를 사용해야 할 때와 하지 말아야 할 때는?

**A:** 타입이 명확할 때(new, 캐스트, 리터럴) var를 사용합니다. 반환 타입이 불명확하거나, 특정 인터페이스로 제한하고 싶을 때는 명시적 타입을 사용합니다.

### Q2: const와 readonly의 차이점은?

**A:**
- **const**: 컴파일 타임 상수, 호출 측에 인라인됨, 프리미티브/문자열만 가능
- **readonly**: 런타임 상수, 참조로 접근, 모든 타입 가능, 생성자에서 초기화 가능

### Q3: record를 언제 사용해야 하나요?

**A:** 불변 데이터, DTO, 값 기반 동등성이 필요한 경우에 사용합니다. 식별자 기반 동등성이 필요한 엔티티에는 class를 사용합니다.

---

## 참고 자료

- Effective C# (Bill Wagner)
- C# in Depth (Jon Skeet)
- [Microsoft C# 코딩 규칙](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions)

---

## 다음 문서

→ [async-await.md](./async-await.md) - 비동기 프로그래밍 원칙
