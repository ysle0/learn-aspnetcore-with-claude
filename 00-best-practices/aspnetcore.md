# ASP.NET Core 베스트 프랙티스

공식 문서와 커뮤니티에서 권장하는 ASP.NET Core 개발 원칙입니다.

---

## 1. 의존성 주입 원칙

### 서비스 로케이터 패턴 피하기

```csharp
// ❌ 나쁨: 서비스 로케이터
public class OrderService
{
    private readonly IServiceProvider _sp;

    public OrderService(IServiceProvider sp)
    {
        _sp = sp;
    }

    public void Process()
    {
        var repo = _sp.GetRequiredService<IOrderRepository>();  // 숨겨진 의존성
        repo.Save(new Order());
    }
}

// ✅ 좋음: 명시적 의존성
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;  // 명시적!
    }

    public void Process()
    {
        _repository.Save(new Order());
    }
}
```

### Captive Dependency 방지

```csharp
// ❌ 나쁨: Singleton이 Scoped를 주입받음
services.AddSingleton<MySingletonService>();  // Singleton
services.AddScoped<MyScopedService>();        // Scoped

public class MySingletonService
{
    private readonly MyScopedService _scoped;  // 포획됨!

    public MySingletonService(MyScopedService scoped)
    {
        _scoped = scoped;
    }
}

// ✅ 좋음: IServiceScopeFactory 사용
public class MySingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MySingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public void DoWork()
    {
        using var scope = _scopeFactory.CreateScope();
        var scoped = scope.ServiceProvider.GetRequiredService<MyScopedService>();
        scoped.Process();
    }
}
```

### 옵션 패턴 활용

```csharp
// ✅ 설정을 타입 안전하게 관리
public class EmailOptions
{
    public const string SectionName = "Email";

    [Required]
    public string SmtpServer { get; set; } = "";

    [Range(1, 65535)]
    public int Port { get; set; } = 587;
}

// 등록 및 검증
builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // 앱 시작 시 검증!

// 사용
public class EmailService(IOptions<EmailOptions> options)
{
    private readonly EmailOptions _options = options.Value;
}
```

---

## 2. 컨트롤러 설계 원칙

### 가벼운 컨트롤러

```csharp
// ❌ 나쁨: 비즈니스 로직이 컨트롤러에
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request)
    {
        // 비즈니스 로직이 컨트롤러에 있음
        if (request.Items.Count == 0)
            return BadRequest("Order must have items");

        var order = new Order { /* ... */ };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();

        // 이메일 발송
        await SendEmailAsync(order);

        return Created($"/api/orders/{order.Id}", order);
    }
}

// ✅ 좋음: 서비스에 위임
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    [HttpPost]
    public async Task<ActionResult<Order>> Create(CreateOrderRequest request)
    {
        var order = await _orderService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }
}
```

### ActionResult<T> 사용

```csharp
// ❌ 나쁨: IActionResult만 사용 (OpenAPI에 타입 정보 없음)
[HttpGet("{id}")]
public async Task<IActionResult> GetById(int id)
{
    var order = await _orderService.GetByIdAsync(id);
    if (order == null) return NotFound();
    return Ok(order);
}

// ✅ 좋음: ActionResult<T> 사용
[HttpGet("{id}")]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult<Order>> GetById(int id)
{
    var order = await _orderService.GetByIdAsync(id);
    if (order == null) return NotFound();
    return order;  // 암시적 Ok()
}
```

---

## 3. 모델 검증

### FluentValidation 활용

```csharp
// ✅ 복잡한 검증 로직 분리
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator(IProductRepository productRepo)
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .WithMessage("Customer ID is required");

        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item");

        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId)
                .MustAsync(async (id, ct) => await productRepo.ExistsAsync(id, ct))
                .WithMessage("Product not found");

            item.RuleFor(i => i.Quantity)
                .GreaterThan(0)
                .WithMessage("Quantity must be positive");
        });
    }
}

// 등록
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Minimal API 필터로 자동 검증
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices
            .GetService<IValidator<T>>();

        if (validator is null)
            return await next(context);

        var argument = context.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null)
            return await next(context);

        var result = await validator.ValidateAsync(argument);
        if (!result.IsValid)
            return Results.ValidationProblem(result.ToDictionary());

        return await next(context);
    }
}
```

---

## 4. 예외 처리

### 전역 예외 처리

