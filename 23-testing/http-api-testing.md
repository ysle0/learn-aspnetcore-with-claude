# HTTP API Integration Testing

REST API의 통합 테스트(인수 테스트) 작성 방법을 다룹니다.

## 기본 구조

### 프로젝트 설정

```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
<PackageReference Include="FluentAssertions" Version="6.12.0" />
<PackageReference Include="xunit" Version="2.6.2" />
```

### 테스트 기본 클래스

```csharp
public class ApiTestBase : IClassFixture<CustomWebApplicationFactory<Program>>
{
    protected readonly HttpClient Client;
    protected readonly CustomWebApplicationFactory<Program> Factory;

    public ApiTestBase(CustomWebApplicationFactory<Program> factory)
    {
        Factory = factory;
        Client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false,
            BaseAddress = new Uri("https://localhost")
        });
    }

    protected async Task<T> GetAsync<T>(string url)
    {
        var response = await Client.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>()
            ?? throw new InvalidOperationException("Null response");
    }

    protected async Task<HttpResponseMessage> PostAsync<T>(string url, T data)
    {
        return await Client.PostAsJsonAsync(url, data);
    }
}
```

---

## CRUD API 테스트

### 테스트 대상 API

```csharp
// ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service) => _service = service;

    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
        => Ok(await _service.GetAllAsync());

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product == null) return NotFound();
        return Ok(product);
    }

    [HttpPost]
    public async Task<ActionResult<ProductDto>> Create(CreateProductDto dto)
    {
        var product = await _service.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, UpdateProductDto dto)
    {
        var success = await _service.UpdateAsync(id, dto);
        if (!success) return NotFound();
        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var success = await _service.DeleteAsync(id);
        if (!success) return NotFound();
        return NoContent();
    }
}
```

### 전체 CRUD 테스트

```csharp
public class ProductsApiTests : ApiTestBase
{
    public ProductsApiTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetAll_ReturnsProductList()
    {
        // Act
        var response = await Client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNull();
    }

    [Fact]
    public async Task GetById_ExistingProduct_ReturnsProduct()
    {
        // Arrange - 먼저 제품 생성
        var createResponse = await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "Test Product", Price = 99.99m });
        var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        // Act
        var response = await Client.GetAsync($"/api/products/{created!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        product!.Name.Should().Be("Test Product");
        product.Price.Should().Be(99.99m);
    }

    [Fact]
    public async Task GetById_NonExistingProduct_ReturnsNotFound()
    {
        // Act
        var response = await Client.GetAsync("/api/products/99999");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task Create_ValidProduct_ReturnsCreated()
    {
        // Arrange
        var newProduct = new CreateProductDto
        {
            Name = "New Product",
            Price = 49.99m,
            Category = "Electronics"
        };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", newProduct);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        product!.Id.Should().BeGreaterThan(0);
        product.Name.Should().Be("New Product");
    }

    [Fact]
    public async Task Create_InvalidProduct_ReturnsBadRequest()
    {
        // Arrange - 필수 필드 누락
        var invalidProduct = new CreateProductDto
        {
            Name = "", // 빈 이름
            Price = -10 // 음수 가격
        };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", invalidProduct);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

        var problemDetails = await response.Content
            .ReadFromJsonAsync<ValidationProblemDetails>();
        problemDetails!.Errors.Should().ContainKey("Name");
        problemDetails.Errors.Should().ContainKey("Price");
    }

    [Fact]
    public async Task Update_ExistingProduct_ReturnsNoContent()
    {
        // Arrange - 제품 생성
        var createResponse = await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "Original", Price = 100m });
        var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        var updateDto = new UpdateProductDto
        {
            Name = "Updated",
            Price = 150m
        };

        // Act
        var response = await Client.PutAsJsonAsync(
            $"/api/products/{created!.Id}", updateDto);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify update
        var getResponse = await Client.GetAsync($"/api/products/{created.Id}");
        var updated = await getResponse.Content.ReadFromJsonAsync<ProductDto>();
        updated!.Name.Should().Be("Updated");
        updated.Price.Should().Be(150m);
    }

    [Fact]
    public async Task Delete_ExistingProduct_ReturnsNoContent()
    {
        // Arrange
        var createResponse = await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "ToDelete", Price = 10m });
        var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        // Act
        var response = await Client.DeleteAsync($"/api/products/{created!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verify deletion
        var getResponse = await Client.GetAsync($"/api/products/{created.Id}");
        getResponse.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## 인증이 필요한 API 테스트

### JWT 토큰 생성 헬퍼

```csharp
public static class TestJwtTokenGenerator
{
    public static string GenerateToken(
        string userId = "1",
        string userName = "testuser",
        string[] roles = null,
        Dictionary<string, string> additionalClaims = null)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId),
            new(ClaimTypes.Name, userName),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };

        if (roles != null)
        {
            claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));
        }

        if (additionalClaims != null)
        {
            claims.AddRange(additionalClaims.Select(kv => new Claim(kv.Key, kv.Value)));
        }

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes("YourSuperSecretTestKey12345678901234567890"));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: "TestIssuer",
            audience: "TestAudience",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### 인증 테스트

