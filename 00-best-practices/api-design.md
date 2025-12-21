# API 설계 베스트 프랙티스

RESTful API와 Clean Code 원칙에 기반한 API 설계 가이드입니다.

---

## 1. RESTful 리소스 설계

### 리소스 명명 규칙

```
┌─────────────────────────────────────────────────────────────────┐
│                    URL 설계 원칙                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ 좋은 예:                                                    │
│  GET    /api/users              → 사용자 목록                   │
│  GET    /api/users/123          → 특정 사용자                   │
│  POST   /api/users              → 사용자 생성                   │
│  PUT    /api/users/123          → 사용자 수정                   │
│  DELETE /api/users/123          → 사용자 삭제                   │
│  GET    /api/users/123/orders   → 사용자의 주문 목록            │
│                                                                 │
│  ❌ 나쁜 예:                                                    │
│  GET    /api/getUsers           → 동사 사용                    │
│  POST   /api/createUser         → 동사 사용                    │
│  GET    /api/user/123           → 단수형 사용                  │
│  DELETE /api/deleteUser/123     → 동사 사용                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 구현 예시

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // GET /api/users
    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetAll()

    // GET /api/users/123
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetById(int id)

    // POST /api/users
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(CreateUserRequest request)

    // PUT /api/users/123
    [HttpPut("{id}")]
    public async Task<ActionResult<UserDto>> Update(int id, UpdateUserRequest request)

    // DELETE /api/users/123
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)

    // GET /api/users/123/orders
    [HttpGet("{id}/orders")]
    public async Task<ActionResult<IEnumerable<OrderDto>>> GetUserOrders(int id)
}
```

---

## 2. HTTP 상태 코드

### 상태 코드 가이드

```csharp
// 2xx 성공
return Ok(data);                    // 200: 조회 성공
return Created(uri, data);          // 201: 생성 성공
return NoContent();                 // 204: 삭제 성공

// 4xx 클라이언트 오류
return BadRequest(errors);          // 400: 잘못된 요청
return Unauthorized();              // 401: 인증 필요
return Forbid();                    // 403: 권한 없음
return NotFound();                  // 404: 리소스 없음
return Conflict();                  // 409: 충돌 (중복 등)
return UnprocessableEntity(errors); // 422: 검증 실패

// 5xx 서버 오류
return StatusCode(500);             // 500: 서버 오류 (자동 처리)
return StatusCode(503);             // 503: 서비스 불가
```

### 일관된 응답 형식

```csharp
// ✅ 성공 응답
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  }
}

// ✅ 에러 응답 (RFC 7807 Problem Details)
{
  "type": "https://example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "instance": "/api/users",
  "errors": {
    "email": ["Invalid email format"],
    "age": ["Age must be between 1 and 150"]
  }
}

// 구현
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
app.UseStatusCodePages();
```

---

## 3. 요청/응답 DTO 설계

### 입력과 출력 분리

```csharp
// ✅ 생성용 DTO
public record CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; init; } = "";

    [Required]
    [EmailAddress]
    public string Email { get; init; } = "";

    [Required]
    [MinLength(8)]
    public string Password { get; init; } = "";
}

// ✅ 수정용 DTO (부분 수정 가능)
public record UpdateUserRequest
{
    [StringLength(100, MinimumLength = 2)]
    public string? Name { get; init; }

    [EmailAddress]
    public string? Email { get; init; }
}

// ✅ 응답용 DTO (민감 정보 제외)
public record UserDto
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";
    public DateTime CreatedAt { get; init; }
    // Password 제외!
}

// ✅ 목록 응답
public record PagedResponse<T>
{
    public IEnumerable<T> Items { get; init; } = [];
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalCount { get; init; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}
```

### Mapping

```csharp
// ✅ 수동 매핑 (간단한 경우)
public static class UserMapper
{
    public static UserDto ToDto(this User user) => new()
    {
        Id = user.Id,
        Name = user.Name,
        Email = user.Email,
        CreatedAt = user.CreatedAt
    };

    public static User ToEntity(this CreateUserRequest request) => new()
    {
        Name = request.Name,
        Email = request.Email,
        PasswordHash = HashPassword(request.Password),
        CreatedAt = DateTime.UtcNow
    };
}

// ✅ AutoMapper 또는 Mapperly (복잡한 경우)
// Mapperly: Source Generator 기반 (성능 최고)
[Mapper]
public partial class UserMapper
{
    public partial UserDto ToDto(User user);
    public partial User ToEntity(CreateUserRequest request);
}
```

---

## 4. 페이지네이션

### 구현

