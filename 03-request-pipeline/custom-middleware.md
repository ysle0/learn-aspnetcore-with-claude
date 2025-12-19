# 커스텀 미들웨어 작성

## 개요

이 문서에서는 실제 프로젝트에서 자주 사용되는 커스텀 미들웨어 패턴들을 다룹니다. 요청 로깅, 성능 측정, 예외 처리, 인증/인가 등 다양한 실용적인 예제를 제공합니다.

---

## 미들웨어 작성 패턴

### 1. 인라인 미들웨어

```csharp
// 가장 간단한 형태 - 빠른 프로토타이핑
app.Use(async (context, next) =>
{
    Console.WriteLine($"Request: {context.Request.Path}");
    await next(context);
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});
```

### 2. 컨벤션 기반 클래스

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation("Request: {Path}", context.Request.Path);
        await _next(context);
        _logger.LogInformation("Response: {StatusCode}", context.Response.StatusCode);
    }
}

// 확장 메서드
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// 사용
app.UseRequestLogging();
```

### 3. IMiddleware 인터페이스

```csharp
public class ScopedMiddleware : IMiddleware
{
    private readonly IScopedService _service;

    public ScopedMiddleware(IScopedService service)
    {
        _service = service;  // 안전하게 Scoped 서비스 주입
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        _service.Process(context);
        await next(context);
    }
}

// 서비스 등록 필수!
builder.Services.AddScoped<ScopedMiddleware>();
app.UseMiddleware<ScopedMiddleware>();
```

---

## 실용적인 미들웨어 예제

### 요청/응답 타이밍 측정

```csharp
public class TimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<TimingMiddleware> _logger;

    public TimingMiddleware(RequestDelegate next, ILogger<TimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        // 응답 시작 시 타이밍 기록
        context.Response.OnStarting(() =>
        {
            stopwatch.Stop();
            context.Response.Headers.Append(
                "X-Response-Time", $"{stopwatch.ElapsedMilliseconds}ms");
            return Task.CompletedTask;
        });

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation(
                "{Method} {Path} completed in {Duration}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}
```

### 요청 본문 로깅

```csharp
public class RequestBodyLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestBodyLoggingMiddleware> _logger;

    public RequestBodyLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestBodyLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 요청 본문 버퍼링 활성화
        context.Request.EnableBuffering();

        // 본문 읽기
        var body = await ReadRequestBodyAsync(context.Request);

        if (!string.IsNullOrEmpty(body))
        {
            _logger.LogInformation(
                "Request body for {Path}: {Body}",
                context.Request.Path,
                SanitizeBody(body));
        }

        // 스트림 위치 리셋
        context.Request.Body.Position = 0;

        await _next(context);
    }

    private static async Task<string> ReadRequestBodyAsync(HttpRequest request)
    {
        using var reader = new StreamReader(
            request.Body,
            Encoding.UTF8,
            detectEncodingFromByteOrderMarks: false,
            leaveOpen: true);

        var body = await reader.ReadToEndAsync();
        request.Body.Position = 0;
        return body;
    }

    private static string SanitizeBody(string body)
    {
        // 민감 정보 마스킹
        var sanitized = Regex.Replace(
            body,
            @"""password""\s*:\s*""[^""]*""",
            @"""password"":""***""",
            RegexOptions.IgnoreCase);

        return sanitized.Length > 1000
            ? sanitized[..1000] + "...(truncated)"
            : sanitized;
    }
}
```

### 응답 본문 로깅

```csharp
public class ResponseBodyLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ResponseBodyLoggingMiddleware> _logger;

    public ResponseBodyLoggingMiddleware(
        RequestDelegate next,
        ILogger<ResponseBodyLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var originalBody = context.Response.Body;

        using var newBody = new MemoryStream();
        context.Response.Body = newBody;

        await _next(context);

        newBody.Position = 0;
        var responseBody = await new StreamReader(newBody).ReadToEndAsync();

        if (!string.IsNullOrEmpty(responseBody))
        {
            _logger.LogInformation(
                "Response body for {Path}: {Body}",
                context.Request.Path,
                responseBody.Length > 1000
                    ? responseBody[..1000] + "..."
                    : responseBody);
        }

        newBody.Position = 0;
        await newBody.CopyToAsync(originalBody);
    }
}
```

### Correlation ID 미들웨어

```csharp
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 요청 헤더에서 읽거나 새로 생성
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // 컨텍스트에 저장
        context.Items["CorrelationId"] = correlationId;
        context.TraceIdentifier = correlationId;

        // 응답 헤더에 추가
        context.Response.OnStarting(() =>
        {
            context.Response.Headers.TryAdd(CorrelationIdHeader, correlationId);
            return Task.CompletedTask;
        });

        // 로그 스코프에 추가 (구조화된 로깅)
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}
```

### API Key 인증 미들웨어

```csharp
public class ApiKeyMiddleware
{
    private const string ApiKeyHeaderName = "X-API-Key";
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;

