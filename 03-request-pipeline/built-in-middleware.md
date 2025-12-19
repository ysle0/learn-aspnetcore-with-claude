# 내장 미들웨어 가이드

## 개요

ASP.NET Core는 다양한 내장 미들웨어를 제공합니다. 이 문서에서는 주요 내장 미들웨어의 역할, 설정 방법, 그리고 권장 순서를 설명합니다.

---

## 미들웨어 권장 순서

```csharp
var app = builder.Build();

// 1. 예외 처리 (가장 먼저!)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

// 2. HTTPS 리다이렉션
app.UseHttpsRedirection();

// 3. 정적 파일 (빠른 반환)
app.UseStaticFiles();

// 4. 쿠키 정책
app.UseCookiePolicy();

// 5. 라우팅 (엔드포인트 결정)
app.UseRouting();

// 6. 요청 제한
app.UseRateLimiter();

// 7. CORS
app.UseCors();

// 8. 인증
app.UseAuthentication();

// 9. 인가
app.UseAuthorization();

// 10. 세션
app.UseSession();

// 11. 응답 캐싱
app.UseResponseCaching();

// 12. 응답 압축
app.UseResponseCompression();

// 13. 엔드포인트 매핑
app.MapControllers();

app.Run();
```

### 순서 시각화

```
┌─────────────────────────────────────────────────────────────────┐
│                    미들웨어 순서 원칙                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  외부 요청 ──►                                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 예외 처리                                           │   │
│  │     → 모든 예외를 캐치하기 위해 가장 먼저               │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  2. HSTS / HTTPS 리다이렉션                             │   │
│  │     → 보안 우선                                         │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  3. 정적 파일                                           │   │
│  │     → 빠른 반환 (파이프라인 단축)                       │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  4. 라우팅                                              │   │
│  │     → 엔드포인트 결정                                   │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  5. CORS                                                │   │
│  │     → 인증 전에 Preflight 처리                          │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  6. 인증 → 인가                                         │   │
│  │     → 순서 필수 (인증 후 인가)                          │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  7. 엔드포인트 실행                                     │   │
│  │     → 컨트롤러/Minimal API                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ◄── 응답                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 예외 처리 미들웨어

### DeveloperExceptionPage

```csharp
// 개발 환경에서만 사용 (민감 정보 노출 주의!)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
```

### ExceptionHandler

```csharp
// 프로덕션용 예외 처리
app.UseExceptionHandler("/Error");

// 또는 인라인 처리
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        if (exceptionFeature != null)
        {
            var error = new
            {
                Message = "An error occurred",
                TraceId = Activity.Current?.Id ?? context.TraceIdentifier
            };
            await context.Response.WriteAsJsonAsync(error);
        }
    });
});
```

### 문제 세부 정보 (Problem Details)

```csharp
// .NET 7+ 권장 방식
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();

// 예외 발생 시 RFC 7807 형식으로 응답
// {
//   "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
//   "title": "An error occurred while processing your request.",
//   "status": 500,
//   "traceId": "00-..."
// }
```

### 커스텀 Problem Details

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance = context.HttpContext.Request.Path;
        context.ProblemDetails.Extensions["traceId"] =
            Activity.Current?.Id ?? context.HttpContext.TraceIdentifier;

        // 커스텀 예외 처리
        if (context.Exception is BusinessException businessEx)
        {
            context.ProblemDetails.Title = businessEx.Title;
            context.ProblemDetails.Status = businessEx.StatusCode;
            context.ProblemDetails.Detail = businessEx.Detail;
        }
    };
});
```

---

## HTTPS 미들웨어

### HTTPS 리다이렉션

```csharp
// HTTP → HTTPS 리다이렉션
app.UseHttpsRedirection();

// 설정
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 5001;
});
```

### HSTS (HTTP Strict Transport Security)

```csharp
// HTTPS 강제 헤더 추가
app.UseHsts();

// 설정
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
    options.ExcludedHosts.Add("localhost");
});
```

