# DI 컨테이너 내부 동작

## 개요

이 문서에서는 ASP.NET Core의 내장 DI 컨테이너(`Microsoft.Extensions.DependencyInjection`)의 내부 구현을 살펴봅니다.

---

## 핵심 구성요소

### ServiceDescriptor

```csharp
// 서비스 등록 정보를 담는 클래스
public class ServiceDescriptor
{
    public Type ServiceType { get; }           // 인터페이스 타입
    public Type? ImplementationType { get; }   // 구현 타입
    public object? ImplementationInstance { get; }  // 인스턴스 (Singleton)
    public Func<IServiceProvider, object>? ImplementationFactory { get; }  // 팩토리
    public ServiceLifetime Lifetime { get; }   // Singleton/Scoped/Transient
    public object? ServiceKey { get; }         // Keyed Services용 (.NET 8+)
}

// 등록 시 내부적으로 ServiceDescriptor 생성
services.AddScoped<IOrderService, OrderService>();
// ↓ 내부적으로
services.Add(new ServiceDescriptor(
    typeof(IOrderService),
    typeof(OrderService),
    ServiceLifetime.Scoped));
```

### ServiceCollection

```csharp
// IServiceCollection의 기본 구현
public class ServiceCollection : IServiceCollection
{
    private readonly List<ServiceDescriptor> _descriptors = new();

    public int Count => _descriptors.Count;

    public void Add(ServiceDescriptor descriptor)
    {
        _descriptors.Add(descriptor);
    }

    // IEnumerable<ServiceDescriptor> 구현
    public IEnumerator<ServiceDescriptor> GetEnumerator()
        => _descriptors.GetEnumerator();
}
```

---

## BuildServiceProvider 과정

### 1단계: ServiceProvider 생성

```csharp
// services.BuildServiceProvider() 호출 시
public static ServiceProvider BuildServiceProvider(
    this IServiceCollection services,
    ServiceProviderOptions options)
{
    return new ServiceProvider(services, options);
}

// ServiceProviderOptions
public class ServiceProviderOptions
{
    public bool ValidateScopes { get; set; }   // Scope 검증
    public bool ValidateOnBuild { get; set; }  // 빌드 시 검증
}
```

### 2단계: CallSite 생성

```
┌─────────────────────────────────────────────────────────────────┐
│                    CallSite 체인                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  서비스 요청: IOrderService                                     │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ConstructorCallSite (OrderService)                     │   │
│  │  │                                                      │   │
│  │  ├── ServiceCallSite (IOrderRepository)                │   │
│  │  │    └── ConstructorCallSite (OrderRepository)        │   │
│  │  │         └── ServiceCallSite (AppDbContext)          │   │
│  │  │                                                      │   │
│  │  └── ServiceCallSite (ILogger<OrderService>)           │   │
│  │       └── FactoryCallSite (Logger<T> 팩토리)            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CallSite = 서비스 인스턴스를 생성하는 방법을 설명하는 객체       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### CallSite 종류

```csharp
// 내부 클래스들 (단순화)
abstract class ServiceCallSite
{
    public Type ServiceType { get; }
    public ServiceLifetime Lifetime { get; }
}

class ConstructorCallSite : ServiceCallSite
{
    public ConstructorInfo Constructor { get; }
    public ServiceCallSite[] Parameters { get; }  // 생성자 파라미터들의 CallSite
}

class FactoryCallSite : ServiceCallSite
{
    public Func<IServiceProvider, object> Factory { get; }
}

class ConstantCallSite : ServiceCallSite
{
    public object DefaultValue { get; }  // 미리 생성된 인스턴스
}

class IEnumerableCallSite : ServiceCallSite
{
    public ServiceCallSite[] ServiceCallSites { get; }  // 다중 등록
}
```

---

## 서비스 해결 과정

### GetService 흐름

```csharp
// 간략화된 서비스 해결 로직
public object? GetService(Type serviceType)
{
    // 1. CallSite 조회 (캐시됨)
    var callSite = _callSiteFactory.GetCallSite(serviceType);
    if (callSite == null)
        return null;

    // 2. Lifetime에 따라 해결
    return callSite.Lifetime switch
    {
        ServiceLifetime.Singleton => ResolveSingleton(callSite),
        ServiceLifetime.Scoped => ResolveScoped(callSite),
        ServiceLifetime.Transient => ResolveTransient(callSite),
    };
}
```

### Singleton 해결

```csharp
private object ResolveSingleton(ServiceCallSite callSite)
{
    // 루트 스코프의 캐시에서 찾거나 생성
    if (!_singletonCache.TryGetValue(callSite, out var instance))
    {
        lock (_syncLock)
        {
            if (!_singletonCache.TryGetValue(callSite, out instance))
            {
                instance = CreateInstance(callSite);
                _singletonCache[callSite] = instance;
            }
        }
    }
    return instance;
}
```

### Scoped 해결

```csharp
private object ResolveScoped(ServiceCallSite callSite)
{
    // 현재 스코프의 캐시에서 찾거나 생성
    var scope = _currentScope;

