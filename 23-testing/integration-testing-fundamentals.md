# Integration Testing 기초

ASP.NET Core의 `WebApplicationFactory`와 `TestServer`를 사용한 통합 테스트를 다룹니다.

## 통합 테스트란?

통합 테스트는 여러 컴포넌트가 함께 올바르게 동작하는지 검증합니다.

### 단위 테스트 vs 통합 테스트

| 특성 | 단위 테스트 | 통합 테스트 |
|------|-----------|-----------|
| 범위 | 단일 컴포넌트 | 여러 컴포넌트 |
| 의존성 | Mock/Fake | 실제 또는 테스트 구현 |
| 속도 | 매우 빠름 | 상대적으로 느림 |
| 신뢰도 | 격리된 동작만 | 통합된 동작 검증 |
| 설정 | 간단 | 복잡 |

---

## WebApplicationFactory

`Microsoft.AspNetCore.Mvc.Testing` 패키지가 제공하는 테스트 서버 팩토리입니다.

### 설치

```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
```

### 프로젝트 설정

```xml
<!-- 테스트 프로젝트.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <!-- 테스트 대상 프로젝트 참조 -->
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.4" />
  </ItemGroup>
</Project>
```

### Program.cs 접근 설정

```csharp
// Program.cs (Minimal API)
var builder = WebApplication.CreateBuilder(args);
// ... 설정
var app = builder.Build();
// ... 미들웨어
app.Run();

// 테스트에서 접근 가능하도록 partial class 선언
public partial class Program { }
```

또는 `InternalsVisibleTo` 사용:

```xml
<!-- MyApp.csproj -->
<ItemGroup>
  <InternalsVisibleTo Include="MyApp.Tests" />
</ItemGroup>
```

---

## 기본 통합 테스트

### 가장 간단한 형태

```csharp
using Microsoft.AspNetCore.Mvc.Testing;

public class BasicTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public BasicTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task Get_EndpointReturnsSuccess()
    {
        // Arrange
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync("/");

        // Assert
        response.EnsureSuccessStatusCode();
    }

    [Theory]
    [InlineData("/")]
    [InlineData("/about")]
    [InlineData("/contact")]
    public async Task Get_Pages_ReturnSuccess(string url)
    {
        var client = _factory.CreateClient();

        var response = await client.GetAsync(url);

        response.EnsureSuccessStatusCode();
        Assert.Equal("text/html; charset=utf-8",
            response.Content.Headers.ContentType?.ToString());
    }
}
```

---

## Custom WebApplicationFactory

테스트 환경을 커스터마이징합니다.

### 기본 구조

```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 서비스 구성 변경
        });

        builder.UseEnvironment("Testing");
    }
}
```

### 데이터베이스 교체 (In-Memory)

```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 기존 DbContext 제거
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));

            if (descriptor != null)
                services.Remove(descriptor);

            // In-Memory Database 추가
            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // 테스트 데이터 시딩
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
            SeedTestData(db);
        });
    }

    private static void SeedTestData(AppDbContext db)
    {
        db.Users.AddRange(
            new User { Id = 1, Name = "Test User 1" },
            new User { Id = 2, Name = "Test User 2" }
        );
        db.SaveChanges();
    }
}
```

### 외부 서비스 Mock 교체

```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    public Mock<IEmailService> MockEmailService { get; } = new();
    public Mock<IPaymentGateway> MockPaymentGateway { get; } = new();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 외부 서비스를 Mock으로 교체
            services.RemoveAll<IEmailService>();
            services.AddSingleton(MockEmailService.Object);

            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton(MockPaymentGateway.Object);
        });
    }
}
```

### 사용 예시

```csharp
public class OrderTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public OrderTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();

        // Mock 설정
        _factory.MockPaymentGateway
            .Setup(p => p.ProcessPaymentAsync(It.IsAny<PaymentRequest>()))
            .ReturnsAsync(new PaymentResult { Success = true });
    }

    [Fact]
    public async Task CreateOrder_ValidOrder_ReturnsCreated()
    {
        var order = new CreateOrderRequest
        {
            CustomerId = 1,
            Items = new[] { new OrderItem { ProductId = 1, Quantity = 2 } }
        };

        var response = await _client.PostAsJsonAsync("/api/orders", order);

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);

        // Mock 호출 검증
        _factory.MockPaymentGateway.Verify(
            p => p.ProcessPaymentAsync(It.IsAny<PaymentRequest>()),
            Times.Once);
    }
}
```

---

## 인증/인가 테스트

### 인증 우회 (테스트용)

```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 테스트용 인증 핸들러 추가
            services.AddAuthentication("Test")
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    "Test", options => { });
        });
    }
}

public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, "TestUser"),
            new Claim(ClaimTypes.NameIdentifier, "1"),
            new Claim(ClaimTypes.Role, "Admin")
        };

        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

### 특정 사용자로 테스트

```csharp
public class AuthorizedClient
{
    private readonly HttpClient _client;

