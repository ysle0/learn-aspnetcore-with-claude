# 미들웨어 기초

## 개요

미들웨어는 ASP.NET Core 애플리케이션에서 **HTTP 요청과 응답을 처리하는 소프트웨어 컴포넌트**입니다. 각 미들웨어는 요청을 다음 컴포넌트에 전달하거나, 직접 응답을 반환하여 파이프라인을 종료할 수 있습니다.

---

## 미들웨어의 기본 개념

### 요청 델리게이트 (RequestDelegate)

```csharp
// RequestDelegate의 정의
public delegate Task RequestDelegate(HttpContext context);

// 미들웨어의 기본 형태
public class MyMiddleware
{
    private readonly RequestDelegate _next;

    public MyMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 요청 처리 전 로직

        await _next(context);  // 다음 미들웨어 호출

        // 응답 처리 후 로직
    }
}
```

### 파이프라인 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                    미들웨어 파이프라인                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request ─────────────────────────────────────────────────►     │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Middleware 1                          │   │
│  │   ┌───────────────────────────────────────────────┐     │   │
│  │   │  전처리 (Before next())                       │     │   │
│  │   ├───────────────────────────────────────────────┤     │   │
│  │   │  await next(context)  ──────────────────┐     │     │   │
│  │   ├───────────────────────────────────────────────┤     │   │
│  │   │  후처리 (After next())                  ◄─────┘     │   │
│  │   └───────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│     │                                          ▲               │
│     ▼                                          │               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Middleware 2                          │   │
│  │   (같은 패턴 반복)                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│     │                                          ▲               │
│     ▼                                          │               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Terminal Middleware                   │   │
│  │   (next() 호출 없음 - 응답 생성)                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ◄──────────────────────────────────────────────── Response    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 인라인 미들웨어

### Use - 체인 연결

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 미들웨어 체인
app.Use(async (context, next) =>
{
    Console.WriteLine("1. 요청 수신");
    await next(context);  // 다음 미들웨어로 전달
    Console.WriteLine("6. 응답 전송 완료");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("2. 인증 확인");
    await next(context);
    Console.WriteLine("5. 인증 로그 기록");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("3. 비즈니스 로직 전");
    await next(context);
    Console.WriteLine("4. 비즈니스 로직 후");
});

app.Run(async context =>
{
    await context.Response.WriteAsync("Hello World!");
});

app.Run();

// 출력 순서:
// 1. 요청 수신
// 2. 인증 확인
// 3. 비즈니스 로직 전
// 4. 비즈니스 로직 후
// 5. 인증 로그 기록
// 6. 응답 전송 완료
```

### Run - 파이프라인 종료

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Final Response");
    // next() 없음 - 파이프라인이 여기서 종료
});

// Run 이후의 미들웨어는 실행되지 않음
app.Use(async (context, next) =>
{
    Console.WriteLine("이 코드는 실행되지 않습니다");
    await next(context);
});
```

### Map - 경로 분기

```csharp
var app = builder.Build();

// /api 경로
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        Console.WriteLine("API 미들웨어");
        await next(context);
    });

    apiApp.Map("/users", usersApp =>
    {
        usersApp.Run(async context =>
        {
            await context.Response.WriteAsync("Users API");
        });
    });

    apiApp.Map("/products", productsApp =>
    {
        productsApp.Run(async context =>
        {
            await context.Response.WriteAsync("Products API");
        });
    });
});

// /admin 경로
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin Panel");
    });
});

// 기본 경로
app.Run(async context =>
{
    await context.Response.WriteAsync("Home Page");
});
```

### MapWhen - 조건부 분기

```csharp
// 쿼리 파라미터로 분기
app.MapWhen(
    context => context.Request.Query.ContainsKey("branch"),
    branchApp =>
    {
        branchApp.Run(async context =>
        {
            var branch = context.Request.Query["branch"];
            await context.Response.WriteAsync($"Branch: {branch}");
        });
    });

// HTTP 메서드로 분기
app.MapWhen(
    context => context.Request.Method == "POST",
    postApp =>
    {
        postApp.Run(async context =>
        {
            await context.Response.WriteAsync("POST Request");
        });
    });

// 헤더로 분기
app.MapWhen(
    context => context.Request.Headers.ContainsKey("X-Custom-Header"),
    customApp =>
    {
        customApp.Run(async context =>
        {
            await context.Response.WriteAsync("Custom Header Present");
        });
    });
```