    if (!scope.ResolvedServices.TryGetValue(callSite, out var instance))
    {
        instance = CreateInstance(callSite);
        scope.ResolvedServices[callSite] = instance;

        // Dispose 추적
        if (instance is IDisposable disposable)
        {
            scope.Disposables.Add(disposable);
        }
    }
    return instance;
}
```

### Transient 해결

```csharp
private object ResolveTransient(ServiceCallSite callSite)
{
    // 항상 새 인스턴스 생성
    var instance = CreateInstance(callSite);

    // Dispose 가능하면 현재 스코프에서 추적
    if (instance is IDisposable disposable)
    {
        _currentScope.Disposables.Add(disposable);
    }

    return instance;
}
```

---

## 생성자 선택 알고리즘

### 규칙

```csharp
public class MyService
{
    // 1. 가장 많은 파라미터를 가진 생성자 선택
    public MyService() { }  // 0개
    public MyService(ILogger logger) { }  // 1개
    public MyService(ILogger logger, ICache cache) { }  // 2개 ✅ 선택

    // 2. 모든 파라미터가 해결 가능해야 함
    // 만약 ICache가 등록되지 않았다면:
    public MyService(ILogger logger) { }  // 1개 ✅ 선택
}
```

### 생성자 선택 로직

```csharp
// 단순화된 생성자 선택 로직
ConstructorInfo? SelectConstructor(Type implementationType)
{
    var constructors = implementationType.GetConstructors()
        .OrderByDescending(c => c.GetParameters().Length)
        .ToList();

    foreach (var ctor in constructors)
    {
        var parameters = ctor.GetParameters();
        var allResolvable = parameters.All(p =>
            _callSiteFactory.CanResolve(p.ParameterType));

        if (allResolvable)
            return ctor;
    }

    throw new InvalidOperationException(
        $"No suitable constructor found for {implementationType}");
}
```

---

## ServiceScope 구현

### 스코프 생성

```csharp
public class ServiceProviderEngineScope : IServiceScope
{
    private readonly Dictionary<ServiceCallSite, object> _resolvedServices = new();
    private readonly List<IDisposable> _disposables = new();
    private bool _disposed;

    public IServiceProvider ServiceProvider => this;

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;

        // 역순으로 Dispose (나중에 생성된 것 먼저)
        for (int i = _disposables.Count - 1; i >= 0; i--)
        {
            _disposables[i].Dispose();
        }

        _disposables.Clear();
        _resolvedServices.Clear();
    }
}
```

### HTTP 요청 스코프

```csharp
// ASP.NET Core는 각 HTTP 요청마다 스코프 생성
public class ServiceProvidersFeature : IServiceProvidersFeature
{
    public IServiceProvider RequestServices { get; set; }
}

// 내부적으로 미들웨어에서
public async Task InvokeAsync(HttpContext context)
{
    using var scope = _serviceProvider.CreateScope();
    context.RequestServices = scope.ServiceProvider;

    await _next(context);

    // scope.Dispose()가 자동 호출
    // → 모든 Scoped 서비스 Dispose
}
```

---

## 최적화 기법

### Compiled Expressions

```csharp
// 실제 DI 컨테이너는 리플렉션 대신 컴파일된 표현식 사용
// ConstructorCallSite → IL 또는 Expression으로 컴파일

// 예: new OrderService(repo, logger) 호출을 위한 팩토리 생성
Expression<Func<IServiceProvider, object>> CreateFactory(
    ConstructorInfo ctor,
    ServiceCallSite[] parameterCallSites)
{
    var sp = Expression.Parameter(typeof(IServiceProvider), "sp");

    var args = parameterCallSites.Select(cs =>
        Expression.Call(sp, "GetService", null,
            Expression.Constant(cs.ServiceType)));

    var newExpr = Expression.New(ctor, args);

    return Expression.Lambda<Func<IServiceProvider, object>>(newExpr, sp);
}

// 컴파일된 델리게이트는 캐시됨 → 빠른 인스턴스 생성
```

### CallSite 캐싱

```
┌─────────────────────────────────────────────────────────────────┐
│                    CallSite 캐시                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ServiceType → CallSite 매핑 (Dictionary)                       │
│                                                                 │
│  IOrderService → ConstructorCallSite (OrderService)             │
│  ILogger<T>    → FactoryCallSite (OpenGeneric)                  │
│  DbContext     → ConstructorCallSite (AppDbContext)             │
│                                                                 │
│  한 번 계산되면 앱 전체에서 재사용                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Open Generic 지원

### 등록

```csharp
// Open Generic 등록
services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// 사용
public class OrderService(IRepository<Order> orderRepo)
{
    // IRepository<Order> → Repository<Order>로 자동 해결
}
```

### 내부 동작

