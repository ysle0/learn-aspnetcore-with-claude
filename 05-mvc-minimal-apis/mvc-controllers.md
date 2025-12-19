# MVC 컨트롤러 패턴

## 개요

MVC (Model-View-Controller) 컨트롤러는 ASP.NET Core의 전통적인 API 개발 방식입니다. 구조화된 코드와 풍부한 기능을 제공합니다.

---

## 기본 구조

### ApiController 속성

```csharp
[ApiController]  // API 동작 활성화
[Route("api/[controller]")]  // 라우팅
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
}
```

### [ApiController]의 기능

```csharp
// 1. 자동 모델 바인딩 소스 추론
[HttpPost]
public IActionResult Create(CreateUserRequest request)
{
    // [FromBody]가 자동으로 추론됨
}

// 2. 자동 400 응답 (ModelState 검증 실패 시)
// ModelState.IsValid 체크 불필요

// 3. 문제 세부 정보 응답 (RFC 7807)
// BadRequest()가 자동으로 Problem Details 형식으로 변환

// 4. 바인딩 소스 추론 규칙:
// - 복합 타입 → [FromBody]
// - IFormFile → [FromForm]
// - 라우트에 있는 파라미터 → [FromRoute]
// - 그 외 → [FromQuery]
```

---

## 라우팅

### 속성 라우팅

```csharp
[ApiController]
[Route("api/[controller]")]  // api/users
public class UsersController : ControllerBase
{
    [HttpGet]                    // GET api/users
    [HttpGet("{id}")]            // GET api/users/5
    [HttpGet("search")]          // GET api/users/search
    [HttpPost]                   // POST api/users
    [HttpPut("{id}")]            // PUT api/users/5
    [HttpDelete("{id}")]         // DELETE api/users/5

    // 경로 결합
    [HttpGet("orders/{orderId}")]  // GET api/users/orders/123

    // 라우트 제약
    [HttpGet("{id:int:min(1)}")]   // 양의 정수만
}

// 명시적 라우트
[Route("api/v1/[controller]")]  // api/v1/users
public class UsersController : ControllerBase
{
    [Route("")]           // api/v1/users
    [Route("{id}")]       // api/v1/users/5
    [Route("[action]")]   // api/v1/users/getall
}
```

### 라우트 토큰

```csharp
[Route("api/[controller]/[action]")]
public class ProductsController : ControllerBase
{
    public IActionResult GetActive()
    {
        // api/products/getactive
    }

    public IActionResult GetInactive()
    {
        // api/products/getinactive
    }
}
```

---

## 모델 바인딩

### 바인딩 소스

```csharp
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(
        [FromRoute] int id,
        [FromQuery] string? include,
        [FromHeader(Name = "X-Correlation-ID")] string? correlationId)
    {
        // /api/orders/5?include=items
    }

    [HttpPost]
    public IActionResult Create(
        [FromBody] CreateOrderRequest request,
        [FromServices] IOrderService orderService)
    {
        return Ok(orderService.Create(request));
    }

    [HttpPost("upload")]
    public IActionResult Upload(
        [FromForm] IFormFile file,
        [FromForm] string description)
    {
        return Ok();
    }
}
```

### 복합 모델 바인딩

```csharp
public class SearchQuery
{
    [FromQuery(Name = "q")]
    public string? Query { get; set; }

    [FromQuery]
    public int Page { get; set; } = 1;

    [FromQuery]
    public int PageSize { get; set; } = 10;

    [FromHeader(Name = "Accept-Language")]
    public string? Language { get; set; }
}

[HttpGet("search")]
public IActionResult Search([FromQuery] SearchQuery query)
{
    // 모든 속성이 적절한 소스에서 바인딩됨
    return Ok();
}
```

---

## 응답 처리

### ActionResult<T>

```csharp
// ActionResult<T>: 타입 정보 보존 + 다양한 응답 가능
[HttpGet("{id}")]
public async Task<ActionResult<User>> GetById(int id)
{
    var user = await _userService.GetByIdAsync(id);

    if (user == null)
        return NotFound();  // ActionResult

    return user;  // T (암시적 변환)
}

// 헬퍼 메서드들
return Ok(value);                    // 200
return Created(uri, value);          // 201
return CreatedAtAction(actionName, routeValues, value);  // 201
return NoContent();                  // 204
return BadRequest(modelState);       // 400
return Unauthorized();               // 401
return Forbid();                     // 403
return NotFound();                   // 404
return Conflict();                   // 409
return UnprocessableEntity(errors);  // 422
return StatusCode(500, error);       // 커스텀 상태
```

### ProducesResponseType