```csharp
public class SecureApiTests : ApiTestBase
{
    public SecureApiTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetSecureEndpoint_WithoutToken_ReturnsUnauthorized()
    {
        // Act
        var response = await Client.GetAsync("/api/secure/data");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GetSecureEndpoint_WithValidToken_ReturnsOk()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken();
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);

        // Act
        var response = await Client.GetAsync("/api/secure/data");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task GetAdminEndpoint_WithUserRole_ReturnsForbidden()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken(roles: new[] { "User" });
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);

        // Act
        var response = await Client.GetAsync("/api/admin/settings");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    [Fact]
    public async Task GetAdminEndpoint_WithAdminRole_ReturnsOk()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken(roles: new[] { "Admin" });
        Client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);

        // Act
        var response = await Client.GetAsync("/api/admin/settings");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

### 인증된 클라이언트 팩토리

```csharp
public class AuthenticatedClientFactory
{
    private readonly WebApplicationFactory<Program> _factory;

    public AuthenticatedClientFactory(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    public HttpClient CreateClientAsUser(int userId, string userName, params string[] roles)
    {
        var client = _factory.CreateClient();
        var token = TestJwtTokenGenerator.GenerateToken(
            userId: userId.ToString(),
            userName: userName,
            roles: roles);
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token);
        return client;
    }

    public HttpClient CreateClientAsAdmin()
        => CreateClientAsUser(1, "admin", "Admin");

    public HttpClient CreateClientAsRegularUser()
        => CreateClientAsUser(2, "user", "User");
}
```

---

## 페이지네이션/필터링 테스트

```csharp
public class PaginationTests : ApiTestBase
{
    public PaginationTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetProducts_WithPagination_ReturnsPagedResult()
    {
        // Act
        var response = await Client.GetAsync("/api/products?page=1&pageSize=10");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var result = await response.Content
            .ReadFromJsonAsync<PagedResult<ProductDto>>();

        result!.Items.Should().HaveCountLessOrEqualTo(10);
        result.Page.Should().Be(1);
        result.PageSize.Should().Be(10);
        result.TotalCount.Should().BeGreaterOrEqualTo(0);
    }

    [Fact]
    public async Task GetProducts_WithFilter_ReturnsFilteredResult()
    {
        // Arrange - 테스트 데이터 생성
        await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "Apple", Category = "Fruit", Price = 1.5m });
        await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "Banana", Category = "Fruit", Price = 0.5m });
        await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "Laptop", Category = "Electronics", Price = 999m });

        // Act
        var response = await Client.GetAsync("/api/products?category=Fruit");

        // Assert
        var result = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        result.Should().OnlyContain(p => p.Category == "Fruit");
    }

    [Fact]
    public async Task GetProducts_WithSorting_ReturnsSortedResult()
    {
        // Act
        var response = await Client.GetAsync("/api/products?sortBy=price&sortOrder=desc");

        // Assert
        var result = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        result.Should().BeInDescendingOrder(p => p.Price);
    }

    [Fact]
    public async Task GetProducts_WithSearch_ReturnsMatchingResults()
    {
        // Arrange
        await Client.PostAsJsonAsync("/api/products",
            new CreateProductDto { Name = "iPhone 15 Pro", Price = 999m });

        // Act
        var response = await Client.GetAsync("/api/products?search=iphone");

        // Assert
        var result = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        result.Should().Contain(p => p.Name.Contains("iPhone", StringComparison.OrdinalIgnoreCase));
    }
}
```

---

## 에러 응답 테스트

```csharp
public class ErrorHandlingTests : ApiTestBase
{
    public ErrorHandlingTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetProduct_NotFound_ReturnsProblemDetails()
    {
        // Act
        var response = await Client.GetAsync("/api/products/99999");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
        response.Content.Headers.ContentType?.MediaType
            .Should().Be("application/problem+json");

        var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();
        problem!.Status.Should().Be(404);
        problem.Title.Should().NotBeEmpty();
    }

    [Fact]
    public async Task CreateProduct_ValidationError_ReturnsProblemDetails()
    {
        // Arrange
        var invalid = new CreateProductDto { Name = "", Price = -1 };

        // Act
        var response = await Client.PostAsJsonAsync("/api/products", invalid);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

        var problem = await response.Content
            .ReadFromJsonAsync<ValidationProblemDetails>();
        problem!.Errors.Should().NotBeEmpty();
    }

