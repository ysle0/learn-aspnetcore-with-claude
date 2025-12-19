# 서비스 생명주기

## 개요

ASP.NET Core DI 컨테이너는 세 가지 서비스 생명주기를 지원합니다: **Singleton**, **Scoped**, **Transient**. 올바른 생명주기 선택은 애플리케이션의 성능과 정확성에 중요합니다.

---

## 세 가지 생명주기

### 시각적 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                    서비스 생명주기 비교                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Singleton (앱 전체에서 하나)                                    │
│  ─────────────────────────────────────────────────────────────  │
│  │         Instance A                                       │  │
│  │ App Start ────────────────────────────────────► App End  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Scoped (HTTP 요청마다 하나)                                     │
│  ─────────────────────────────────────────────────────────────  │
│  │ Request 1 │    Instance A    │                            │  │
│  │ Request 2 │    Instance B    │                            │  │
│  │ Request 3 │    Instance C    │                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Transient (주입마다 새로 생성)                                   │
│  ─────────────────────────────────────────────────────────────  │
│  │ Request 1 │ A │ B │ C │                                   │  │
│  │ Request 2 │ D │ E │                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Singleton

### 특징

- 앱 시작부터 종료까지 **하나의 인스턴스**
- 모든 요청, 모든 스레드에서 **공유**
- **스레드 안전(Thread-Safe)**해야 함

### 등록 방법

```csharp
// 타입 등록
services.AddSingleton<ICacheService, MemoryCacheService>();

// 인스턴스 등록
services.AddSingleton<ICacheService>(new MemoryCacheService());

// 팩토리 등록
services.AddSingleton<ICacheService>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new MemoryCacheService(config["CacheSize"]);
});
```

### 적합한 사용 사례

```csharp
// ✅ 캐시 서비스
public class CacheService : ICacheService
{
    private readonly ConcurrentDictionary<string, object> _cache = new();

    public void Set(string key, object value) => _cache[key] = value;
    public object? Get(string key) => _cache.TryGetValue(key, out var v) ? v : null;
}

// ✅ 설정/옵션
public class AppSettings
{
    public string ApiKey { get; init; }
    public string BaseUrl { get; init; }
}

// ✅ 상태 없는 유틸리티
public class HashingService : IHashingService
{
    public string ComputeHash(string input)
    {
        using var sha = SHA256.Create();
        var bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToBase64String(bytes);
    }
}

// ✅ HttpClient 팩토리 (내부적으로 풀링)
services.AddHttpClient<IApiClient, ApiClient>();
```

### 주의사항

```csharp
// ❌ 상태 있는 Singleton - 위험!
public class BadSingleton
{
    private int _requestCount;  // 모든 요청이 공유

    public void Process()
    {
        _requestCount++;  // 레이스 컨디션!
    }
}

// ✅ 스레드 안전하게 수정
public class SafeSingleton
{
    private int _requestCount;

    public void Process()
    {
        Interlocked.Increment(ref _requestCount);
    }
}
```

---

## Scoped

### 특징

- **HTTP 요청(스코프)당 하나의 인스턴스**
- 같은 요청 내에서 여러 번 주입해도 **같은 인스턴스**
- 요청 종료 시 자동 **Dispose**

### 등록 방법

```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
```

### 적합한 사용 사례

```csharp
// ✅ DbContext - 가장 대표적인 Scoped 서비스
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
// AddDbContext는 기본적으로 Scoped로 등록

// ✅ 현재 사용자 정보
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _accessor;

    public CurrentUserService(IHttpContextAccessor accessor)
    {
        _accessor = accessor;
    }

    public string? UserId =>
        _accessor.HttpContext?.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
}

// ✅ Unit of Work 패턴
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context) => _context = context;

    public Task<int> SaveChangesAsync() => _context.SaveChangesAsync();
}
```

### 같은 요청 내 공유 확인

```csharp
public class ServiceA
{
    private readonly Guid _id = Guid.NewGuid();
    public Guid Id => _id;
}

public class Consumer1(ServiceA serviceA)
{
    public Guid GetServiceId() => serviceA.Id;
}

public class Consumer2(ServiceA serviceA)
{
    public Guid GetServiceId() => serviceA.Id;
}

app.MapGet("/test", (Consumer1 c1, Consumer2 c2) =>
{
    // Scoped면 같은 ID
    // Transient면 다른 ID
    return new { C1 = c1.GetServiceId(), C2 = c2.GetServiceId() };
});
```