```csharp
public record PaginationQuery
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;

    [Range(1, 100)]
    public int PageSize { get; init; } = 10;
}

[HttpGet]
public async Task<ActionResult<PagedResponse<UserDto>>> GetAll(
    [FromQuery] PaginationQuery pagination,
    CancellationToken cancellationToken)
{
    var totalCount = await _dbContext.Users.CountAsync(cancellationToken);

    var users = await _dbContext.Users
        .OrderBy(u => u.Id)
        .Skip((pagination.Page - 1) * pagination.PageSize)
        .Take(pagination.PageSize)
        .Select(u => u.ToDto())
        .ToListAsync(cancellationToken);

    return Ok(new PagedResponse<UserDto>
    {
        Items = users,
        Page = pagination.Page,
        PageSize = pagination.PageSize,
        TotalCount = totalCount
    });
}
```

### 커서 기반 페이지네이션 (대용량)

```csharp
public record CursorQuery
{
    public int? Cursor { get; init; }  // 마지막으로 본 ID
    public int Limit { get; init; } = 20;
}

public record CursorResponse<T>
{
    public IEnumerable<T> Items { get; init; } = [];
    public int? NextCursor { get; init; }
    public bool HasMore { get; init; }
}

[HttpGet]
public async Task<ActionResult<CursorResponse<UserDto>>> GetAll(
    [FromQuery] CursorQuery query,
    CancellationToken cancellationToken)
{
    var usersQuery = _dbContext.Users.OrderBy(u => u.Id);

    if (query.Cursor.HasValue)
    {
        usersQuery = usersQuery.Where(u => u.Id > query.Cursor.Value);
    }

    var users = await usersQuery
        .Take(query.Limit + 1)  // 하나 더 가져와서 HasMore 판단
        .Select(u => u.ToDto())
        .ToListAsync(cancellationToken);

    var hasMore = users.Count > query.Limit;
    if (hasMore)
    {
        users = users[..query.Limit];
    }

    return Ok(new CursorResponse<UserDto>
    {
        Items = users,
        NextCursor = users.LastOrDefault()?.Id,
        HasMore = hasMore
    });
}
```

---

## 5. 필터링과 정렬

### 구현

```csharp
public record UserFilterQuery : PaginationQuery
{
    public string? Name { get; init; }
    public string? Email { get; init; }
    public bool? IsActive { get; init; }
    public string? SortBy { get; init; }
    public bool SortDescending { get; init; }
}

[HttpGet]
public async Task<ActionResult<PagedResponse<UserDto>>> GetAll(
    [FromQuery] UserFilterQuery query,
    CancellationToken cancellationToken)
{
    var usersQuery = _dbContext.Users.AsQueryable();

    // 필터링
    if (!string.IsNullOrWhiteSpace(query.Name))
    {
        usersQuery = usersQuery.Where(u =>
            u.Name.Contains(query.Name));
    }

    if (!string.IsNullOrWhiteSpace(query.Email))
    {
        usersQuery = usersQuery.Where(u =>
            u.Email.Contains(query.Email));
    }

    if (query.IsActive.HasValue)
    {
        usersQuery = usersQuery.Where(u => u.IsActive == query.IsActive.Value);
    }

    // 정렬
    usersQuery = query.SortBy?.ToLower() switch
    {
        "name" => query.SortDescending
            ? usersQuery.OrderByDescending(u => u.Name)
            : usersQuery.OrderBy(u => u.Name),
        "email" => query.SortDescending
            ? usersQuery.OrderByDescending(u => u.Email)
            : usersQuery.OrderBy(u => u.Email),
        "createdat" => query.SortDescending
            ? usersQuery.OrderByDescending(u => u.CreatedAt)
            : usersQuery.OrderBy(u => u.CreatedAt),
        _ => usersQuery.OrderBy(u => u.Id)
    };

    // 페이지네이션
    var totalCount = await usersQuery.CountAsync(cancellationToken);
    var users = await usersQuery
        .Skip((query.Page - 1) * query.PageSize)
        .Take(query.PageSize)
        .Select(u => u.ToDto())
        .ToListAsync(cancellationToken);

    return Ok(new PagedResponse<UserDto> { /* ... */ });
}
```

---

## 6. API 버전 관리

### URL 경로 방식 (권장)

```csharp
// ✅ URL 경로에 버전 포함
[ApiController]
[Route("api/v1/users")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "v1" });
}

[ApiController]
[Route("api/v2/users")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "v2", "new-format" });
}

// 또는 Asp.Versioning 패키지 사용
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok("v1");

    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok("v2");
}
```

---

## 7. API 문서화 (OpenAPI)