```csharp
// ✅ Problem Details 활용
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] =
            ctx.HttpContext.TraceIdentifier;
    };
});

app.UseExceptionHandler();
app.UseStatusCodePages();

// ✅ 커스텀 예외 → HTTP 상태 매핑
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;

        var (statusCode, title) = exception switch
        {
            NotFoundException => (404, "Resource not found"),
            ValidationException => (400, "Validation failed"),
            UnauthorizedException => (401, "Unauthorized"),
            _ => (500, "An error occurred")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = exception?.Message,
            Instance = context.Request.Path
        });
    });
});
```

### 도메인 예외

```csharp
// ✅ 명확한 도메인 예외
public class DomainException : Exception
{
    public string Code { get; }

    public DomainException(string code, string message) : base(message)
    {
        Code = code;
    }
}

public class OrderNotFoundException : DomainException
{
    public OrderNotFoundException(int orderId)
        : base("ORDER_NOT_FOUND", $"Order {orderId} not found") { }
}

public class InsufficientStockException : DomainException
{
    public InsufficientStockException(int productId, int requested, int available)
        : base("INSUFFICIENT_STOCK",
            $"Product {productId}: requested {requested}, available {available}") { }
}
```

---

## 5. 로깅 원칙

### 구조화된 로깅

```csharp
// ❌ 나쁨: 문자열 보간
_logger.LogInformation($"Order {order.Id} created by {user.Name}");

// ✅ 좋음: 구조화된 로깅
_logger.LogInformation(
    "Order {OrderId} created by {UserName}",
    order.Id,
    user.Name);

// ✅ 민감 정보 제외
_logger.LogInformation(
    "User {UserId} logged in",  // Email, Password 등 제외!
    user.Id);

// ✅ 고성능 로깅 (Source Generator)
public static partial class LoggerExtensions
{
    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "Order {OrderId} created by {UserId}")]
    public static partial void LogOrderCreated(
        this ILogger logger, int orderId, int userId);
}
```

### 로그 레벨 가이드

```csharp
// Trace: 매우 상세한 디버그 정보
_logger.LogTrace("Entering method with parameter {Param}", param);

// Debug: 개발 중 유용한 정보
_logger.LogDebug("Cache miss for key {Key}", key);

// Information: 정상 동작 기록
_logger.LogInformation("Order {OrderId} shipped", orderId);

// Warning: 잠재적 문제
_logger.LogWarning("Retry attempt {Attempt} for order {OrderId}", attempt, orderId);

// Error: 오류 발생 (복구 가능)
_logger.LogError(ex, "Failed to process order {OrderId}", orderId);

// Critical: 심각한 오류 (앱 크래시 가능)
_logger.LogCritical(ex, "Database connection lost");
```

---

## 6. 설정 관리

### 환경별 설정

```json
// appsettings.json (기본)
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}

// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

### 비밀 정보 관리

```csharp
// ❌ 나쁨: 하드코딩
var apiKey = "sk_live_xxxxx";

// ❌ 나쁨: appsettings.json에 비밀 저장

// ✅ 좋음: 환경 변수
var apiKey = Environment.GetEnvironmentVariable("API_KEY");

// ✅ 좋음: User Secrets (개발)
// dotnet user-secrets set "ApiKey" "sk_live_xxxxx"
var apiKey = builder.Configuration["ApiKey"];

// ✅ 좋음: Azure Key Vault / AWS Secrets Manager
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());
```

---

## 7. 보안 원칙

### HTTPS 강제

```csharp
// ✅ HTTPS 리다이렉션
app.UseHttpsRedirection();

// ✅ HSTS (프로덕션)
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
```

### 인증/인가

```csharp
// ✅ 정책 기반 인가
builder.Services.AddAuthorization(options =>
{
    // 기본 정책: 인증된 사용자
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // 커스텀 정책
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanEditOrders", policy =>
        policy.RequireClaim("permission", "orders:edit"));
});

// 미들웨어 순서 중요!
app.UseAuthentication();
app.UseAuthorization();
```

### 입력 검증

```csharp
// ✅ 파라미터 검증
app.MapGet("/users/{id:int:min(1)}", (int id) => ...);

// ✅ SQL Injection 방지
// EF Core 사용 시 자동 파라미터화
var users = await _db.Users
    .Where(u => u.Name == name)  // 안전
    .ToListAsync();

