# 고급 DI 패턴

## 개요

이 문서에서는 실제 프로젝트에서 자주 사용되는 고급 의존성 주입 패턴들을 다룹니다.

---

## 팩토리 패턴

### 기본 팩토리

```csharp
public interface IReportGeneratorFactory
{
    IReportGenerator Create(ReportType type);
}

public class ReportGeneratorFactory : IReportGeneratorFactory
{
    private readonly IServiceProvider _sp;

    public ReportGeneratorFactory(IServiceProvider sp)
    {
        _sp = sp;
    }

    public IReportGenerator Create(ReportType type)
    {
        return type switch
        {
            ReportType.Pdf => _sp.GetRequiredService<PdfReportGenerator>(),
            ReportType.Excel => _sp.GetRequiredService<ExcelReportGenerator>(),
            ReportType.Html => _sp.GetRequiredService<HtmlReportGenerator>(),
            _ => throw new ArgumentException($"Unknown report type: {type}")
        };
    }
}

// 등록
services.AddTransient<PdfReportGenerator>();
services.AddTransient<ExcelReportGenerator>();
services.AddTransient<HtmlReportGenerator>();
services.AddSingleton<IReportGeneratorFactory, ReportGeneratorFactory>();
```

### Func<T> 팩토리

```csharp
// 등록
services.AddTransient<IOrderProcessor, OrderProcessor>();
services.AddSingleton<Func<IOrderProcessor>>(sp =>
    () => sp.CreateScope().ServiceProvider.GetRequiredService<IOrderProcessor>());

// 사용
public class OrderBatchService
{
    private readonly Func<IOrderProcessor> _processorFactory;

    public OrderBatchService(Func<IOrderProcessor> processorFactory)
    {
        _processorFactory = processorFactory;
    }

    public void ProcessBatch(IEnumerable<Order> orders)
    {
        foreach (var order in orders)
        {
            var processor = _processorFactory();  // 매번 새 인스턴스
            processor.Process(order);
        }
    }
}
```

### ObjectFactory 활용

```csharp
// ActivatorUtilities를 사용한 팩토리
services.AddSingleton(sp =>
{
    var factory = ActivatorUtilities.CreateFactory(
        typeof(OrderProcessor),
        new[] { typeof(Order) });  // 런타임 파라미터 타입

    return (Order order) =>
    {
        return (IOrderProcessor)factory(sp, new object[] { order });
    };
});

// 사용
public class OrderService
{
    private readonly Func<Order, IOrderProcessor> _factory;

    public OrderService(Func<Order, IOrderProcessor> factory)
    {
        _factory = factory;
    }

    public void Process(Order order)
    {
        var processor = _factory(order);  // order를 생성자에 전달
        processor.Execute();
    }
}
```

---

## 데코레이터 패턴

### 수동 데코레이터

```csharp
public interface IOrderService
{
    Task<Order> GetOrderAsync(int id);
}

public class OrderService : IOrderService
{
    public Task<Order> GetOrderAsync(int id) => /* 실제 구현 */;
}

public class CachingOrderServiceDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly IMemoryCache _cache;

    public CachingOrderServiceDecorator(
        IOrderService inner,
        IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        var key = $"order_{id}";

        if (_cache.TryGetValue(key, out Order? order))
            return order!;

        order = await _inner.GetOrderAsync(id);
        _cache.Set(key, order, TimeSpan.FromMinutes(5));

        return order;
    }
}

public class LoggingOrderServiceDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger<LoggingOrderServiceDecorator> _logger;

    public LoggingOrderServiceDecorator(
        IOrderService inner,
        ILogger<LoggingOrderServiceDecorator> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        _logger.LogInformation("Getting order {OrderId}", id);
        var order = await _inner.GetOrderAsync(id);
        _logger.LogInformation("Got order {OrderId}", id);
        return order;
    }
}

// 등록 (내부에서 외부로)
services.AddScoped<OrderService>();
services.AddScoped<IOrderService>(sp =>
{
    var inner = sp.GetRequiredService<OrderService>();
    var cache = sp.GetRequiredService<IMemoryCache>();
    var logger = sp.GetRequiredService<ILogger<LoggingOrderServiceDecorator>>();

    var cached = new CachingOrderServiceDecorator(inner, cache);
    var logged = new LoggingOrderServiceDecorator(cached, logger);

    return logged;
});

// 호출 순서: Logging → Caching → OrderService
```