---

## Transient

### 특징

- **주입할 때마다 새 인스턴스**
- 가장 짧은 수명
- 경량 서비스에 적합

### 등록 방법

```csharp
services.AddTransient<IValidator<Order>, OrderValidator>();
services.AddTransient<IEmailBuilder, EmailBuilder>();
```

### 적합한 사용 사례

```csharp
// ✅ 검증 로직
public class OrderValidator : IValidator<Order>
{
    public ValidationResult Validate(Order order)
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(order.CustomerName))
            errors.Add("CustomerName is required");
        if (order.Items.Count == 0)
            errors.Add("Order must have items");
        return new ValidationResult(errors.Count == 0, errors);
    }
}

// ✅ 팩토리에서 생성되는 객체
public class EmailBuilderFactory
{
    private readonly IServiceProvider _sp;

    public EmailBuilderFactory(IServiceProvider sp) => _sp = sp;

    public IEmailBuilder Create()
    {
        return _sp.GetRequiredService<IEmailBuilder>();  // 매번 새 인스턴스
    }
}

// ✅ 짧은 수명의 연산
public class GuidGenerator : IGuidGenerator
{
    public Guid Generate() => Guid.NewGuid();
}
```

---

## 생명주기 선택 가이드

### 의사결정 플로우

```
┌─────────────────────────────────────────────────────────────────┐
│                    생명주기 선택 가이드                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  서비스가 상태를 가지고 있는가?                                   │
│      │                                                          │
│      ├── No ──► 무거운 초기화가 필요한가?                        │
│      │              │                                           │
│      │              ├── Yes ──► Singleton                       │
│      │              └── No ──► Transient                        │
│      │                                                          │
│      └── Yes ──► 요청 간 공유되어야 하는가?                      │
│                     │                                           │
│                     ├── Yes ──► 스레드 안전한가?                 │
│                     │              │                            │
│                     │              ├── Yes ──► Singleton        │
│                     │              └── No ──► Scoped 또는 재설계 │
│                     │                                           │
│                     └── No ──► Scoped                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 서비스 유형별 권장 생명주기

| 서비스 유형 | 권장 생명주기 | 이유 |
|------------|-------------|------|
| DbContext | Scoped | 트랜잭션/연결 관리 |
| HttpClient | Singleton* | 연결 풀링 (*IHttpClientFactory 사용) |
| Logger | Singleton | 상태 없음 |
| Cache | Singleton | 앱 전체 공유 |
| CurrentUser | Scoped | 요청별 사용자 |
| Validator | Transient | 경량, 상태 없음 |
| Repository | Scoped | DbContext와 동일 수명 |
| Background Worker | Singleton | 앱 수명과 동일 |

---

## Captive Dependency 문제

### 문제 상황

```csharp
// ❌ Singleton이 Scoped를 주입받음
services.AddSingleton<SingletonService>();
services.AddScoped<ScopedService>();

public class SingletonService
{
    private readonly ScopedService _scoped;  // 첫 요청의 인스턴스가 포획됨!

    public SingletonService(ScopedService scoped)
    {
        _scoped = scoped;
    }
}
```

### 문제 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                    Captive Dependency                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1:                                                     │
│    Singleton 생성 ──► ScopedService A 주입 (포획됨)             │
│                                                                 │
│  Request 2:                                                     │
│    Singleton 재사용 ──► 여전히 ScopedService A 사용! ❌          │
│    (새 ScopedService B가 생성되어야 하는데...)                   │
│                                                                 │
│  Request 3:                                                     │
│    Singleton 재사용 ──► 여전히 ScopedService A 사용! ❌          │
│                                                                 │
│  결과: ScopedService A가 앱 전체 수명동안 유지됨                 │
│       → DbContext라면 연결 고갈, 데이터 불일치 발생 가능          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 해결 방법 1: IServiceScopeFactory

```csharp
public class SingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public SingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public void DoWork()
    {
        using var scope = _scopeFactory.CreateScope();
        var scoped = scope.ServiceProvider.GetRequiredService<ScopedService>();
        scoped.Process();
    }  // scope Dispose 시 ScopedService도 Dispose
}
```

### 해결 방법 2: Func<T> 팩토리

```csharp
services.AddScoped<ScopedService>();
services.AddSingleton<Func<ScopedService>>(sp =>
{
    return () =>
    {
        var scope = sp.CreateScope();
        return scope.ServiceProvider.GetRequiredService<ScopedService>();
    };
});

