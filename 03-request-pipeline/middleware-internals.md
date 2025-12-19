# 미들웨어 내부 동작

## 개요

이 문서에서는 ASP.NET Core 미들웨어 파이프라인의 내부 구현을 살펴봅니다. `IApplicationBuilder`, `RequestDelegate`, 미들웨어 팩토리 패턴 등 프레임워크 내부 동작을 이해합니다.

---

## ApplicationBuilder 내부 구조

### IApplicationBuilder 인터페이스

```csharp
public interface IApplicationBuilder
{
    IServiceProvider ApplicationServices { get; set; }
    IFeatureCollection ServerFeatures { get; }
    IDictionary<string, object?> Properties { get; }

    IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware);
    IApplicationBuilder New();
    RequestDelegate Build();
}
```

### ApplicationBuilder 핵심 구현

```csharp
// 단순화된 ApplicationBuilder 구현
public class ApplicationBuilder : IApplicationBuilder
{
    private readonly List<Func<RequestDelegate, RequestDelegate>> _middlewares = new();

    public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware)
    {
        _middlewares.Add(middleware);
        return this;
    }

    public RequestDelegate Build()
    {
        // 최종 미들웨어 (404 반환)
        RequestDelegate app = context =>
        {
            context.Response.StatusCode = 404;
            return Task.CompletedTask;
        };

        // 역순으로 미들웨어를 래핑
        for (int i = _middlewares.Count - 1; i >= 0; i--)
        {
            app = _middlewares[i](app);
        }

        return app;
    }
}
```

### 미들웨어 체인 구성 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                    미들웨어 체인 구성                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  등록 순서:                                                     │
│    app.Use(M1)                                                 │
│    app.Use(M2)                                                 │
│    app.Use(M3)                                                 │
│                                                                 │
│  Build() 과정 (역순 래핑):                                      │
│                                                                 │
│    1. app = 404 Handler                                        │
│    2. app = M3(app)   → M3이 404를 감싸기                       │
│    3. app = M2(app)   → M2가 M3(404)를 감싸기                   │
│    4. app = M1(app)   → M1이 M2(M3(404))를 감싸기               │
│                                                                 │
│  최종 결과:                                                     │
│    M1 → M2 → M3 → 404                                          │
│                                                                 │
│  실행 순서:                                                     │
│    Request → M1 전처리 → M2 전처리 → M3 전처리 →               │
│    404 → M3 후처리 → M2 후처리 → M1 후처리 → Response          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## RequestDelegate 체인

### Use 확장 메서드 분석

```csharp
public static class UseExtensions
{
    // 가장 기본적인 Use
    public static IApplicationBuilder Use(
        this IApplicationBuilder app,
        Func<HttpContext, Func<Task>, Task> middleware)
    {
        return app.Use(next =>
        {
            return context =>
            {
                Func<Task> simpleNext = () => next(context);
                return middleware(context, simpleNext);
            };
        });
    }

    // HttpContext, RequestDelegate를 받는 Use
    public static IApplicationBuilder Use(
        this IApplicationBuilder app,
        Func<HttpContext, RequestDelegate, Task> middleware)
    {
        return app.Use(next =>
        {
            return context => middleware(context, next);
        });
    }
}
```

### Run 확장 메서드 분석

```csharp
public static class RunExtensions
{
    public static void Run(
        this IApplicationBuilder app,
        RequestDelegate handler)
    {
        // Run은 next를 무시하고 handler만 실행
        app.Use(_ => handler);
    }
}
```

### Map 확장 메서드 분석

```csharp
public static class MapExtensions
{
    public static IApplicationBuilder Map(
        this IApplicationBuilder app,
        PathString pathMatch,
        Action<IApplicationBuilder> configuration)
    {
        // 새로운 분기 빌더 생성
        var branchBuilder = app.New();
        configuration(branchBuilder);
        var branch = branchBuilder.Build();

        return app.Use(async (context, next) =>
        {
            var path = context.Request.Path;

            if (path.StartsWithSegments(pathMatch, out var remaining))
            {
                // 경로 매치 시 분기 파이프라인 실행
                var originalPath = context.Request.Path;
                var originalPathBase = context.Request.PathBase;

                context.Request.PathBase = originalPathBase.Add(pathMatch);
                context.Request.Path = remaining;

                try
                {
                    await branch(context);
                }
                finally
                {
                    context.Request.PathBase = originalPathBase;
                    context.Request.Path = originalPath;
                }
            }
            else
            {
                // 매치되지 않으면 다음 미들웨어로
                await next(context);
            }
        });
    }
}
```