### UseWhen - 조건부 실행 (다시 합류)

```csharp
// MapWhen과 달리 조건 후에도 메인 파이프라인으로 합류
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    appBuilder =>
    {
        appBuilder.Use(async (context, next) =>
        {
            Console.WriteLine("API 요청 로깅");
            await next(context);
        });
    });

// 위 조건과 관계없이 모든 요청이 이 미들웨어를 통과
app.Use(async (context, next) =>
{
    Console.WriteLine("공통 미들웨어");
    await next(context);
});
```

---

## Short-Circuit (파이프라인 단축)

### next() 호출 없이 응답

```csharp
app.Use(async (context, next) =>
{
    // 조건에 따라 파이프라인 단축
    if (context.Request.Path == "/health")
    {
        context.Response.StatusCode = 200;
        await context.Response.WriteAsync("OK");
        return;  // next() 호출하지 않음 - 파이프라인 종료
    }

    await next(context);  // 조건 불충족 시 계속 진행
});
```

### 인증/인가 실패 시 단축

```csharp
app.Use(async (context, next) =>
{
    var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();

    if (string.IsNullOrEmpty(apiKey))
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsJsonAsync(new { Error = "API Key required" });
        return;  // 파이프라인 단축
    }

    if (apiKey != "valid-api-key")
    {
        context.Response.StatusCode = 403;
        await context.Response.WriteAsJsonAsync(new { Error = "Invalid API Key" });
        return;  // 파이프라인 단축
    }

    await next(context);
});
```

### 캐시 히트 시 단축

```csharp
app.Use(async (context, next) =>
{
    var cacheKey = context.Request.Path.ToString();
    var cachedResponse = await _cache.GetAsync(cacheKey);

    if (cachedResponse != null)
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsync(cachedResponse);
        return;  // 캐시에서 응답 - 파이프라인 단축
    }

    // 응답 가로채기 준비
    var originalBody = context.Response.Body;
    using var memoryStream = new MemoryStream();
    context.Response.Body = memoryStream;

    await next(context);  // 실제 처리

    // 응답 캐싱
    memoryStream.Seek(0, SeekOrigin.Begin);
    var responseBody = await new StreamReader(memoryStream).ReadToEndAsync();
    await _cache.SetAsync(cacheKey, responseBody);

    // 원본 스트림에 쓰기
    memoryStream.Seek(0, SeekOrigin.Begin);
    await memoryStream.CopyToAsync(originalBody);
    context.Response.Body = originalBody;
});
```

---

## HttpContext 활용

### 요청 정보 읽기

```csharp
app.Use(async (context, next) =>
{
    // 요청 정보
    var method = context.Request.Method;           // GET, POST 등
    var path = context.Request.Path;               // /api/users
    var queryString = context.Request.QueryString; // ?id=123
    var host = context.Request.Host;               // example.com
    var scheme = context.Request.Scheme;           // https

    // 헤더
    var contentType = context.Request.ContentType;
    var userAgent = context.Request.Headers["User-Agent"];

    // 쿼리 파라미터
    var id = context.Request.Query["id"];

    // 본문 읽기
    context.Request.EnableBuffering();  // 여러 번 읽기 위해
    using var reader = new StreamReader(context.Request.Body);
    var body = await reader.ReadToEndAsync();
    context.Request.Body.Position = 0;  // 다음 미들웨어를 위해 되감기

    await next(context);
});
```

### 응답 조작

```csharp
app.Use(async (context, next) =>
{
    // 응답 시작 전에만 헤더 설정 가능
    context.Response.Headers.Append("X-Custom-Header", "MyValue");

    await next(context);

    // 응답 상태 확인 (변경은 불가)
    if (context.Response.StatusCode == 404)
    {
        Console.WriteLine("Not Found 응답");
    }
});
```

### 응답 가로채기