---

## 정적 파일 미들웨어

### 기본 설정

```csharp
// wwwroot 폴더에서 제공
app.UseStaticFiles();

// 커스텀 폴더
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.ContentRootPath, "StaticFiles")),
    RequestPath = "/files"
});
```

### 캐싱 설정

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        // 1년 캐시
        ctx.Context.Response.Headers.Append(
            "Cache-Control", "public,max-age=31536000,immutable");
    }
});
```

### 디렉토리 브라우징

```csharp
// 주의: 보안상 프로덕션에서는 비활성화 권장
builder.Services.AddDirectoryBrowser();

app.UseDirectoryBrowser(new DirectoryBrowserOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.WebRootPath, "downloads")),
    RequestPath = "/downloads"
});
```

---

## 라우팅 미들웨어

### 기본 설정

```csharp
app.UseRouting();

// 엔드포인트 미들웨어 (UseRouting과 MapXxx 사이에 인증/인가)
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.MapRazorPages();
```

### 엔드포인트 메타데이터

```csharp
app.UseRouting();

// 엔드포인트 접근
app.Use(async (context, next) =>
{
    var endpoint = context.GetEndpoint();
    if (endpoint != null)
    {
        var metadata = endpoint.Metadata;
        Console.WriteLine($"Endpoint: {endpoint.DisplayName}");

        // 커스텀 메타데이터 확인
        var customAttr = metadata.GetMetadata<MyCustomAttribute>();
    }

    await next(context);
});

app.MapControllers();
```

---

## CORS 미들웨어

### 명명된 정책

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecific", policy =>
    {
        policy.WithOrigins("https://example.com", "https://app.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials();
    });

    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

var app = builder.Build();

// 전역 적용
app.UseCors("AllowSpecific");

// 또는 엔드포인트별 적용
app.MapGet("/api/public", () => "Public")
    .RequireCors("AllowAll");

app.MapGet("/api/private", () => "Private")
    .RequireCors("AllowSpecific");
```

### 기본 정책

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://example.com")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

app.UseCors();  // 기본 정책 적용
```

---

## 인증/인가 미들웨어

### 기본 설정

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://myapp.com",
            ValidAudience = "https://myapp.com",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("my-secret-key"))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("MustBeEmployee", policy =>
        policy.RequireClaim("employee_id"));
});

var app = builder.Build();

app.UseAuthentication();  // 인증 먼저!
app.UseAuthorization();   // 인가 나중!
```

---

## 세션 미들웨어

### 설정

```csharp
builder.Services.AddDistributedMemoryCache();  // 또는 Redis
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

var app = builder.Build();

app.UseSession();

// 사용
app.MapGet("/set", async (HttpContext context) =>
{
    context.Session.SetString("Key", "Value");
    await context.Session.CommitAsync();
    return "Set!";
});

app.MapGet("/get", (HttpContext context) =>
{
    return context.Session.GetString("Key") ?? "Not found";
});
```

---

## 응답 캐싱 미들웨어

### 설정

```csharp
builder.Services.AddResponseCaching();

var app = builder.Build();

app.UseResponseCaching();

// 엔드포인트에 캐시 지시
app.MapGet("/cached", [ResponseCache(Duration = 60)] () =>
{
    return new { Time = DateTime.Now };
});

// 또는 컨트롤러에서
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Client)]
public IActionResult GetData()
{
    return Ok(new { Time = DateTime.Now });
}
```

### 출력 캐싱 (.NET 7+)

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromSeconds(10)));

    options.AddPolicy("CacheFor1Hour", builder =>
        builder.Expire(TimeSpan.FromHours(1)));

    options.AddPolicy("VaryByQuery", builder =>
        builder.SetVaryByQuery("page", "pageSize"));
});

var app = builder.Build();

app.UseOutputCache();

app.MapGet("/cached", () => DateTime.Now)
    .CacheOutput("CacheFor1Hour");