---

## 미들웨어 클래스 컨벤션

### 컨벤션 기반 미들웨어

```csharp
// 컨벤션: 생성자에 RequestDelegate, InvokeAsync 또는 Invoke 메서드
public class ConventionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ConventionMiddleware> _logger;

    // 생성자: RequestDelegate는 필수, 다른 종속성은 DI
    public ConventionMiddleware(
        RequestDelegate next,
        ILogger<ConventionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // InvokeAsync 또는 Invoke 메서드
    public async Task InvokeAsync(
        HttpContext context,
        IMyService myService)  // 요청 스코프 서비스는 메서드 주입
    {
        _logger.LogInformation("Before");

        await _next(context);

        _logger.LogInformation("After");
    }
}

// 사용
app.UseMiddleware<ConventionMiddleware>();
```

### UseMiddleware 내부 동작

```csharp
// 단순화된 UseMiddleware 구현
public static IApplicationBuilder UseMiddleware<TMiddleware>(
    this IApplicationBuilder app,
    params object[] args)
{
    return app.Use(next =>
    {
        // 미들웨어 타입 분석
        var methods = typeof(TMiddleware).GetMethods(BindingFlags.Instance | BindingFlags.Public);
        var invokeMethod = methods.FirstOrDefault(m =>
            m.Name == "InvokeAsync" || m.Name == "Invoke");

        if (invokeMethod == null)
            throw new InvalidOperationException("Invoke method not found");

        // 팩토리 함수 생성
        var ctorArgs = new object[] { next }.Concat(args).ToArray();

        return async context =>
        {
            // 미들웨어 인스턴스 생성
            var middleware = ActivatorUtilities.CreateInstance(
                app.ApplicationServices,
                typeof(TMiddleware),
                ctorArgs);

            // 메서드 파라미터 해결
            var parameters = invokeMethod.GetParameters();
            var paramValues = new object[parameters.Length];
            paramValues[0] = context;

            for (int i = 1; i < parameters.Length; i++)
            {
                var paramType = parameters[i].ParameterType;
                var requestServices = context.RequestServices;
                paramValues[i] = requestServices.GetRequiredService(paramType);
            }

            // 메서드 호출
            var task = (Task)invokeMethod.Invoke(middleware, paramValues)!;
            await task;
        };
    });
}
```

---

## IMiddleware 인터페이스

### 팩토리 기반 미들웨어

```csharp
public interface IMiddleware
{
    Task InvokeAsync(HttpContext context, RequestDelegate next);
}

// 구현
public class FactoryMiddleware : IMiddleware
{
    private readonly ILogger<FactoryMiddleware> _logger;

    public FactoryMiddleware(ILogger<FactoryMiddleware> logger)
    {
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        _logger.LogInformation("Factory middleware executing");
        await next(context);
    }
}

// 등록 (서비스 등록 필수!)
builder.Services.AddTransient<FactoryMiddleware>();

// 사용
app.UseMiddleware<FactoryMiddleware>();
```

### 컨벤션 vs 팩토리 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              컨벤션 vs 팩토리 미들웨어                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  컨벤션 기반 미들웨어:                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ • 인스턴스 생명주기: Singleton (앱 시작 시 1번 생성)       │ │
│  │ • 생성자 DI: 앱 시작 시 해결                              │ │
│  │ • 메서드 DI: 요청마다 해결 (Scoped 서비스 OK)              │ │
│  │ • 서비스 등록: 불필요                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  팩토리 기반 미들웨어 (IMiddleware):                            │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ • 인스턴스 생명주기: 등록된 대로 (Transient/Scoped)        │ │
│  │ • 생성자 DI: 요청마다 해결 (등록에 따라)                   │ │
│  │ • 서비스 등록: 필수                                       │ │
│  │ • 장점: 상태 격리, Scoped 서비스를 생성자에서 주입 가능    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 언제 IMiddleware를 사용해야 하나?