    [Fact]
    public async Task InvalidEndpoint_Returns404()
    {
        // Act
        var response = await Client.GetAsync("/api/nonexistent");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task InvalidMethod_Returns405()
    {
        // Act
        var response = await Client.PatchAsync("/api/products/1",
            new StringContent(""));

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.MethodNotAllowed);
    }
}
```

---

## 파일 업로드 테스트

```csharp
public class FileUploadTests : ApiTestBase
{
    public FileUploadTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task UploadFile_ValidImage_ReturnsOk()
    {
        // Arrange
        using var content = new MultipartFormDataContent();
        var imageBytes = CreateTestImage();
        var imageContent = new ByteArrayContent(imageBytes);
        imageContent.Headers.ContentType = MediaTypeHeaderValue.Parse("image/png");
        content.Add(imageContent, "file", "test.png");

        // Act
        var response = await Client.PostAsync("/api/files/upload", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var result = await response.Content.ReadFromJsonAsync<FileUploadResult>();
        result!.FileName.Should().EndWith(".png");
        result.Url.Should().NotBeEmpty();
    }

    [Fact]
    public async Task UploadFile_TooLarge_ReturnsBadRequest()
    {
        // Arrange
        using var content = new MultipartFormDataContent();
        var largeFile = new byte[50 * 1024 * 1024]; // 50MB
        content.Add(new ByteArrayContent(largeFile), "file", "large.bin");

        // Act
        var response = await Client.PostAsync("/api/files/upload", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task UploadFile_InvalidType_ReturnsBadRequest()
    {
        // Arrange
        using var content = new MultipartFormDataContent();
        var exeContent = new ByteArrayContent(new byte[] { 0x4D, 0x5A }); // MZ header
        exeContent.Headers.ContentType = MediaTypeHeaderValue.Parse("application/octet-stream");
        content.Add(exeContent, "file", "malware.exe");

        // Act
        var response = await Client.PostAsync("/api/files/upload", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    private static byte[] CreateTestImage()
    {
        // 1x1 transparent PNG
        return Convert.FromBase64String(
            "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==");
    }
}
```

---

## Content Negotiation 테스트

```csharp
public class ContentNegotiationTests : ApiTestBase
{
    public ContentNegotiationTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetProduct_AcceptJson_ReturnsJson()
    {
        // Arrange
        Client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));

        // Act
        var response = await Client.GetAsync("/api/products/1");

        // Assert
        response.Content.Headers.ContentType?.MediaType.Should().Be("application/json");
    }

    [Fact]
    public async Task GetProduct_AcceptXml_ReturnsXml()
    {
        // Arrange
        Client.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/xml"));

        // Act
        var response = await Client.GetAsync("/api/products/1");

        // Assert
        response.Content.Headers.ContentType?.MediaType.Should().Be("application/xml");
    }
}
```

---

## 성능/부하 관련 테스트

```csharp
public class PerformanceTests : ApiTestBase
{
    public PerformanceTests(CustomWebApplicationFactory<Program> factory)
        : base(factory) { }

    [Fact]
    public async Task GetProducts_RespondsWithinTimeout()
    {
        // Arrange
        var stopwatch = Stopwatch.StartNew();

        // Act
        var response = await Client.GetAsync("/api/products");

        // Assert
        stopwatch.Stop();
        response.EnsureSuccessStatusCode();
        stopwatch.ElapsedMilliseconds.Should().BeLessThan(1000); // 1초 이내
    }

    [Fact]
    public async Task ConcurrentRequests_AllSucceed()
    {
        // Arrange
        var tasks = Enumerable.Range(0, 10)
            .Select(_ => Client.GetAsync("/api/products"))
            .ToList();

        // Act
        var responses = await Task.WhenAll(tasks);

        // Assert
        responses.Should().OnlyContain(r => r.IsSuccessStatusCode);
    }
}
```

---

## 면접 질문

### Q1: 통합 테스트에서 외부 API를 어떻게 처리하나요?
**A**: `WebApplicationFactory.ConfigureServices`에서 외부 서비스를 Mock으로 교체하거나, `WireMock`을 사용하여 실제 HTTP 응답을 시뮬레이션합니다.

### Q2: 테스트 간 데이터 격리는 어떻게 하나요?
**A**:
1. 각 테스트마다 고유한 In-Memory DB 생성
2. Testcontainers로 격리된 DB 컨테이너 사용
3. 각 테스트 후 데이터 정리 (`IAsyncLifetime`)
4. Transaction 롤백 패턴

### Q3: API 버저닝 테스트는 어떻게 하나요?
**A**:
```csharp
// URL 경로 버저닝
await Client.GetAsync("/api/v1/products");
await Client.GetAsync("/api/v2/products");

// 헤더 버저닝
Client.DefaultRequestHeaders.Add("api-version", "2.0");
```