```csharp
// Open Generic CallSite 생성
ServiceCallSite CreateOpenGenericCallSite(Type closedType)
{
    // closedType = IRepository<Order>
    var genericDefinition = closedType.GetGenericTypeDefinition();
    // genericDefinition = IRepository<>

    var descriptor = FindOpenGenericDescriptor(genericDefinition);
    // descriptor = IRepository<> → Repository<>

    var closedImplType = descriptor.ImplementationType!
        .MakeGenericType(closedType.GetGenericArguments());
    // closedImplType = Repository<Order>

    return CreateConstructorCallSite(closedImplType);
}
```

---

## ValidateScopes 구현

### Scope 검증 로직

```csharp
// ValidateScopes = true일 때
void ValidateScopeLifetime(ServiceCallSite callSite, ServiceCallSite dependency)
{
    // Singleton이 Scoped를 주입받으려 하면 예외
    if (callSite.Lifetime == ServiceLifetime.Singleton &&
        dependency.Lifetime == ServiceLifetime.Scoped)
    {
        throw new InvalidOperationException(
            $"Cannot consume scoped service '{dependency.ServiceType}' " +
            $"from singleton '{callSite.ServiceType}'.");
    }
}
```

### ValidateOnBuild

```csharp
// ValidateOnBuild = true일 때
void ValidateOnBuild()
{
    foreach (var descriptor in _descriptors)
    {
        try
        {
            // 모든 서비스의 CallSite 생성 시도
            _callSiteFactory.GetCallSite(descriptor.ServiceType);
        }
        catch (Exception ex)
        {
            throw new AggregateException(
                $"Service '{descriptor.ServiceType}' cannot be resolved.",
                ex);
        }
    }
}
```

---

## 메모리 관리

### Singleton 캐시

```csharp
// 루트 ServiceProvider에 저장
private readonly ConcurrentDictionary<ServiceCallSite, object> _singletonCache;

// 앱 종료 시까지 유지
// Dispose 시 모든 Singleton Dispose
```

### Scoped 캐시

```csharp
// 각 ServiceScope에 저장
class ServiceProviderEngineScope
{
    // 요청별 캐시
    private Dictionary<ServiceCallSite, object> _resolvedServices;

    // Scope Dispose 시 해제
}
```

### Transient 추적

```csharp
// Transient도 Dispose를 위해 추적됨
// 현재 Scope에 등록됨
if (instance is IDisposable disposable)
{
    _currentScope.Disposables.Add(disposable);
}

// 주의: 루트 ServiceProvider에서 Transient 해결하면
// 앱 종료까지 Dispose 안됨!
```

---

## 성능 특성

### 벤치마크 비교

| 작업 | 시간 (상대) |
|------|------------|
| Singleton 해결 | 1x (가장 빠름) |
| Scoped 해결 | 1.2x |
| Transient 해결 | 2x |
| 생성자 1개 파라미터 | 1x |
| 생성자 5개 파라미터 | 3x |
| 생성자 10개 파라미터 | 5x |

### 최적화 팁

```csharp
// ✅ 서비스 해결은 시작 시 한 번
public class MyService
{
    private readonly IOrderService _orderService;

    public MyService(IOrderService orderService)
    {
        _orderService = orderService;  // 생성 시 주입
    }
}

// ❌ 메서드마다 해결하지 않기
public class BadService
{
    private readonly IServiceProvider _sp;

    public void DoWork()
    {
        var service = _sp.GetService<IOrderService>();  // 매번 해결
    }
}
```

---

## 면접 예상 질문

### Q1: DI 컨테이너가 생성자를 선택하는 기준은?

**A:** 모든 파라미터가 DI 컨테이너에서 해결 가능한 생성자 중 가장 많은 파라미터를 가진 생성자를 선택합니다. [ActivatorUtilitiesConstructor] 속성으로 명시적 지정도 가능합니다.

### Q2: Scoped 서비스는 어떻게 요청별로 관리되나요?

**A:** ASP.NET Core는 각 HTTP 요청마다 `IServiceScope`를 생성합니다. 이 스코프 내에서 해결된 Scoped 서비스는 Dictionary에 캐시되어 같은 요청 내에서 재사용됩니다. 요청 종료 시 스코프가 Dispose되면서 캐시된 서비스도 Dispose됩니다.

### Q3: ValidateOnBuild의 동작 원리는?

**A:** 앱 시작 시 모든 등록된 서비스에 대해 CallSite를 생성해봅니다. 생성자 파라미터 중 해결 불가능한 서비스가 있으면 즉시 예외가 발생합니다. 개발 중 설정 오류를 조기에 발견할 수 있습니다.

---

## 참고 자료

- [Microsoft.Extensions.DependencyInjection 소스코드](https://github.com/dotnet/runtime/tree/main/src/libraries/Microsoft.Extensions.DependencyInjection)
- [DI 컨테이너 내부 구현](https://andrewlock.net/exploring-the-code-behind-ihostedservice/)

---

## 다음 문서

→ [advanced-patterns.md](./advanced-patterns.md) - 고급 DI 패턴
