# Minimal APIs 완벽 가이드

## 개요

Minimal APIs는 .NET 6에서 도입된 경량 API 개발 방식입니다. 최소한의 코드로 빠르게 HTTP 엔드포인트를 정의할 수 있습니다.

---

## 기본 구조

### Hello World

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

### HTTP 메서드 매핑

```csharp
var app = builder.Build();

app.MapGet("/api/items", () => { });       // GET
app.MapPost("/api/items", () => { });      // POST
app.MapPut("/api/items/{id}", () => { });  // PUT
app.MapPatch("/api/items/{id}", () => { });// PATCH
app.MapDelete("/api/items/{id}", () => { });// DELETE

// 여러 메서드 한 번에
app.MapMethods("/api/items", new[] { "OPTIONS", "HEAD" }, () => { });
```

---

## 라우팅

### 경로 파라미터

```csharp
// 기본 경로 파라미터
app.MapGet("/users/{id}", (int id) => $"User {id}");

// 여러 파라미터
app.MapGet("/posts/{year}/{month}/{slug}",
    (int year, int month, string slug) =>
        $"Post: {year}/{month}/{slug}");

// 선택적 파라미터
app.MapGet("/products/{category?}",
    (string? category) =>
        category ?? "All products");

// 기본값
app.MapGet("/search/{query=all}",
    (string query) => $"Searching: {query}");

// 와일드카드
app.MapGet("/files/{*path}",
    (string path) => $"File: {path}");
// /files/docs/readme.md → path = "docs/readme.md"
```

### 라우트 제약

```csharp
// 숫자만
app.MapGet("/users/{id:int}", (int id) => $"User {id}");

// 범위
app.MapGet("/age/{value:range(1,150)}", (int value) => $"Age: {value}");

// 길이
app.MapGet("/code/{value:length(6)}", (string value) => $"Code: {value}");

// 정규식
app.MapGet("/phone/{number:regex(^\\d{{3}}-\\d{{4}}-\\d{{4}}$)}",
    (string number) => $"Phone: {number}");

// 여러 제약 조합
app.MapGet("/products/{id:int:min(1)}", (int id) => $"Product {id}");

// 사용 가능한 제약:
// int, long, float, double, decimal, bool, datetime, guid
// minlength(n), maxlength(n), length(n), length(min,max)
// min(n), max(n), range(min,max)
// alpha, regex(pattern)
// required
```

---

## 파라미터 바인딩

### 바인딩 소스

```csharp
// 라우트에서 바인딩
app.MapGet("/users/{id}", (int id) => ...);

// 쿼리스트링에서 바인딩
app.MapGet("/search", (string query, int? page) => ...);
// /search?query=test&page=1

// 헤더에서 바인딩
app.MapGet("/api/data", ([FromHeader(Name = "X-Api-Key")] string apiKey) => ...);

// 바디에서 바인딩 (POST, PUT 등)
app.MapPost("/users", ([FromBody] CreateUserRequest request) => ...);

// 서비스 주입
app.MapGet("/users", ([FromServices] IUserService service) => ...);

// 여러 소스 조합
app.MapPost("/users/{id}/comments",
    (int id,
     [FromBody] CreateCommentRequest request,
     [FromHeader(Name = "Authorization")] string auth,
     [FromServices] ICommentService service) => ...);
```

### 특수 타입 바인딩

```csharp
// HttpContext
app.MapGet("/info", (HttpContext context) =>
{
    return $"Path: {context.Request.Path}";
});

// HttpRequest, HttpResponse
app.MapGet("/request", (HttpRequest request, HttpResponse response) =>
{
    response.Headers.Append("X-Custom", "value");
    return request.Path.ToString();
});

// CancellationToken
app.MapGet("/long-running", async (CancellationToken cancellationToken) =>
{
    await Task.Delay(5000, cancellationToken);
    return "Done";
});

// ClaimsPrincipal (인증된 사용자)
app.MapGet("/me", (ClaimsPrincipal user) =>
{
    return user.Identity?.Name ?? "Anonymous";
});
```

### 커스텀 바인딩 (TryParse)

```csharp
// TryParse 정적 메서드를 가진 타입
public record Point(int X, int Y)
{
    public static bool TryParse(string? value, out Point? result)
    {
        result = null;
        if (value is null) return false;

        var parts = value.Split(',');
        if (parts.Length != 2) return false;

        if (int.TryParse(parts[0], out var x) &&
            int.TryParse(parts[1], out var y))
        {
            result = new Point(x, y);
            return true;
        }
        return false;
    }
}

app.MapGet("/location", (Point point) => $"X={point.X}, Y={point.Y}");
// /location?point=10,20
```