```csharp
[HttpGet("{id}")]
[ProducesResponseType(typeof(User), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
[ProducesResponseType(StatusCodes.Status500InternalServerError)]
public async Task<ActionResult<User>> GetById(int id)
{
    // OpenAPI 문서에 응답 타입 표시
}

// 또는 클래스 레벨에서
[Produces("application/json")]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
[ApiController]
public class UsersController : ControllerBase
{
}
```

### 파일 다운로드

```csharp
[HttpGet("download/{filename}")]
public IActionResult Download(string filename)
{
    var bytes = System.IO.File.ReadAllBytes($"files/{filename}");

    return File(bytes, "application/octet-stream", filename);

    // 또는
    return PhysicalFile($"files/{filename}", "application/pdf");

    // 스트림으로
    var stream = new FileStream($"files/{filename}", FileMode.Open);
    return File(stream, "application/octet-stream");
}
```

---

## 모델 검증

### DataAnnotations

```csharp
public class CreateUserRequest
{
    [Required(ErrorMessage = "이름은 필수입니다")]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = "";

    [Required]
    [EmailAddress]
    public string Email { get; set; } = "";

    [Range(1, 150)]
    public int Age { get; set; }

    [RegularExpression(@"^\d{3}-\d{4}-\d{4}$")]
    public string? Phone { get; set; }

    [Compare("Password")]
    public string ConfirmPassword { get; set; } = "";
}

// [ApiController]가 있으면 자동 검증
// ModelState.IsValid가 false면 자동으로 400 반환
```

### 수동 검증

```csharp
// [ApiController] 없이 사용할 때
[HttpPost]
public IActionResult Create(CreateUserRequest request)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }

    // 또는 커스텀 검증
    if (request.Age < 18)
    {
        ModelState.AddModelError(nameof(request.Age), "Must be 18 or older");
        return BadRequest(ModelState);
    }

    return Ok();
}
```

### 커스텀 검증 속성

```csharp
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        if (value is DateTime date && date <= DateTime.Today)
        {
            return new ValidationResult("Date must be in the future");
        }
        return ValidationResult.Success;
    }
}

public class Event
{
    [FutureDate]
    public DateTime EventDate { get; set; }
}
```

---

## 비동기 처리

### async/await

```csharp
[HttpGet]
public async Task<ActionResult<IEnumerable<User>>> GetAll(
    CancellationToken cancellationToken)
{
    var users = await _userService.GetAllAsync(cancellationToken);
    return Ok(users);
}

[HttpPost]
public async Task<ActionResult<User>> Create(
    CreateUserRequest request,
    CancellationToken cancellationToken)
{
    var user = await _userService.CreateAsync(request, cancellationToken);
    return CreatedAtAction(
        nameof(GetById),
        new { id = user.Id },
        user);
}
```

### IAsyncEnumerable

```csharp
[HttpGet("stream")]
public async IAsyncEnumerable<int> StreamData(
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        yield return i;
        await Task.Delay(100, cancellationToken);
    }
}
```

---

## 의존성 주입

### 생성자 주입

```csharp
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(
        IOrderService orderService,
        ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
}

// Primary Constructor (C# 12+)
public class OrdersController(
    IOrderService orderService,
    ILogger<OrdersController> logger) : ControllerBase
{
}
```

### 액션 메서드 주입

```csharp
[HttpGet]
public IActionResult Get([FromServices] ISpecialService specialService)
{
    // 특정 액션에서만 필요한 서비스
    return Ok(specialService.GetData());
}
```

---

## 면접 예상 질문

### Q1: ControllerBase와 Controller의 차이는?

**A:**
- **ControllerBase**: API 전용, 뷰 관련 기능 없음
- **Controller**: ControllerBase 상속 + View, PartialView, Json, ViewData, TempData 등 뷰 관련 기능 포함

### Q2: [ApiController]가 제공하는 기능은?

**A:**
1. 자동 모델 검증 및 400 응답
2. 바인딩 소스 자동 추론 ([FromBody], [FromQuery] 등)
3. Problem Details 형식 오류 응답
4. 속성 라우팅 필수

### Q3: ActionResult<T>의 장점은?

**A:** 반환 타입 정보를 보존하면서도 NotFound, BadRequest 같은 다른 상태 코드를 반환할 수 있습니다. OpenAPI 문서 생성에도 유용합니다.

---

## 참고 자료

- [ASP.NET Core MVC 개요](https://learn.microsoft.com/aspnet/core/mvc/overview)
- [컨트롤러 액션](https://learn.microsoft.com/aspnet/core/mvc/controllers/actions)
- [모델 바인딩](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding)

---

## 다음 문서

→ [model-binding.md](./model-binding.md) - 모델 바인딩 상세