### Swagger 설정

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "API Description",
        Contact = new OpenApiContact
        {
            Name = "Support",
            Email = "support@example.com"
        }
    });

    // JWT 인증 설정
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.Http,
        Scheme = "bearer"
    });

    // XML 주석 포함
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));
});

// 개발 환경에서만 활성화
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

### XML 주석

```csharp
/// <summary>
/// 사용자 목록을 조회합니다.
/// </summary>
/// <param name="query">페이지네이션 및 필터 옵션</param>
/// <param name="cancellationToken">취소 토큰</param>
/// <returns>페이지네이션된 사용자 목록</returns>
/// <response code="200">조회 성공</response>
/// <response code="400">잘못된 요청 파라미터</response>
[HttpGet]
[ProducesResponseType(typeof(PagedResponse<UserDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<ActionResult<PagedResponse<UserDto>>> GetAll(
    [FromQuery] UserFilterQuery query,
    CancellationToken cancellationToken)
```

---

## 8. 멱등성 (Idempotency)

### 원칙

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP 메서드 멱등성                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  메서드     멱등?   안전?   설명                                │
│  GET        ✅      ✅      조회                                │
│  HEAD       ✅      ✅      헤더만 조회                          │
│  OPTIONS    ✅      ✅      옵션 조회                            │
│  PUT        ✅      ❌      전체 교체                            │
│  DELETE     ✅      ❌      삭제                                 │
│  POST       ❌      ❌      생성                                 │
│  PATCH      ❌      ❌      부분 수정                            │
│                                                                 │
│  멱등: 여러 번 호출해도 결과가 같음                              │
│  안전: 리소스를 변경하지 않음                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Idempotency Key 구현

```csharp
// 중복 요청 방지
[HttpPost]
public async Task<IActionResult> Create(
    [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey,
    CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    if (!string.IsNullOrEmpty(idempotencyKey))
    {
        var cached = await _cache.GetAsync<OrderDto>(
            $"idempotency:{idempotencyKey}", cancellationToken);

        if (cached != null)
        {
            return Ok(cached);  // 이전 응답 반환
        }
    }

    var order = await _orderService.CreateAsync(request, cancellationToken);
    var dto = order.ToDto();

    if (!string.IsNullOrEmpty(idempotencyKey))
    {
        await _cache.SetAsync(
            $"idempotency:{idempotencyKey}",
            dto,
            TimeSpan.FromHours(24),
            cancellationToken);
    }

    return CreatedAtAction(nameof(GetById), new { id = order.Id }, dto);
}
```

---

## 안티패턴 정리

```
┌─────────────────────────────────────────────────────────────────┐
│                    API 설계 안티패턴                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  URL:                                                           │
│  • 동사 사용 (/getUsers, /createOrder)                         │
│  • 단수형 리소스 (/user/123)                                   │
│  • 액션 표현 (/users/123/delete)                               │
│                                                                 │
│  응답:                                                          │
│  • 일관성 없는 응답 구조                                        │
│  • 부적절한 상태 코드 (항상 200)                                │
│  • 민감 정보 노출 (password 등)                                 │
│                                                                 │
│  요청:                                                          │
│  • Entity를 직접 받음 (DTO 없이)                               │
│  • 입력 검증 누락                                               │
│  • 너무 많은 쿼리 파라미터                                      │
│                                                                 │
│  버전:                                                          │
│  • 버전 관리 없음                                               │
│  • 하위 호환성 무시한 변경                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 면접 예상 질문

### Q1: RESTful API에서 동사를 사용하면 안 되는 이유는?

**A:** REST는 리소스 중심 아키텍처입니다. HTTP 메서드(GET, POST, PUT, DELETE)가 동작을 표현하므로 URL에는 리소스(명사)만 포함해야 합니다. `/getUsers` 대신 `GET /users`를 사용합니다.

### Q2: PUT과 PATCH의 차이점은?

**A:** PUT은 전체 리소스를 교체(전송하지 않은 필드는 null로), PATCH는 부분 수정(전송한 필드만 변경)입니다. PUT은 멱등하지만 PATCH는 그렇지 않을 수 있습니다.

### Q3: 페이지네이션 방식의 선택 기준은?

**A:**
- **오프셋 기반**: 랜덤 페이지 접근 필요, 데이터 적음
- **커서 기반**: 대용량 데이터, 실시간 변경이 많음, 무한 스크롤

---

## 참고 자료

- [REST API 설계 가이드](https://restfulapi.net/)
- [Microsoft REST API 가이드라인](https://github.com/microsoft/api-guidelines)
- [RFC 7807 - Problem Details](https://datatracker.ietf.org/doc/html/rfc7807)

---

## 다음 섹션

→ [01. Fundamentals](../01-fundamentals/) - ASP.NET Core 기초