// Dapper 사용 시 파라미터 사용
var users = await connection.QueryAsync<User>(
    "SELECT * FROM Users WHERE Name = @Name",  // 안전
    new { Name = name });

// ❌ 절대 금지: 문자열 연결
var query = $"SELECT * FROM Users WHERE Name = '{name}'";  // SQL Injection!
```

---

## 8. 성능 원칙

### 비동기 I/O

```csharp
// ✅ 모든 I/O는 비동기로
[HttpGet]
public async Task<IActionResult> Get(CancellationToken cancellationToken)
{
    var data = await _dbContext.Orders
        .ToListAsync(cancellationToken);
    return Ok(data);
}

// ✅ CancellationToken 전파
public async Task<Order?> GetOrderAsync(int id, CancellationToken cancellationToken)
{
    return await _dbContext.Orders
        .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
}
```

### 응답 캐싱

```csharp
// ✅ Output Cache (.NET 7+)
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromMinutes(5)));
    options.AddPolicy("NoCache", builder => builder.NoCache());
});

app.UseOutputCache();

app.MapGet("/products", async (AppDbContext db) =>
    await db.Products.ToListAsync())
    .CacheOutput();

app.MapGet("/cart", GetCart)
    .CacheOutput("NoCache");
```

### 응답 압축

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});

app.UseResponseCompression();
```

---

## 9. 테스트 용이성

### 인터페이스 분리

```csharp
// ✅ 테스트 가능한 설계
public interface ITimeProvider
{
    DateTime UtcNow { get; }
}

public class SystemTimeProvider : ITimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

// 테스트에서
public class FakeTimeProvider : ITimeProvider
{
    public DateTime UtcNow { get; set; } = new DateTime(2024, 1, 1);
}

// 또는 .NET 8의 TimeProvider 사용
builder.Services.AddSingleton(TimeProvider.System);
```

### 통합 테스트

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                // 테스트용 서비스 교체
                services.RemoveAll<IEmailService>();
                services.AddSingleton<IEmailService, FakeEmailService>();
            });
        }).CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ReturnsCreated()
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new
        {
            CustomerId = 1,
            Items = new[] { new { ProductId = 1, Quantity = 2 } }
        });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

---

## 안티패턴 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASP.NET Core 안티패턴                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DI:                                                            │
│  • 서비스 로케이터 패턴 사용                                     │
│  • Singleton에서 Scoped 서비스 주입                              │
│  • 생성자에서 비동기 작업 수행                                   │
│                                                                 │
│  컨트롤러:                                                       │
│  • 컨트롤러에 비즈니스 로직 작성                                 │
│  • 동기 I/O 사용                                                │
│  • CancellationToken 무시                                       │
│                                                                 │
│  보안:                                                          │
│  • 문자열 연결로 SQL 작성                                       │
│  • 민감 정보 로깅                                               │
│  • 비밀을 appsettings.json에 저장                               │
│                                                                 │
│  성능:                                                          │
│  • HttpClient 매번 생성                                         │
│  • .Result/.Wait() 사용                                         │
│  • 대용량 데이터 한번에 로드                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 면접 예상 질문

### Q1: 컨트롤러를 가볍게 유지해야 하는 이유는?

**A:** 테스트 용이성, 관심사 분리, 재사용성을 위해서입니다. 비즈니스 로직은 서비스 레이어에 두고, 컨트롤러는 HTTP 요청/응답 처리만 담당합니다.

### Q2: appsettings.json에 비밀을 저장하면 안 되는 이유는?

**A:** 소스 제어에 포함되어 노출 위험이 있습니다. 개발 시 User Secrets, 프로덕션에서 환경 변수나 Key Vault를 사용해야 합니다.

### Q3: CancellationToken을 사용해야 하는 이유는?

**A:** 클라이언트가 요청을 취소하거나 앱이 종료될 때 불필요한 작업을 중단할 수 있습니다. 리소스를 절약하고 graceful shutdown을 지원합니다.

---

## 참고 자료

- [ASP.NET Core 모범 사례](https://learn.microsoft.com/aspnet/core/fundamentals/best-practices)
- [ASP.NET Core 성능 모범 사례](https://learn.microsoft.com/aspnet/core/performance/performance-best-practices)
- [ASP.NET Core 보안](https://learn.microsoft.com/aspnet/core/security)

---

## 다음 문서

→ [api-design.md](./api-design.md) - API 설계 원칙
