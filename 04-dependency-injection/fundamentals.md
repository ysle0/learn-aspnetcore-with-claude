# DI 기초

## 개요

의존성 주입(Dependency Injection, DI)은 ASP.NET Core의 핵심 기능입니다. 모든 프레임워크 서비스와 사용자 서비스가 DI 컨테이너를 통해 관리됩니다.

---

## 의존성 주입이 필요한 이유

### Before: 강한 결합

```csharp
public class OrderService
{
    // 문제점:
    // 1. SqlRepository에 직접 의존
    // 2. 테스트할 때 실제 DB 필요
    // 3. 다른 Repository로 교체 어려움
    private readonly SqlRepository _repository = new SqlRepository();
    private readonly SmtpEmailService _email = new SmtpEmailService();

    public void CreateOrder(Order order)
    {
        _repository.Save(order);
        _email.SendConfirmation(order);
    }
}
```

### After: 느슨한 결합

```csharp
public class OrderService
{
    private readonly IRepository _repository;
    private readonly IEmailService _email;

    // 생성자 주입: 의존성을 외부에서 받음
    public OrderService(IRepository repository, IEmailService email)
    {
        _repository = repository;
        _email = email;
    }

    public void CreateOrder(Order order)
    {
        _repository.Save(order);
        _email.SendConfirmation(order);
    }
}

// 테스트에서 Mock 주입 가능
var mockRepo = new Mock<IRepository>();
var mockEmail = new Mock<IEmailService>();
var service = new OrderService(mockRepo.Object, mockEmail.Object);
```

---

## 서비스 등록

### 기본 등록 방법

```csharp
var builder = WebApplication.CreateBuilder(args);
var services = builder.Services;

// 1. 인터페이스 → 구현 타입 매핑
services.AddTransient<IEmailService, EmailService>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddSingleton<ICacheService, MemoryCacheService>();

// 2. 구현 타입만 등록 (인터페이스 없이)
services.AddScoped<OrderService>();

// 3. 인스턴스 직접 등록
services.AddSingleton<ICacheService>(new MemoryCacheService());

// 4. 팩토리 함수로 등록
services.AddScoped<IOrderService>(sp =>
{
    var repo = sp.GetRequiredService<IOrderRepository>();
    var logger = sp.GetRequiredService<ILogger<OrderService>>();
    return new OrderService(repo, logger);
});
```

### TryAdd 메서드

```csharp
// 이미 등록되어 있으면 건너뜀 (중복 방지)
services.TryAddTransient<IEmailService, EmailService>();
services.TryAddScoped<IOrderRepository, OrderRepository>();
services.TryAddSingleton<ICacheService, MemoryCacheService>();

// 라이브러리에서 유용 - 사용자가 이미 등록했을 수 있음
public static class MyLibraryExtensions
{
    public static IServiceCollection AddMyLibrary(this IServiceCollection services)
    {
        services.TryAddSingleton<IMyService, DefaultMyService>();
        return services;
    }
}
```

### Replace 메서드

```csharp
// 기존 등록을 교체
services.AddSingleton<IEmailService, ProductionEmailService>();

// 나중에 교체 (테스트에서 유용)
services.Replace(ServiceDescriptor.Singleton<IEmailService, MockEmailService>());
```

---

## 서비스 주입 방법

### 1. 생성자 주입 (권장)

```csharp
public class OrderController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrderController> _logger;

    // 생성자에서 모든 의존성 선언
    public OrderController(
        IOrderService orderService,
        ILogger<OrderController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpGet]
    public IActionResult GetOrders()
    {
        _logger.LogInformation("Getting orders");
        return Ok(_orderService.GetAll());
    }
}
```

### 2. Minimal API 파라미터 주입

```csharp
// 파라미터로 직접 주입
app.MapGet("/orders", (IOrderService orderService) =>
{
    return orderService.GetAll();
});

// 여러 서비스 주입
app.MapPost("/orders", (
    CreateOrderRequest request,
    IOrderService orderService,
    IValidator<CreateOrderRequest> validator,
    ILogger<Program> logger) =>
{
    var validation = validator.Validate(request);
    if (!validation.IsValid)
        return Results.BadRequest(validation.Errors);

    var order = orderService.Create(request);
    logger.LogInformation("Order {OrderId} created", order.Id);
    return Results.Created($"/orders/{order.Id}", order);
});
```