```csharp
// ❌ 컨벤션 기반에서 Scoped 서비스 생성자 주입 - 문제 발생!
public class BadMiddleware
{
    private readonly IDbContext _db;  // Scoped 서비스

    public BadMiddleware(RequestDelegate next, IDbContext db)
    {
        _db = db;  // Singleton 수명으로 캡처됨 - 위험!
    }
}

// ✅ 컨벤션 기반에서 올바른 방법
public class CorrectConventionMiddleware
{
    private readonly RequestDelegate _next;

    public CorrectConventionMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context, IDbContext db)
    {
        // 메서드 파라미터로 주입 - 요청마다 새로운 인스턴스
        await _next(context);
    }
}

// ✅ IMiddleware 사용 - 생성자 주입 안전
public class CorrectFactoryMiddleware : IMiddleware
{
    private readonly IDbContext _db;  // 안전 - 요청마다 새 인스턴스

    public CorrectFactoryMiddleware(IDbContext db)
    {
        _db = db;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        await _next(context);
    }
}
```

---

## 미들웨어 실행 컨텍스트

### 스레드와 동기화 컨텍스트

```csharp
public class ThreadInfoMiddleware
{
    private readonly RequestDelegate _next;

    public ThreadInfoMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var threadBefore = Thread.CurrentThread.ManagedThreadId;
        Console.WriteLine($"Before: Thread {threadBefore}");

        await _next(context);

        var threadAfter = Thread.CurrentThread.ManagedThreadId;
        Console.WriteLine($"After: Thread {threadAfter}");

        // ASP.NET Core는 동기화 컨텍스트가 없으므로
        // await 이후 다른 스레드에서 실행될 수 있음
    }
}
```

### HttpContext 스레드 안전성

```csharp
public class ThreadSafetyMiddleware
{
    private readonly RequestDelegate _next;

    public ThreadSafetyMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // ❌ 위험: HttpContext를 다른 스레드에서 사용
        _ = Task.Run(async () =>
        {
            // HttpContext는 요청 스레드에서만 사용해야 함
            await context.Response.WriteAsync("Danger!");
        });

        // ✅ 안전: 필요한 데이터를 미리 추출
        var path = context.Request.Path.ToString();
        _ = Task.Run(() =>
        {
            Console.WriteLine($"Processing: {path}");
        });

        await _next(context);
    }
}
```

---

## 분기 파이프라인 내부

### 분기 구현 상세

```csharp
public static class BranchingMiddlewareExtensions
{
    public static IApplicationBuilder UseBranching(
        this IApplicationBuilder app,
        Func<HttpContext, bool> predicate,
        Action<IApplicationBuilder> configure)
    {
        // 분기용 새 빌더 생성
        var branchBuilder = app.New();
        configure(branchBuilder);
        var branch = branchBuilder.Build();

        return app.Use(async (context, next) =>
        {
            if (predicate(context))
            {
                await branch(context);
            }
            else
            {
                await next(context);
            }
        });
    }
}

// 사용 예
app.UseBranching(
    ctx => ctx.Request.Path.StartsWithSegments("/api"),
    apiApp =>
    {
        apiApp.UseAuthentication();
        apiApp.UseAuthorization();
    });
```

### 복잡한 분기 패턴

```csharp
public static class AdvancedBranchingExtensions
{
    public static IApplicationBuilder UseRouteSwitch(
        this IApplicationBuilder app,
        Dictionary<string, Action<IApplicationBuilder>> routes)
    {
        var branches = routes.ToDictionary(
            kv => kv.Key,
            kv =>
            {
                var branchBuilder = app.New();
                kv.Value(branchBuilder);
                return branchBuilder.Build();
            });

        return app.Use(async (context, next) =>
        {
            foreach (var (path, branch) in branches)
            {
                if (context.Request.Path.StartsWithSegments(path))
                {
                    await branch(context);
                    return;
                }
            }

            await next(context);
        });
    }
}

// 사용
app.UseRouteSwitch(new Dictionary<string, Action<IApplicationBuilder>>
{
    ["/api/v1"] = v1 => { v1.UseMiddleware<V1Middleware>(); },
    ["/api/v2"] = v2 => { v2.UseMiddleware<V2Middleware>(); },
    ["/admin"] = admin => { admin.UseMiddleware<AdminMiddleware>(); }
});
```

---

## 성능 최적화

### 미들웨어 성능 팁

