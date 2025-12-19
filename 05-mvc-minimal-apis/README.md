# 05. MVC & Minimal APIs

ASP.NET Core의 두 가지 웹 API 개발 방식을 비교하고 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [minimal-apis.md](./minimal-apis.md) | Minimal APIs 완벽 가이드 | 표면 |
| [mvc-controllers.md](./mvc-controllers.md) | MVC 컨트롤러 패턴 | 중간 |
| [model-binding.md](./model-binding.md) | 모델 바인딩과 검증 | 중간 |
| [filters.md](./filters.md) | 액션 필터와 엔드포인트 필터 | 실용 |

---

## 핵심 비교

### Minimal APIs vs MVC

```
┌─────────────────────────────────────────────────────────────────┐
│                    두 가지 API 스타일                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Minimal APIs (.NET 6+)                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  app.MapGet("/api/users", async (IUserService svc) =>   │   │
│  │  {                                                       │   │
│  │      return await svc.GetAllAsync();                    │   │
│  │  });                                                     │   │
│  │                                                          │   │
│  │  ✅ 간결함, 빠른 시작                                     │   │
│  │  ✅ 람다 기반, 함수형 스타일                              │   │
│  │  ✅ 최소 오버헤드                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  MVC Controllers                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  [ApiController]                                         │   │
│  │  [Route("api/[controller]")]                            │   │
│  │  public class UsersController : ControllerBase          │   │
│  │  {                                                       │   │
│  │      [HttpGet]                                          │   │
│  │      public async Task<IActionResult> GetAll() { }      │   │
│  │  }                                                       │   │
│  │                                                          │   │
│  │  ✅ 구조화, 대규모 프로젝트                               │   │
│  │  ✅ 풍부한 필터 시스템                                    │   │
│  │  ✅ 자동 모델 검증                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기능 비교표

| 기능 | Minimal APIs | MVC Controllers |
|------|-------------|-----------------|
| 학습 곡선 | 낮음 | 보통 |
| 코드량 | 적음 | 많음 |
| 모델 바인딩 | ✅ | ✅ |
| 모델 검증 | 수동/미들웨어 | ✅ 자동 |
| 필터 | 엔드포인트 필터 | 액션 필터 |
| 인증/인가 | ✅ | ✅ |
| OpenAPI | ✅ | ✅ |
| 뷰 렌더링 | ❌ | ✅ |
| 성능 | 약간 빠름 | 표준 |

---

## 빠른 시작

### Minimal APIs

```csharp
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// 엔드포인트 정의
app.MapGet("/api/users", async (IUserService service) =>
{
    return await service.GetAllAsync();
});

app.MapGet("/api/users/{id}", async (int id, IUserService service) =>
{
    var user = await service.GetByIdAsync(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.MapPost("/api/users", async (CreateUserRequest request, IUserService service) =>
{
    var user = await service.CreateAsync(request);
    return Results.Created($"/api/users/{user.Id}", user);
});

app.MapPut("/api/users/{id}", async (int id, UpdateUserRequest request, IUserService service) =>
{
    var user = await service.UpdateAsync(id, request);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});

app.MapDelete("/api/users/{id}", async (int id, IUserService service) =>
{
    var deleted = await service.DeleteAsync(id);
    return deleted ? Results.NoContent() : Results.NotFound();
});

app.Run();
```

### MVC Controllers

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

app.MapControllers();

app.Run();

// Controllers/UsersController.cs
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<User>>> GetAll()
    {
        return Ok(await _userService.GetAllAsync());
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetById(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        return user is not null ? Ok(user) : NotFound();
    }

    [HttpPost]
    public async Task<ActionResult<User>> Create(CreateUserRequest request)
    {
        var user = await _userService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult<User>> Update(int id, UpdateUserRequest request)
    {
        var user = await _userService.UpdateAsync(id, request);
        return user is not null ? Ok(user) : NotFound();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _userService.DeleteAsync(id);
        return deleted ? NoContent() : NotFound();
    }
}
```

---

## 언제 무엇을 사용할까?

### Minimal APIs 권장

- 마이크로서비스
- 간단한 API
- 빠른 프로토타이핑
- 서버리스/Lambda 함수
- 성능 중심 시나리오

### MVC Controllers 권장

- 대규모 엔터프라이즈 앱
- 복잡한 비즈니스 로직
- 기존 ASP.NET Core 프로젝트
- MVC Views 필요
- 팀 컨벤션이 MVC인 경우

---

## 공통 개념

### 라우팅

```csharp
// Minimal APIs
app.MapGet("/api/products/{id}", (int id) => ...);
app.MapGet("/api/products/{category}/{name}", (string category, string name) => ...);

// MVC
[Route("api/products/{id}")]
public IActionResult Get(int id) { ... }

[Route("api/products/{category}/{name}")]
public IActionResult Get(string category, string name) { ... }
```

### 모델 바인딩 소스

```csharp
// Minimal APIs
app.MapGet("/search", (
    [FromQuery] string query,           // ?query=test
    [FromHeader] string authorization,  // Header
    [FromRoute] int id) => ...);        // /search/{id}

app.MapPost("/users", (
    [FromBody] CreateUserRequest body,  // Request Body
    [FromServices] IUserService svc     // DI 컨테이너
) => ...);

// MVC
public IActionResult Search(
    [FromQuery] string query,
    [FromHeader] string authorization,
    [FromRoute] int id) { ... }
```

### 응답 유형

```csharp
// Minimal APIs
app.MapGet("/api/users/{id}", async (int id, IUserService svc) =>
{
    var user = await svc.GetByIdAsync(id);

    return user switch
    {
        null => Results.NotFound(),
        _ => Results.Ok(user)
    };
});

// 타입화된 결과 (OpenAPI 문서화 개선)
app.MapGet("/api/users/{id}", async Task<Results<Ok<User>, NotFound>> (int id, IUserService svc) =>
{
    var user = await svc.GetByIdAsync(id);
    return user is not null ? TypedResults.Ok(user) : TypedResults.NotFound();
});

// MVC
[HttpGet("{id}")]
[ProducesResponseType(typeof(User), 200)]
[ProducesResponseType(404)]
public async Task<IActionResult> GetById(int id)
{
    var user = await _userService.GetByIdAsync(id);
    return user is not null ? Ok(user) : NotFound();
}
```

---

## 면접 빈출 질문

1. **Minimal APIs와 MVC의 차이점은?**
2. **Minimal APIs에서 필터를 어떻게 사용하나요?**
3. **[ApiController] 속성의 역할은?**
4. **모델 바인딩은 어떻게 동작하나요?**
5. **TypedResults의 장점은?**

---

## 다음 단계

→ [06. Data Access](../06-data-access/) - 데이터 액세스 레이어