### 3. [FromServices] 속성

```csharp
[ApiController]
[Route("[controller]")]
public class OrderController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetById(
        int id,
        [FromServices] IOrderService orderService)
    {
        // 특정 액션에서만 필요한 서비스
        return Ok(orderService.GetById(id));
    }
}
```

### 4. HttpContext.RequestServices

```csharp
// 미들웨어나 특수한 상황에서 사용
app.Use(async (context, next) =>
{
    var service = context.RequestServices.GetRequiredService<IOrderService>();
    // ...
    await next(context);
});

// 컨트롤러에서도 가능 (권장하지 않음)
public class OrderController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        var service = HttpContext.RequestServices.GetRequiredService<IOrderService>();
        return Ok(service.GetAll());
    }
}
```

### 5. Primary Constructor (C# 12+)

```csharp
public class OrderController(
    IOrderService orderService,
    ILogger<OrderController> logger) : ControllerBase
{
    [HttpGet]
    public IActionResult GetOrders()
    {
        logger.LogInformation("Getting orders");
        return Ok(orderService.GetAll());
    }
}
```

---

## 다중 구현 등록

### 같은 인터페이스에 여러 구현

```csharp
// 여러 구현 등록
services.AddTransient<INotificationService, EmailNotificationService>();
services.AddTransient<INotificationService, SmsNotificationService>();
services.AddTransient<INotificationService, PushNotificationService>();

// 단일 주입 시 마지막 등록된 것이 주입됨
public class NotifyController(INotificationService service)
{
    // PushNotificationService가 주입됨
}

// 모든 구현 주입
public class NotifyController(IEnumerable<INotificationService> services)
{
    public void NotifyAll(string message)
    {
        foreach (var service in services)
        {
            service.Send(message);
        }
    }
}
```

### Keyed Services (.NET 8+)

```csharp
// 키로 구분하여 등록
services.AddKeyedTransient<INotificationService, EmailNotificationService>("email");
services.AddKeyedTransient<INotificationService, SmsNotificationService>("sms");
services.AddKeyedTransient<INotificationService, PushNotificationService>("push");

// 키로 주입
public class NotifyController(
    [FromKeyedServices("email")] INotificationService emailService,
    [FromKeyedServices("sms")] INotificationService smsService)
{
    public void SendEmail(string message) => emailService.Send(message);
    public void SendSms(string message) => smsService.Send(message);
}

// Minimal API에서
app.MapPost("/notify/email", (
    [FromKeyedServices("email")] INotificationService service,
    string message) =>
{
    service.Send(message);
    return Results.Ok();
});
```

---

## 서비스 해결

### GetService vs GetRequiredService

```csharp
// GetService: 없으면 null 반환
var service = serviceProvider.GetService<IMyService>();
if (service != null)
{
    service.DoSomething();
}

// GetRequiredService: 없으면 예외 발생 (권장)
var service = serviceProvider.GetRequiredService<IMyService>();
service.DoSomething();

// 차이점
try
{
    var required = provider.GetRequiredService<IMissingService>();
}
catch (InvalidOperationException ex)
{
    // "No service for type 'IMissingService' has been registered."
}
```

### 여러 서비스 해결

```csharp
// 모든 등록된 구현 가져오기
var allServices = serviceProvider.GetServices<INotificationService>();

// Keyed 서비스 해결
var emailService = serviceProvider.GetRequiredKeyedService<INotificationService>("email");
```

---

## Options 패턴

### 설정 클래스 정의

```csharp
public class EmailOptions
{
    public const string SectionName = "Email";

    public string SmtpServer { get; set; } = "";
    public int Port { get; set; } = 587;
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
    public bool UseSsl { get; set; } = true;
}
```

### appsettings.json

```json
{
  "Email": {
    "SmtpServer": "smtp.example.com",
    "Port": 587,
    "Username": "user@example.com",
    "Password": "secret",
    "UseSsl": true
  }
}
```

### 등록 및 사용