### Scrutor 라이브러리 활용

```csharp
// NuGet: Scrutor
services.AddScoped<IOrderService, OrderService>();

services.Decorate<IOrderService, CachingOrderServiceDecorator>();
services.Decorate<IOrderService, LoggingOrderServiceDecorator>();

// 더 간결하게 데코레이터 체인 구성
```

---

## 전략 패턴

### 키 기반 전략 선택

```csharp
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessAsync(Payment payment);
}

public class CreditCardProcessor : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(Payment payment) => /* ... */;
}

public class PayPalProcessor : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(Payment payment) => /* ... */;
}

public class BankTransferProcessor : IPaymentProcessor
{
    public Task<PaymentResult> ProcessAsync(Payment payment) => /* ... */;
}

// Keyed Services (.NET 8+)
services.AddKeyedScoped<IPaymentProcessor, CreditCardProcessor>("credit-card");
services.AddKeyedScoped<IPaymentProcessor, PayPalProcessor>("paypal");
services.AddKeyedScoped<IPaymentProcessor, BankTransferProcessor>("bank-transfer");

// 전략 선택자
public interface IPaymentProcessorSelector
{
    IPaymentProcessor Select(PaymentMethod method);
}

public class PaymentProcessorSelector : IPaymentProcessorSelector
{
    private readonly IServiceProvider _sp;

    public PaymentProcessorSelector(IServiceProvider sp) => _sp = sp;

    public IPaymentProcessor Select(PaymentMethod method)
    {
        var key = method switch
        {
            PaymentMethod.CreditCard => "credit-card",
            PaymentMethod.PayPal => "paypal",
            PaymentMethod.BankTransfer => "bank-transfer",
            _ => throw new ArgumentException($"Unknown payment method: {method}")
        };

        return _sp.GetRequiredKeyedService<IPaymentProcessor>(key);
    }
}
```

### IEnumerable 기반 전략

```csharp
public interface IExportStrategy
{
    string Format { get; }
    byte[] Export(IEnumerable<Order> orders);
}

public class CsvExportStrategy : IExportStrategy
{
    public string Format => "csv";
    public byte[] Export(IEnumerable<Order> orders) => /* ... */;
}

public class JsonExportStrategy : IExportStrategy
{
    public string Format => "json";
    public byte[] Export(IEnumerable<Order> orders) => /* ... */;
}

public class XmlExportStrategy : IExportStrategy
{
    public string Format => "xml";
    public byte[] Export(IEnumerable<Order> orders) => /* ... */;
}

// 모든 전략 등록
services.AddTransient<IExportStrategy, CsvExportStrategy>();
services.AddTransient<IExportStrategy, JsonExportStrategy>();
services.AddTransient<IExportStrategy, XmlExportStrategy>();

// 전략 선택자
public class ExportService
{
    private readonly IEnumerable<IExportStrategy> _strategies;

    public ExportService(IEnumerable<IExportStrategy> strategies)
    {
        _strategies = strategies;
    }

    public byte[] Export(IEnumerable<Order> orders, string format)
    {
        var strategy = _strategies.FirstOrDefault(s =>
            s.Format.Equals(format, StringComparison.OrdinalIgnoreCase))
            ?? throw new ArgumentException($"Unknown format: {format}");

        return strategy.Export(orders);
    }

    public IEnumerable<string> GetSupportedFormats()
        => _strategies.Select(s => s.Format);
}
```

---

## Composite 패턴

### 다중 구현 조합