```csharp
// ✅ 비동기 메서드 최적화 - 동기 작업에는 ValueTask 사용
public async Task InvokeAsync(HttpContext context)
{
    // 빠른 경로: 조기 반환
    if (context.Request.Path == "/health")
    {
        context.Response.StatusCode = 200;
        return;  // await 없이 동기 완료
    }

    await _next(context);
}

// ✅ 문자열 할당 최소화
private static readonly byte[] _healthResponse = "OK"u8.ToArray();

public async Task InvokeAsync(HttpContext context)
{
    if (context.Request.Path == "/health")
    {
        context.Response.StatusCode = 200;
        context.Response.ContentType = "text/plain";
        await context.Response.Body.WriteAsync(_healthResponse);
        return;
    }

    await _next(context);
}

// ✅ 조건 체크 최적화
public class OptimizedMiddleware
{
    private readonly RequestDelegate _next;
    private readonly bool _isEnabled;

    public OptimizedMiddleware(RequestDelegate next, IOptions<MyOptions> options)
    {
        _next = next;
        _isEnabled = options.Value.IsEnabled;
    }

    public Task InvokeAsync(HttpContext context)
    {
        // 비활성화 시 바로 다음으로 (async 오버헤드 없음)
        if (!_isEnabled)
        {
            return _next(context);
        }

        return InvokeAsyncCore(context);
    }

    private async Task InvokeAsyncCore(HttpContext context)
    {
        // 실제 로직
        await _next(context);
    }
}
```

### 미들웨어 벤치마크

```csharp
// BenchmarkDotNet을 사용한 미들웨어 성능 측정
[MemoryDiagnoser]
public class MiddlewareBenchmarks
{
    private RequestDelegate _inlineMiddleware = null!;
    private RequestDelegate _classMiddleware = null!;

    [GlobalSetup]
    public void Setup()
    {
        var builder = new ApplicationBuilder(new ServiceCollection().BuildServiceProvider());

        // 인라인 미들웨어
        builder.Use(async (context, next) =>
        {
            context.Items["test"] = true;
            await next(context);
        });
        builder.Run(context => Task.CompletedTask);
        _inlineMiddleware = builder.Build();

        // 클래스 미들웨어
        builder = new ApplicationBuilder(new ServiceCollection().BuildServiceProvider());
        builder.UseMiddleware<SimpleMiddleware>();
        builder.Run(context => Task.CompletedTask);
        _classMiddleware = builder.Build();
    }

    [Benchmark]
    public async Task InlineMiddleware()
    {
        var context = new DefaultHttpContext();
        await _inlineMiddleware(context);
    }

    [Benchmark]
    public async Task ClassMiddleware()
    {
        var context = new DefaultHttpContext();
        await _classMiddleware(context);
    }
}
```

---

## 면접 예상 질문

### Q1: ApplicationBuilder.Build()는 어떻게 동작하나요?

**A:** 등록된 미들웨어를 **역순으로 순회**하면서 각 미들웨어 함수를 호출합니다. 마지막에 등록된 미들웨어부터 시작하여 첫 번째 미들웨어가 최종 RequestDelegate가 됩니다. 기본 터미널은 404를 반환합니다.

### Q2: 컨벤션 기반과 IMiddleware 기반의 차이점은?

**A:**
- **컨벤션 기반**: Singleton 수명, 생성자 DI는 앱 시작 시, 메서드 DI는 요청마다
- **IMiddleware**: 등록된 수명(Transient/Scoped), 서비스 등록 필수, Scoped 서비스를 생성자에서 안전하게 주입 가능

### Q3: 미들웨어에서 Scoped 서비스를 어떻게 사용하나요?

**A:** 두 가지 방법:
1. **컨벤션 기반**: InvokeAsync 메서드의 파라미터로 주입
2. **IMiddleware**: 생성자에서 직접 주입 (Transient/Scoped로 등록)

생성자에서 캡처하면 Singleton 수명이 되어 Scoped 서비스가 올바르게 동작하지 않습니다.

### Q4: Map과 UseWhen의 차이점은?

**A:**
- **Map**: 경로가 매치되면 **별도의 파이프라인**으로 분기. 원래 파이프라인으로 돌아오지 않음.
- **UseWhen**: 조건이 참이면 추가 미들웨어 실행 후 **원래 파이프라인으로 합류**.

---

## 참고 자료

- [ASP.NET Core 소스 코드 - ApplicationBuilder](https://github.com/dotnet/aspnetcore/blob/main/src/Http/Http/src/Builder/ApplicationBuilder.cs)
- [미들웨어 작성 가이드](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/write)

---

## 다음 문서

→ [built-in-middleware.md](./built-in-middleware.md) - 내장 미들웨어 가이드