app.MapGet("/list", (int page) => $"Page {page}")
    .CacheOutput("VaryByQuery");
```

---

## 응답 압축 미들웨어

### 설정

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;  // HTTPS에서도 활성화
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json", "text/plain" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Optimal;
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.SmallestSize;
});

var app = builder.Build();

// 응답 압축은 HTTPS 리다이렉션 후에!
app.UseHttpsRedirection();
app.UseResponseCompression();
```

---

## 요청 제한 미들웨어 (.NET 7+)

### 설정

```csharp
builder.Services.AddRateLimiter(options =>
{
    // 고정 윈도우
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 10;
    });

    // 슬라이딩 윈도우
    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.SegmentsPerWindow = 6;  // 10초 단위
    });

    // 토큰 버킷
    options.AddTokenBucketLimiter("token", opt =>
    {
        opt.TokenLimit = 100;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 10;
    });

    // 거부 시 응답
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            Error = "Too many requests",
            RetryAfter = context.Lease.TryGetMetadata(
                MetadataName.RetryAfter, out var retryAfter)
                    ? retryAfter.TotalSeconds : 60
        });
    };
});

var app = builder.Build();

app.UseRateLimiter();

app.MapGet("/api/data", () => "Data")
    .RequireRateLimiting("fixed");
```

---

## 헬스 체크 미들웨어

### 설정

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddSqlServer(connectionString, name: "database")
    .AddRedis(redisConnectionString, name: "redis");

var app = builder.Build();

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = new
        {
            Status = report.Status.ToString(),
            Checks = report.Entries.Select(e => new
            {
                Name = e.Key,
                Status = e.Value.Status.ToString(),
                Duration = e.Value.Duration.TotalMilliseconds
            })
        };
        await context.Response.WriteAsJsonAsync(result);
    }
});

// 상세 헬스 체크
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // 항상 건강
});
```

---

## 면접 예상 질문

### Q1: UseAuthentication과 UseAuthorization의 순서가 중요한 이유는?

**A:** `UseAuthorization`은 인증된 사용자 정보(`ClaimsPrincipal`)를 기반으로 권한을 확인합니다. `UseAuthentication`이 먼저 실행되어 사용자를 인증해야 `User` 속성이 설정되고, 이를 기반으로 인가가 올바르게 동작합니다.

### Q2: 정적 파일 미들웨어를 왜 앞쪽에 배치하나요?

**A:** 정적 파일은 인증이나 인가가 필요 없는 경우가 많고, 파이프라인을 단축(short-circuit)하여 빠르게 응답할 수 있습니다. 불필요한 미들웨어 실행을 건너뛰어 성능을 향상시킵니다.

### Q3: CORS 미들웨어가 인증 전에 와야 하는 이유는?

**A:** 브라우저가 보내는 Preflight 요청(OPTIONS)에는 인증 헤더가 없습니다. CORS가 인증 뒤에 있으면 Preflight가 401로 실패하여 실제 요청이 전송되지 않습니다.

### Q4: ResponseCaching과 OutputCache의 차이점은?

**A:**
- **ResponseCaching**: HTTP 캐시 헤더(Cache-Control)를 기반으로 동작. 클라이언트/프록시 캐싱 용도.
- **OutputCache**: 서버 측 캐싱. HTTP 헤더와 독립적으로 동작하며, 더 세밀한 제어 가능.

---

## 참고 자료

- [ASP.NET Core 미들웨어](https://learn.microsoft.com/aspnet/core/fundamentals/middleware)
- [내장 미들웨어 순서](https://learn.microsoft.com/aspnet/core/fundamentals/middleware#middleware-order)
- [Rate Limiting](https://learn.microsoft.com/aspnet/core/performance/rate-limit)
- [Output Cache](https://learn.microsoft.com/aspnet/core/performance/caching/output)

---

## 다음 문서

→ [custom-middleware.md](./custom-middleware.md) - 커스텀 미들웨어 작성