```csharp
public interface INotificationSender
{
    Task SendAsync(Notification notification);
}

public class EmailNotificationSender : INotificationSender
{
    public Task SendAsync(Notification notification) => /* 이메일 전송 */;
}

public class SmsNotificationSender : INotificationSender
{
    public Task SendAsync(Notification notification) => /* SMS 전송 */;
}

public class PushNotificationSender : INotificationSender
{
    public Task SendAsync(Notification notification) => /* 푸시 전송 */;
}

// Composite 구현
public class CompositeNotificationSender : INotificationSender
{
    private readonly IEnumerable<INotificationSender> _senders;
    private readonly ILogger<CompositeNotificationSender> _logger;

    public CompositeNotificationSender(
        IEnumerable<INotificationSender> senders,
        ILogger<CompositeNotificationSender> logger)
    {
        _senders = senders;
        _logger = logger;
    }

    public async Task SendAsync(Notification notification)
    {
        var tasks = _senders.Select(async sender =>
        {
            try
            {
                await sender.SendAsync(notification);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Sender {SenderType} failed",
                    sender.GetType().Name);
            }
        });

        await Task.WhenAll(tasks);
    }
}

// 등록
services.AddTransient<EmailNotificationSender>();
services.AddTransient<SmsNotificationSender>();
services.AddTransient<PushNotificationSender>();

// 내부 구현들
services.AddTransient<INotificationSender, EmailNotificationSender>();
services.AddTransient<INotificationSender, SmsNotificationSender>();
services.AddTransient<INotificationSender, PushNotificationSender>();

// Composite를 주 구현으로 등록
services.AddTransient<INotificationSender>(sp =>
{
    var senders = new INotificationSender[]
    {
        sp.GetRequiredService<EmailNotificationSender>(),
        sp.GetRequiredService<SmsNotificationSender>(),
        sp.GetRequiredService<PushNotificationSender>()
    };
    var logger = sp.GetRequiredService<ILogger<CompositeNotificationSender>>();
    return new CompositeNotificationSender(senders, logger);
});
```

---

## Lazy 초기화

### Lazy<T>

```csharp
// 비용이 큰 서비스의 지연 초기화
services.AddSingleton<ExpensiveService>();
services.AddSingleton(sp => new Lazy<ExpensiveService>(
    () => sp.GetRequiredService<ExpensiveService>()));

public class MyService
{
    private readonly Lazy<ExpensiveService> _expensiveService;

    public MyService(Lazy<ExpensiveService> expensiveService)
    {
        _expensiveService = expensiveService;
    }

    public void DoWork()
    {
        // 실제 필요할 때만 생성
        var service = _expensiveService.Value;
        service.Process();
    }
}
```

### LazyAsync<T>

```csharp
// 비동기 초기화가 필요한 경우
public class AsyncLazy<T>
{
    private readonly Lazy<Task<T>> _lazy;

    public AsyncLazy(Func<Task<T>> factory)
    {
        _lazy = new Lazy<Task<T>>(factory);
    }

    public Task<T> Value => _lazy.Value;
}

// 등록
services.AddSingleton(sp => new AsyncLazy<DatabaseConnection>(async () =>
{
    var connection = sp.GetRequiredService<DatabaseConnection>();
    await connection.InitializeAsync();
    return connection;
}));
```

---

## 조건부 등록

### 환경별 등록

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddConditionalService<TService, TImplementation>(
        this IServiceCollection services,
        IWebHostEnvironment env)
        where TService : class
        where TImplementation : class, TService
    {
        if (env.IsDevelopment())
        {
            services.AddScoped<TService, MockImplementation>();
        }
        else
        {
            services.AddScoped<TService, TImplementation>();
        }

        return services;
    }
}

// 사용
services.AddConditionalService<IPaymentService, StripePaymentService>(env);
```

### 설정 기반 등록

```csharp
// appsettings.json
// { "Email": { "Provider": "SendGrid" } }

public static class EmailServiceExtensions
{
    public static IServiceCollection AddEmailService(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var provider = configuration["Email:Provider"];

        switch (provider?.ToLower())
        {
            case "sendgrid":
                services.AddTransient<IEmailService, SendGridEmailService>();
                break;
            case "ses":
                services.AddTransient<IEmailService, AwsSesEmailService>();
                break;
            case "smtp":
                services.AddTransient<IEmailService, SmtpEmailService>();
                break;
            default:
                services.AddTransient<IEmailService, NullEmailService>();
                break;
        }

        return services;
    }
}
```

---

## 모듈 패턴

### 서비스 모듈

```csharp
public interface IServiceModule
{
    void ConfigureServices(IServiceCollection services, IConfiguration configuration);
}