public class SingletonService
{
    private readonly Func<ScopedService> _scopedFactory;

    public SingletonService(Func<ScopedService> scopedFactory)
    {
        _scopedFactory = scopedFactory;
    }

    public void DoWork()
    {
        var scoped = _scopedFactory();  // 호출 시 새 인스턴스
        scoped.Process();
    }
}
```

### 검증 활성화

```csharp
var builder = WebApplication.CreateBuilder(args);

// 개발 환경에서 Captive Dependency 검증
if (builder.Environment.IsDevelopment())
{
    builder.Host.UseDefaultServiceProvider(options =>
    {
        options.ValidateScopes = true;  // Scope 검증
        options.ValidateOnBuild = true; // 빌드 시 검증
    });
}
```

---

## Dispose 패턴

### 자동 Dispose

```csharp
public class DisposableService : IDisposable
{
    public void Dispose()
    {
        Console.WriteLine("DisposableService disposed");
    }
}

// Scoped: 요청 끝에 Dispose
// Singleton: 앱 종료 시 Dispose
// Transient: 스코프 끝에 Dispose (주의!)
```

### Transient의 Dispose 주의점

```csharp
// ❌ Transient가 제대로 Dispose 되지 않을 수 있음
services.AddTransient<IDisposableService, DisposableService>();

app.MapGet("/", (IDisposableService service) =>
{
    // service는 요청 끝에 Dispose됨 (Scoped처럼 동작)
    return "OK";
});

// ❌ 직접 ServiceProvider에서 가져오면 Dispose 안됨!
app.MapGet("/bad", (IServiceProvider sp) =>
{
    var service = sp.GetRequiredService<IDisposableService>();
    // Transient인데 요청 스코프에 캡처되어 요청 끝에 Dispose
    // 하지만 루트 ServiceProvider에서 가져오면 앱 종료까지 Dispose 안됨!
    return "Bad";
});
```

### IAsyncDisposable

```csharp
public class AsyncDisposableService : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await Task.Delay(100);  // 비동기 정리 작업
        Console.WriteLine("Async disposed");
    }
}

// IAsyncDisposable도 자동으로 처리됨
services.AddScoped<AsyncDisposableService>();
```

---

## 면접 예상 질문

### Q1: Scoped 서비스를 Background Service에서 어떻게 사용하나요?

**A:** `IServiceScopeFactory`를 주입받아 명시적으로 스코프를 생성합니다:

```csharp
public class MyBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MyBackgroundService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var dbContext = scope.ServiceProvider
                .GetRequiredService<AppDbContext>();

            // DbContext 사용
            await dbContext.Orders.ToListAsync(stoppingToken);

            await Task.Delay(60000, stoppingToken);
        }
    }
}
```

### Q2: ValidateScopes 옵션의 역할은?

**A:** 개발 환경에서 Captive Dependency 문제를 감지합니다. Singleton이 Scoped 서비스를 직접 주입받으려 하면 예외가 발생합니다.

### Q3: Transient 서비스도 Dispose 되나요?

**A:** 네, 하지만 스코프에 의존합니다. HTTP 요청 컨텍스트에서 주입되면 요청 스코프 종료 시 Dispose됩니다. 하지만 루트 ServiceProvider에서 직접 해결하면 앱 종료까지 Dispose되지 않습니다.

---

## 참고 자료

- [서비스 수명](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#service-lifetimes)
- [Scope 검증](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#scope-validation)
- [Background 서비스에서 Scoped 서비스 사용](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services#consuming-a-scoped-service-in-a-background-task)

---

## 다음 문서

→ [internals.md](./internals.md) - DI 컨테이너 내부 동작