    public AuthorizedClient(WebApplicationFactory<Program> factory, ClaimsPrincipal user)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                services.AddAuthentication("Test")
                    .AddScheme<TestAuthSchemeOptions, ConfigurableAuthHandler>(
                        "Test", options => options.User = user);
            });
        }).CreateClient();
    }

    public HttpClient Client => _client;
}

public class TestAuthSchemeOptions : AuthenticationSchemeOptions
{
    public ClaimsPrincipal? User { get; set; }
}

public class ConfigurableAuthHandler : AuthenticationHandler<TestAuthSchemeOptions>
{
    public ConfigurableAuthHandler(
        IOptionsMonitor<TestAuthSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (Options.User == null)
            return Task.FromResult(AuthenticateResult.Fail("No user configured"));

        var ticket = new AuthenticationTicket(Options.User, "Test");
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

---

## 테스트 격리

### 테스트별 데이터베이스

```csharp
public class IsolatedDatabaseTests : IAsyncLifetime
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly string _dbName;

    public IsolatedDatabaseTests()
    {
        _dbName = $"TestDb_{Guid.NewGuid()}";
        _factory = new CustomWebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseInMemoryDatabase(_dbName));
                });
            });
    }

    public async Task InitializeAsync()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.EnsureDeletedAsync();
    }

    [Fact]
    public async Task Test1() { /* ... */ }
}
```

### Collection Fixture로 공유

```csharp
public class SharedDatabaseFixture : IAsyncLifetime
{
    public CustomWebApplicationFactory<Program> Factory { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Factory = new CustomWebApplicationFactory<Program>();
        // 공유 데이터 시딩
    }

    public async Task DisposeAsync()
    {
        await Factory.DisposeAsync();
    }
}

[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<SharedDatabaseFixture> { }

[Collection("Database")]
public class TestClass1
{
    private readonly SharedDatabaseFixture _fixture;
    public TestClass1(SharedDatabaseFixture fixture) => _fixture = fixture;
}

[Collection("Database")]
public class TestClass2
{
    private readonly SharedDatabaseFixture _fixture;
    public TestClass2(SharedDatabaseFixture fixture) => _fixture = fixture;
}
```

---

## Testcontainers 사용

실제 데이터베이스를 Docker 컨테이너로 실행합니다.

### 설치

```xml
<PackageReference Include="Testcontainers" Version="3.6.0" />
<PackageReference Include="Testcontainers.PostgreSql" Version="3.6.0" />
```

### PostgreSQL 컨테이너

```csharp
public class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:15")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await _container.DisposeAsync();
    }
}

public class DatabaseIntegrationTests : IClassFixture<PostgresFixture>
{
    private readonly PostgresFixture _fixture;

    public DatabaseIntegrationTests(PostgresFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task Database_ConnectionWorks()
    {
        await using var connection = new NpgsqlConnection(_fixture.ConnectionString);
        await connection.OpenAsync();

        var result = await connection.ExecuteScalarAsync<int>("SELECT 1");

        Assert.Equal(1, result);
    }
}
```

### WebApplicationFactory와 Testcontainers 통합

```csharp
public class IntegrationTestFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithImage("postgres:15")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_dbContainer.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        // 마이그레이션 실행
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

---

## 응답 검증 헬퍼

```csharp
public static class HttpResponseMessageExtensions
{
    public static async Task<T> DeserializeAsync<T>(
        this HttpResponseMessage response)
    {
        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<T>(content,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true })!;
    }

    public static async Task ShouldBeSuccessWithContentAsync<T>(
        this HttpResponseMessage response,
        Action<T> assertions)
    {
        response.EnsureSuccessStatusCode();
        var content = await response.DeserializeAsync<T>();
        assertions(content);
    }
}

// 사용
[Fact]
public async Task GetUsers_ReturnsUsers()
{
    var response = await _client.GetAsync("/api/users");

    await response.ShouldBeSuccessWithContentAsync<List<UserDto>>(users =>
    {
        Assert.NotEmpty(users);
        Assert.All(users, u => Assert.NotNull(u.Name));
    });
}
```

---

## 면접 질문

### Q1: WebApplicationFactory란?
**A**: ASP.NET Core 애플리케이션을 메모리 내 테스트 서버로 호스팅하는 팩토리입니다. 실제 HTTP 요청 없이 전체 요청 파이프라인을 테스트할 수 있습니다.

### Q2: 통합 테스트에서 데이터베이스를 어떻게 처리하나요?
**A**:
1. **In-Memory Database**: 빠르지만 실제 DB와 동작 차이
2. **SQLite In-Memory**: 좋은 절충안
3. **Testcontainers**: 실제 DB를 Docker로 실행 (가장 신뢰성 높음)

### Q3: 테스트 격리는 어떻게 보장하나요?
**A**:
- 각 테스트마다 고유한 데이터베이스 이름 사용
- `IAsyncLifetime`으로 설정/정리
- Transaction 롤백 패턴
- Testcontainers로 완전히 격리된 환경