### BindAsync 패턴

```csharp
public record Pagination(int Page, int PageSize)
{
    public static ValueTask<Pagination?> BindAsync(HttpContext context)
    {
        var page = int.TryParse(context.Request.Query["page"], out var p) ? p : 1;
        var pageSize = int.TryParse(context.Request.Query["pageSize"], out var ps) ? ps : 10;

        return ValueTask.FromResult<Pagination?>(new Pagination(page, pageSize));
    }
}

app.MapGet("/items", (Pagination pagination) =>
    $"Page {pagination.Page}, Size {pagination.PageSize}");
```

---

## 응답 처리

### 기본 응답

```csharp
// 문자열 → text/plain
app.MapGet("/text", () => "Hello");

// 객체 → application/json
app.MapGet("/json", () => new { Name = "John", Age = 30 });

// IResult
app.MapGet("/result", () => Results.Ok(new { Message = "Success" }));
```

### Results 헬퍼

```csharp
app.MapGet("/users/{id}", async (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);

    return user switch
    {
        null => Results.NotFound(),
        _ => Results.Ok(user)
    };
});

// 주요 Results 메서드
Results.Ok(value)                    // 200
Results.Created(uri, value)          // 201
Results.NoContent()                  // 204
Results.BadRequest(error)            // 400
Results.Unauthorized()               // 401
Results.Forbid()                     // 403
Results.NotFound()                   // 404
Results.Conflict(error)              // 409
Results.UnprocessableEntity(error)   // 422
Results.Problem(problemDetails)      // RFC 7807 Problem Details

// 리다이렉트
Results.Redirect(url)                // 302
Results.RedirectPermanent(url)       // 301
Results.LocalRedirect(localUrl)      // 302 (로컬)
Results.RedirectToRoute(routeName)   // 라우트로 리다이렉트

// 파일
Results.File(bytes, contentType, fileName)
Results.Stream(stream, contentType)
Results.PhysicalFile(path, contentType)

// 기타
Results.Json(object, options)
Results.Text(content, contentType)
Results.Bytes(bytes, contentType)
Results.Empty                        // 빈 응답
```

### TypedResults (OpenAPI 향상)

```csharp
// TypedResults는 반환 타입 정보를 포함
app.MapGet("/users/{id}",
    async Task<Results<Ok<User>, NotFound>> (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user is not null
        ? TypedResults.Ok(user)
        : TypedResults.NotFound();
});

// OpenAPI 문서에 정확한 응답 타입 표시
```

---

## 라우트 그룹

### 기본 그룹화

```csharp
var users = app.MapGroup("/api/users");

users.MapGet("/", () => "Get all users");
users.MapGet("/{id}", (int id) => $"Get user {id}");
users.MapPost("/", () => "Create user");
users.MapPut("/{id}", (int id) => $"Update user {id}");
users.MapDelete("/{id}", (int id) => $"Delete user {id}");

// 결과: /api/users, /api/users/{id}, ...
```

### 중첩 그룹

```csharp
var api = app.MapGroup("/api");

var v1 = api.MapGroup("/v1");
var v2 = api.MapGroup("/v2");

v1.MapGet("/users", () => "V1 Users");  // /api/v1/users
v2.MapGet("/users", () => "V2 Users");  // /api/v2/users
```

### 그룹에 공통 설정

```csharp
var adminApi = app.MapGroup("/api/admin")
    .RequireAuthorization("AdminPolicy")
    .AddEndpointFilter<LoggingFilter>()
    .WithTags("Admin");

adminApi.MapGet("/users", GetUsers);
adminApi.MapPost("/users", CreateUser);

// 모든 /api/admin/* 엔드포인트에 인증/필터/태그 적용
```

---

## 검증

### FluentValidation 통합

```csharp
// NuGet: FluentValidation.DependencyInjectionExtensions
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Name).NotEmpty().MinimumLength(2).MaximumLength(100);
        RuleFor(x => x.Age).InclusiveBetween(1, 150);
    }
}

// 등록
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// 사용
app.MapPost("/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService service) =>
{
    var validationResult = await validator.ValidateAsync(request);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    var user = await service.CreateAsync(request);
    return Results.Created($"/users/{user.Id}", user);
});
```

### 엔드포인트 필터로 검증

```csharp
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

// 사용
app.MapPost("/users", CreateUser)
    .AddEndpointFilter<ValidationFilter<CreateUserRequest>>();
```

---

## 엔드포인트 필터

### 기본 필터