```csharp
app.Use(async (context, next) =>
{
    var originalBody = context.Response.Body;

    using var newBody = new MemoryStream();
    context.Response.Body = newBody;

    await next(context);

    // 응답 내용 읽기
    newBody.Seek(0, SeekOrigin.Begin);
    var responseText = await new StreamReader(newBody).ReadToEndAsync();

    // 응답 수정
    var modifiedResponse = responseText.Replace("Hello", "Hi");

    // 원본 스트림에 쓰기
    context.Response.Body = originalBody;
    context.Response.ContentLength = null;  // 길이가 변경되었으므로
    await context.Response.WriteAsync(modifiedResponse);
});
```

---

## Items를 통한 데이터 전달

```csharp
app.Use(async (context, next) =>
{
    // 첫 번째 미들웨어에서 데이터 설정
    context.Items["RequestStartTime"] = DateTime.UtcNow;
    context.Items["CorrelationId"] = Guid.NewGuid().ToString();

    await next(context);

    // 응답 후 데이터 사용
    var startTime = (DateTime)context.Items["RequestStartTime"]!;
    var duration = DateTime.UtcNow - startTime;
    Console.WriteLine($"Request took {duration.TotalMilliseconds}ms");
});

app.Use(async (context, next) =>
{
    // 다른 미들웨어에서 데이터 사용
    var correlationId = context.Items["CorrelationId"]?.ToString();
    Console.WriteLine($"Processing request {correlationId}");

    await next(context);
});
```

---

## 타입 안전한 데이터 전달

```csharp
// 키 정의
public static class HttpContextKeys
{
    public static readonly object RequestTimingKey = new();
    public static readonly object UserContextKey = new();
}

// 데이터 모델
public record RequestTiming(DateTime StartTime, string CorrelationId);
public record UserContext(string UserId, string[] Roles);

// 확장 메서드
public static class HttpContextExtensions
{
    public static RequestTiming GetRequestTiming(this HttpContext context)
        => (RequestTiming)context.Items[HttpContextKeys.RequestTimingKey]!;

    public static void SetRequestTiming(this HttpContext context, RequestTiming timing)
        => context.Items[HttpContextKeys.RequestTimingKey] = timing;

    public static UserContext? GetUserContext(this HttpContext context)
        => context.Items[HttpContextKeys.UserContextKey] as UserContext;

    public static void SetUserContext(this HttpContext context, UserContext user)
        => context.Items[HttpContextKeys.UserContextKey] = user;
}

// 사용
app.Use(async (context, next) =>
{
    context.SetRequestTiming(new RequestTiming(DateTime.UtcNow, Guid.NewGuid().ToString()));
    await next(context);
});

app.MapGet("/", (HttpContext context) =>
{
    var timing = context.GetRequestTiming();
    return $"Started at: {timing.StartTime}";
});
```

---

## 면접 예상 질문

### Q1: Use, Run, Map의 차이점은?

**A:**
- **Use**: 다음 미들웨어를 호출할 수 있는 체인 미들웨어. `next(context)`로 전달.
- **Run**: 파이프라인을 종료하는 터미널 미들웨어. `next()` 없음.
- **Map**: 경로에 따라 파이프라인을 분기. 별도의 독립적인 파이프라인 생성.

### Q2: Short-circuit이란?

**A:** `next(context)`를 호출하지 않고 미들웨어에서 직접 응답을 반환하여 파이프라인을 조기에 종료하는 것입니다. 캐시 히트, 인증 실패, 헬스 체크 등에서 사용됩니다.

### Q3: 미들웨어에서 응답 본문을 수정하려면?

**A:** Response.Body를 MemoryStream으로 교체하여 가로챈 후, next() 호출 후 내용을 수정하고 원본 스트림에 기록합니다. 이때 `ContentLength`를 null로 설정해야 합니다.

### Q4: 미들웨어 간 데이터 전달 방법은?

**A:** `HttpContext.Items` 딕셔너리를 사용합니다. 요청 스코프에서만 유효하며, 타입 안전성을 위해 확장 메서드 패턴을 권장합니다.

---

## 참고 자료

- [ASP.NET Core 미들웨어](https://learn.microsoft.com/aspnet/core/fundamentals/middleware)
- [미들웨어 순서 가이드](https://learn.microsoft.com/aspnet/core/fundamentals/middleware#middleware-order)

---

## 다음 문서

→ [middleware-internals.md](./middleware-internals.md) - 미들웨어 내부 동작 이해하기