    public ApiKeyMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 헬스 체크는 건너뛰기
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            await _next(context);
            return;
        }

        if (!context.Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "API Key required",
                Message = $"Please provide {ApiKeyHeaderName} header"
            });
            return;
        }

        var validApiKeys = _configuration.GetSection("ApiKeys").Get<string[]>() ?? [];

        if (!validApiKeys.Contains(apiKey.ToString()))
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "Invalid API Key"
            });
            return;
        }

        await _next(context);
    }
}
```

### 요청 제한 미들웨어 (간단한 구현)

```csharp
public class SimpleRateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    private readonly int _requestLimit;
    private readonly TimeSpan _window;

    public SimpleRateLimitMiddleware(
        RequestDelegate next,
        IMemoryCache cache,
        int requestLimit = 100,
        int windowSeconds = 60)
    {
        _next = next;
        _cache = cache;
        _requestLimit = requestLimit;
        _window = TimeSpan.FromSeconds(windowSeconds);
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var clientIp = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var cacheKey = $"RateLimit_{clientIp}";

        var requestCount = _cache.GetOrCreate(cacheKey, entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _window;
            return 0;
        });

        if (requestCount >= _requestLimit)
        {
            context.Response.StatusCode = 429;
            context.Response.Headers.Append("Retry-After", _window.TotalSeconds.ToString());
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "Too Many Requests",
                RetryAfter = _window.TotalSeconds
            });
            return;
        }

        _cache.Set(cacheKey, requestCount + 1, _window);

        await _next(context);
    }
}
```

### 유지보수 모드 미들웨어

```csharp
public class MaintenanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IOptionsMonitor<MaintenanceOptions> _options;

    public MaintenanceMiddleware(
        RequestDelegate next,
        IOptionsMonitor<MaintenanceOptions> options)
    {
        _next = next;
        _options = options;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (_options.CurrentValue.IsEnabled)
        {
            // 특정 IP는 허용
            var clientIp = context.Connection.RemoteIpAddress?.ToString();
            if (_options.CurrentValue.AllowedIPs?.Contains(clientIp) == true)
            {
                await _next(context);
                return;
            }

            // 헬스 체크는 허용
            if (context.Request.Path.StartsWithSegments("/health"))
            {
                await _next(context);
                return;
            }

            context.Response.StatusCode = 503;
            context.Response.Headers.Append("Retry-After", "3600");
            await context.Response.WriteAsJsonAsync(new
            {
                Error = "Service Unavailable",
                Message = _options.CurrentValue.Message ?? "The service is under maintenance."
            });
            return;
        }

        await _next(context);
    }
}

public class MaintenanceOptions
{
    public bool IsEnabled { get; set; }
    public string? Message { get; set; }
    public string[]? AllowedIPs { get; set; }
}
```

---

## 조건부 미들웨어

### 경로 기반 조건

```csharp
public static class ConditionalMiddlewareExtensions
{
    public static IApplicationBuilder UseMiddlewareWhen<TMiddleware>(
        this IApplicationBuilder app,
        Func<HttpContext, bool> predicate)
        where TMiddleware : class
    {
        return app.UseWhen(predicate, appBuilder =>
        {
            appBuilder.UseMiddleware<TMiddleware>();
        });
    }
}

// 사용
app.UseMiddlewareWhen<ApiLoggingMiddleware>(
    context => context.Request.Path.StartsWithSegments("/api"));

app.UseMiddlewareWhen<AdminMiddleware>(
    context => context.Request.Path.StartsWithSegments("/admin"));
```

### 환경 기반 조건

```csharp
public static class EnvironmentMiddlewareExtensions
{
    public static IApplicationBuilder UseMiddlewareInDevelopment<TMiddleware>(
        this IApplicationBuilder app)
        where TMiddleware : class
    {
        var env = app.ApplicationServices.GetRequiredService<IWebHostEnvironment>();

        if (env.IsDevelopment())
        {
            app.UseMiddleware<TMiddleware>();
        }

        return app;
    }
}