public class OrderModule : IServiceModule
{
    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        services.Configure<OrderOptions>(configuration.GetSection("Order"));
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddTransient<IOrderValidator, OrderValidator>();
    }
}

public class PaymentModule : IServiceModule
{
    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        services.Configure<PaymentOptions>(configuration.GetSection("Payment"));
        services.AddScoped<IPaymentService, PaymentService>();
        services.AddTransient<IPaymentValidator, PaymentValidator>();
    }
}

// 확장 메서드
public static class ServiceModuleExtensions
{
    public static IServiceCollection AddModule<TModule>(
        this IServiceCollection services,
        IConfiguration configuration)
        where TModule : IServiceModule, new()
    {
        var module = new TModule();
        module.ConfigureServices(services, configuration);
        return services;
    }

    public static IServiceCollection AddModulesFromAssembly(
        this IServiceCollection services,
        Assembly assembly,
        IConfiguration configuration)
    {
        var moduleTypes = assembly.GetTypes()
            .Where(t => typeof(IServiceModule).IsAssignableFrom(t) && !t.IsAbstract);

        foreach (var moduleType in moduleTypes)
        {
            var module = (IServiceModule)Activator.CreateInstance(moduleType)!;
            module.ConfigureServices(services, configuration);
        }

        return services;
    }
}

// Program.cs
builder.Services.AddModule<OrderModule>(builder.Configuration);
builder.Services.AddModule<PaymentModule>(builder.Configuration);

// 또는 어셈블리에서 자동 탐색
builder.Services.AddModulesFromAssembly(typeof(Program).Assembly, builder.Configuration);
```

---

## 서비스 재정의

### 테스트에서 서비스 교체

```csharp
// 통합 테스트
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                // 실제 서비스를 Mock으로 교체
                services.RemoveAll<IPaymentService>();
                services.AddScoped<IPaymentService, MockPaymentService>();

                // 또는 Replace 사용
                services.Replace(ServiceDescriptor.Scoped<IEmailService, MockEmailService>());
            });
        }).CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ReturnsSuccess()
    {
        var response = await _client.PostAsJsonAsync("/api/orders", new { });
        response.EnsureSuccessStatusCode();
    }
}
```

---

## 면접 예상 질문

### Q1: 데코레이터 패턴을 DI에서 어떻게 구현하나요?

**A:** 팩토리 함수를 사용하여 내부 서비스를 먼저 해결한 후, 그것을 감싸는 데코레이터를 반환합니다. Scrutor 라이브러리를 사용하면 `Decorate<TService, TDecorator>()` 메서드로 간단하게 구성할 수 있습니다.

### Q2: 전략 패턴을 DI에서 구현하는 방법은?

**A:** 두 가지 방법:
1. **Keyed Services (.NET 8+)**: 각 전략을 키로 등록하고 `GetRequiredKeyedService`로 해결
2. **IEnumerable<T>**: 모든 전략을 등록하고 런타임에 조건으로 선택

### Q3: Lazy<T>를 사용하는 이유는?

**A:** 비용이 큰 서비스(외부 연결, 대용량 데이터 로드 등)의 초기화를 실제로 필요할 때까지 지연시킵니다. 앱 시작 시간을 단축하고, 사용되지 않는 서비스의 초기화 비용을 절약합니다.

### Q4: 모듈 패턴의 장점은?

**A:**
- **관심사 분리**: 도메인별로 서비스 등록 로직 분리
- **테스트 용이**: 모듈 단위로 테스트 가능
- **재사용성**: 모듈을 다른 프로젝트에서 재사용
- **유지보수성**: 관련 서비스를 한 곳에서 관리

---

## 참고 자료

- [DI 지침](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- [고급 DI 시나리오](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#advanced-dependency-injection-scenarios)
- [Scrutor 라이브러리](https://github.com/khellang/Scrutor)

---

## 다음 섹션

→ [05. MVC & Minimal APIs](../05-mvc-minimal-apis/) - 웹 API 개발