```csharp
// 등록
builder.Services.Configure<EmailOptions>(
    builder.Configuration.GetSection(EmailOptions.SectionName));

// 사용 - IOptions<T>
public class EmailService
{
    private readonly EmailOptions _options;

    public EmailService(IOptions<EmailOptions> options)
    {
        _options = options.Value;
    }

    public void Send(string to, string message)
    {
        Console.WriteLine($"Sending via {_options.SmtpServer}:{_options.Port}");
    }
}

// IOptionsSnapshot<T> - Scoped, 설정 변경 시 새 값 반영
public class EmailService(IOptionsSnapshot<EmailOptions> options)
{
    public void Send() => Console.WriteLine(options.Value.SmtpServer);
}

// IOptionsMonitor<T> - Singleton에서 사용, 변경 감지
public class EmailService : IDisposable
{
    private readonly IOptionsMonitor<EmailOptions> _optionsMonitor;
    private readonly IDisposable? _changeListener;

    public EmailService(IOptionsMonitor<EmailOptions> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
        _changeListener = optionsMonitor.OnChange(OnOptionsChanged);
    }

    private void OnOptionsChanged(EmailOptions options)
    {
        Console.WriteLine($"Options changed: {options.SmtpServer}");
    }

    public void Dispose() => _changeListener?.Dispose();
}
```

### Options 검증

```csharp
// 데이터 어노테이션으로 검증
public class EmailOptions
{
    [Required]
    public string SmtpServer { get; set; } = "";

    [Range(1, 65535)]
    public int Port { get; set; } = 587;

    [EmailAddress]
    public string Username { get; set; } = "";
}

// 등록 시 검증 활성화
builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // 앱 시작 시 검증

// 커스텀 검증
builder.Services.AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .Validate(options =>
    {
        return !string.IsNullOrEmpty(options.SmtpServer) && options.Port > 0;
    }, "Invalid email configuration")
    .ValidateOnStart();
```

---

## 서비스 확장 메서드 패턴

### 확장 메서드 작성

```csharp
public static class OrderServiceExtensions
{
    public static IServiceCollection AddOrderServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // 옵션 등록
        services.Configure<OrderOptions>(
            configuration.GetSection("Order"));

        // 서비스 등록
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddTransient<IOrderValidator, OrderValidator>();

        return services;
    }
}

// Program.cs에서 사용
builder.Services.AddOrderServices(builder.Configuration);
```

### 조건부 등록

```csharp
public static IServiceCollection AddEmailService(
    this IServiceCollection services,
    IConfiguration configuration)
{
    var emailConfig = configuration.GetSection("Email");

    if (emailConfig.GetValue<bool>("UseMock"))
    {
        services.AddSingleton<IEmailService, MockEmailService>();
    }
    else
    {
        services.AddSingleton<IEmailService, SmtpEmailService>();
    }

    return services;
}
```

---

## 면접 예상 질문

### Q1: 생성자 주입을 권장하는 이유는?

**A:**
1. **명시적 의존성**: 필요한 의존성이 생성자에 명확히 드러남
2. **불변성**: 의존성이 readonly로 주입되어 변경 불가
3. **테스트 용이**: Mock 객체를 쉽게 전달 가능
4. **컴파일 타임 검증**: 의존성 누락 시 컴파일 오류

### Q2: IOptions vs IOptionsSnapshot vs IOptionsMonitor 차이는?

**A:**
- **IOptions<T>**: Singleton, 앱 시작 시 값 고정
- **IOptionsSnapshot<T>**: Scoped, 요청마다 최신 값 (런타임 변경 반영)
- **IOptionsMonitor<T>**: Singleton이지만 변경 감지 및 콜백 지원

### Q3: TryAddXxx를 사용하는 이유는?

**A:** 라이브러리 개발 시 사용자가 이미 같은 서비스를 등록했을 수 있습니다. TryAdd는 기존 등록이 없을 때만 등록하여 사용자의 설정을 존중합니다.

### Q4: Keyed Services의 용도는?

**A:** 같은 인터페이스의 여러 구현 중 특정 구현을 키로 선택하여 주입할 때 사용합니다. 전략 패턴, 다중 데이터베이스 연결 등에 유용합니다.

---

## 참고 자료

- [ASP.NET Core에서 의존성 주입](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
- [옵션 패턴](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options)
- [Keyed Services](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#keyed-services)

---

## 다음 문서

→ [lifetimes.md](./lifetimes.md) - 서비스 생명주기 상세