```csharp
app.MapGet("/", () => "Hello")
    .AddEndpointFilter(async (context, next) =>
    {
        Console.WriteLine("Before");
        var result = await next(context);
        Console.WriteLine("After");
        return result;
    });
```

### IEndpointFilter 구현

```csharp
public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var path = context.HttpContext.Request.Path;
        _logger.LogInformation("Request to {Path}", path);

        var stopwatch = Stopwatch.StartNew();
        var result = await next(context);
        stopwatch.Stop();

        _logger.LogInformation("Request to {Path} completed in {Duration}ms",
            path, stopwatch.ElapsedMilliseconds);

        return result;
    }
}

// 사용
app.MapGet("/", GetData).AddEndpointFilter<LoggingFilter>();
```

### 필터 체인

```csharp
app.MapPost("/users", CreateUser)
    .AddEndpointFilter<LoggingFilter>()        // 1번째 실행
    .AddEndpointFilter<ValidationFilter>()     // 2번째 실행
    .AddEndpointFilter<AuthorizationFilter>(); // 3번째 실행

// 실행 순서:
// Logging(Before) → Validation(Before) → Auth(Before)
// → Handler
// Auth(After) → Validation(After) → Logging(After)
```

---

## 인증/인가

### 설정

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidIssuer = "https://myapp.com",
            ValidAudience = "https://myapp.com",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("your-secret-key"))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
```

### 엔드포인트에 적용

```csharp
// 인증 필요
app.MapGet("/protected", () => "Secret data")
    .RequireAuthorization();

// 특정 정책
app.MapGet("/admin", () => "Admin only")
    .RequireAuthorization("AdminOnly");

// 역할
app.MapGet("/managers", () => "Managers")
    .RequireAuthorization(policy => policy.RequireRole("Manager", "Admin"));

// 익명 허용 (전역 인증 시)
app.MapGet("/public", () => "Anyone can see")
    .AllowAnonymous();
```

---

## OpenAPI (Swagger)

### 설정

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "Sample API with Minimal APIs"
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

### 메타데이터 추가

```csharp
app.MapGet("/users/{id}", GetUser)
    .WithName("GetUser")                     // 작업 ID
    .WithTags("Users")                       // 그룹화
    .WithSummary("Get a user by ID")         // 요약
    .WithDescription("Retrieves a user by their unique identifier")
    .Produces<User>(200)                     // 성공 응답
    .Produces(404)                           // 404 응답
    .WithOpenApi();

// 또는 TypedResults 사용 시 자동 추론
app.MapGet("/users/{id}",
    async Task<Results<Ok<User>, NotFound>> (int id) =>
    {
        // OpenAPI 문서에 자동 반영
    });
```

---

## 파일 업로드

```csharp
app.MapPost("/upload", async (IFormFile file) =>
{
    if (file.Length == 0)
        return Results.BadRequest("No file uploaded");

    var path = Path.Combine("uploads", file.FileName);
    using var stream = File.OpenWrite(path);
    await file.CopyToAsync(stream);

    return Results.Ok(new { FileName = file.FileName, Size = file.Length });
})
.DisableAntiforgery();  // 필요시

// 여러 파일
app.MapPost("/upload-multiple", async (IFormFileCollection files) =>
{
    foreach (var file in files)
    {
        // 처리
    }
    return Results.Ok($"Uploaded {files.Count} files");
});
```

---

## 면접 예상 질문

### Q1: Minimal APIs의 장단점은?

**A:**
- **장점**: 코드 간결, 학습 곡선 낮음, 성능 우수, 빠른 개발
- **단점**: 구조화 어려움, 자동 검증 없음, 뷰 렌더링 없음

### Q2: TypedResults와 Results의 차이는?

**A:** `TypedResults`는 컴파일 타임에 반환 타입 정보를 포함하여 OpenAPI 문서에 정확한 응답 타입이 표시됩니다. `Results`는 `IResult`를 반환하여 타입 정보가 손실됩니다.

### Q3: 엔드포인트 필터와 미들웨어의 차이는?

**A:**
- **미들웨어**: 모든 요청에 적용, 파이프라인 전체에서 동작
- **엔드포인트 필터**: 특정 엔드포인트에만 적용, 엔드포인트 메타데이터 접근 가능, MVC 액션 필터와 유사

---

## 참고 자료

- [Minimal APIs 개요](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis)
- [Minimal APIs 참조](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/min-api-filters)
- [라우트 핸들러](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/route-handlers)

---

## 다음 문서

→ [mvc-controllers.md](./mvc-controllers.md) - MVC 컨트롤러 패턴