// 사용
app.UseMiddlewareInDevelopment<RequestBodyLoggingMiddleware>();
```

---

## 미들웨어 옵션 패턴

### 옵션 클래스 정의

```csharp
public class LoggingMiddlewareOptions
{
    public bool LogRequestBody { get; set; }
    public bool LogResponseBody { get; set; }
    public bool LogHeaders { get; set; }
    public int MaxBodyLength { get; set; } = 4096;
    public string[] SensitiveHeaders { get; set; } = ["Authorization", "Cookie"];
    public Func<HttpContext, bool>? ShouldLog { get; set; }
}
```

### 옵션을 사용하는 미들웨어

```csharp
public class ConfigurableLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ConfigurableLoggingMiddleware> _logger;
    private readonly LoggingMiddlewareOptions _options;

    public ConfigurableLoggingMiddleware(
        RequestDelegate next,
        ILogger<ConfigurableLoggingMiddleware> logger,
        IOptions<LoggingMiddlewareOptions> options)
    {
        _next = next;
        _logger = logger;
        _options = options.Value;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (_options.ShouldLog?.Invoke(context) == false)
        {
            await _next(context);
            return;
        }

        if (_options.LogHeaders)
        {
            LogHeaders(context);
        }

        if (_options.LogRequestBody)
        {
            await LogRequestBodyAsync(context);
        }

        await _next(context);

        if (_options.LogResponseBody)
        {
            // 응답 로깅 로직...
        }
    }

    private void LogHeaders(HttpContext context)
    {
        var headers = context.Request.Headers
            .Where(h => !_options.SensitiveHeaders.Contains(h.Key))
            .ToDictionary(h => h.Key, h => h.Value.ToString());

        _logger.LogInformation("Request headers: {@Headers}", headers);
    }

    private async Task LogRequestBodyAsync(HttpContext context)
    {
        context.Request.EnableBuffering();
        using var reader = new StreamReader(context.Request.Body, leaveOpen: true);
        var body = await reader.ReadToEndAsync();
        context.Request.Body.Position = 0;

        if (body.Length > _options.MaxBodyLength)
        {
            body = body[.._options.MaxBodyLength] + "...(truncated)";
        }

        _logger.LogInformation("Request body: {Body}", body);
    }
}
```

### 확장 메서드

```csharp
public static class LoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseConfigurableLogging(
        this IApplicationBuilder builder,
        Action<LoggingMiddlewareOptions>? configure = null)
    {
        var options = new LoggingMiddlewareOptions();
        configure?.Invoke(options);

        builder.ApplicationServices
            .GetRequiredService<IOptions<LoggingMiddlewareOptions>>()
            .Value = options;

        return builder.UseMiddleware<ConfigurableLoggingMiddleware>();
    }
}

// 또는 서비스 등록 방식
builder.Services.Configure<LoggingMiddlewareOptions>(options =>
{
    options.LogRequestBody = true;
    options.LogHeaders = true;
    options.ShouldLog = ctx => ctx.Request.Path.StartsWithSegments("/api");
});

app.UseMiddleware<ConfigurableLoggingMiddleware>();
```

---

## 테스트 가능한 미들웨어

### 단위 테스트

```csharp
public class TimingMiddlewareTests
{
    [Fact]
    public async Task Adds_ResponseTime_Header()
    {
        // Arrange
        var context = new DefaultHttpContext();
        var middleware = new TimingMiddleware(
            next: _ => Task.CompletedTask,
            logger: NullLogger<TimingMiddleware>.Instance);

        // Act
        await middleware.InvokeAsync(context);

        // Assert
        Assert.True(context.Response.Headers.ContainsKey("X-Response-Time"));
    }

    [Fact]
    public async Task Calls_Next_Middleware()
    {
        // Arrange
        var nextCalled = false;
        var context = new DefaultHttpContext();
        var middleware = new TimingMiddleware(
            next: _ =>
            {
                nextCalled = true;
                return Task.CompletedTask;
            },
            logger: NullLogger<TimingMiddleware>.Instance);

        // Act
        await middleware.InvokeAsync(context);

        // Assert
        Assert.True(nextCalled);
    }
}
```

### 통합 테스트

```csharp
public class MiddlewareIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public MiddlewareIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task CorrelationId_Is_Added_To_Response()
    {
        // Arrange
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync("/api/test");

        // Assert
        Assert.True(response.Headers.Contains("X-Correlation-ID"));
    }

    [Fact]
    public async Task ApiKey_Middleware_Returns_401_Without_Key()
    {
        // Arrange
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync("/api/protected");

        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
}
```

---

## 면접 예상 질문

### Q1: 미들웨어에서 Scoped 서비스를 어떻게 사용하나요?

**A:** 두 가지 방법:
1. **컨벤션 기반**: `InvokeAsync` 메서드의 파라미터로 주입
2. **IMiddleware**: 생성자에서 직접 주입 (서비스 등록 필수)

생성자에 Scoped 서비스를 주입하면 미들웨어가 Singleton이므로 문제가 발생합니다.

### Q2: 미들웨어와 ActionFilter의 차이점은?

**A:**
- **미들웨어**: 모든 요청에 적용, 파이프라인 전체에서 동작, 라우팅 전에도 실행 가능
- **ActionFilter**: 컨트롤러/액션에만 적용, MVC 파이프라인 내에서만 동작, 모델 바인딩 후 실행

### Q3: 미들웨어 테스트 방법은?

**A:**
- **단위 테스트**: `DefaultHttpContext` 사용, Mock `RequestDelegate`
- **통합 테스트**: `WebApplicationFactory` 사용, 실제 HTTP 요청

### Q4: 미들웨어에서 옵션 패턴을 사용하는 이유는?

**A:**
- 설정을 외부화하여 재사용성 향상
- `IOptionsMonitor`로 런타임 설정 변경 가능
- DI를 통한 테스트 용이성
- 관심사 분리

---

## 참고 자료

- [커스텀 미들웨어 작성](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/write)
- [미들웨어 팩토리 기반 활성화](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/extensibility)
- [미들웨어 테스트](https://learn.microsoft.com/aspnet/core/test/middleware)

---

## 다음 섹션

→ [04. Dependency Injection](../04-dependency-injection/) - DI 컨테이너 이해하기
