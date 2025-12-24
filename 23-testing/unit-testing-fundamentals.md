# Unit Testing 기초

## 단위 테스트란?

단위 테스트는 **격리된 환경**에서 **단일 동작 단위**를 검증하는 테스트입니다.

### 좋은 단위 테스트의 특성 (F.I.R.S.T)

| 원칙 | 설명 |
|------|------|
| **F**ast | 빠르게 실행되어야 함 (밀리초 단위) |
| **I**solated | 다른 테스트에 의존하지 않음 |
| **R**epeatable | 언제 실행해도 같은 결과 |
| **S**elf-validating | 성공/실패가 자동으로 판정됨 |
| **T**imely | 프로덕션 코드와 함께 작성 |

---

## AAA 패턴 (Arrange-Act-Assert)

```csharp
[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum()
{
    // Arrange (준비) - 테스트 대상과 의존성 설정
    var calculator = new Calculator();

    // Act (실행) - 테스트할 동작 수행
    var result = calculator.Add(2, 3);

    // Assert (검증) - 결과 확인
    Assert.Equal(5, result);
}
```

### 각 단계의 비율
- **Arrange**: 가장 크거나 별도 메서드로 추출
- **Act**: 보통 한 줄 (테스트 대상 동작)
- **Assert**: 하나의 논리적 개념만 검증

---

## 테스트 명명 규칙

### 권장 패턴: `[Method]_[Scenario]_[ExpectedBehavior]`

```csharp
// ✅ 좋은 예
[Fact]
public void Withdraw_InsufficientFunds_ThrowsException() { }

[Fact]
public void GetUser_ValidId_ReturnsUser() { }

[Fact]
public void CreateOrder_EmptyCart_ReturnsValidationError() { }

// ❌ 나쁜 예
[Fact]
public void Test1() { }

[Fact]
public void WithdrawTest() { }
```

### BDD 스타일 (Given-When-Then)

```csharp
[Fact]
public void Given_InsufficientBalance_When_Withdraw_Then_ThrowsException() { }
```

---

## 테스트 대상 분류

### 1. 도메인 로직 (순수 함수)
```csharp
public class PriceCalculator
{
    public decimal CalculateDiscount(decimal price, int quantity)
    {
        if (quantity >= 10) return price * 0.9m;
        if (quantity >= 5) return price * 0.95m;
        return price;
    }
}

[Theory]
[InlineData(100, 10, 90)]   // 10% 할인
[InlineData(100, 5, 95)]    // 5% 할인
[InlineData(100, 3, 100)]   // 할인 없음
public void CalculateDiscount_VariousQuantities_ReturnsCorrectDiscount(
    decimal price, int quantity, decimal expected)
{
    var calculator = new PriceCalculator();

    var result = calculator.CalculateDiscount(price, quantity);

    Assert.Equal(expected, result);
}
```

### 2. 서비스 레이어 (의존성 있음)
```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;

    public OrderService(IOrderRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        var order = new Order(dto.CustomerId, dto.Items);
        await _repository.AddAsync(order);
        await _emailService.SendOrderConfirmationAsync(order);
        return order;
    }
}

[Fact]
public async Task CreateOrder_ValidOrder_SavesAndSendsEmail()
{
    // Arrange
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();
    var service = new OrderService(mockRepo.Object, mockEmail.Object);
    var dto = new CreateOrderDto { CustomerId = 1, Items = new[] { new OrderItem() } };

    // Act
    var result = await service.CreateOrderAsync(dto);

    // Assert
    mockRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
    mockEmail.Verify(e => e.SendOrderConfirmationAsync(It.IsAny<Order>()), Times.Once);
}
```

### 3. 컨트롤러 단위 테스트
```csharp
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        if (user == null) return NotFound();
        return Ok(user);
    }
}

[Fact]
public async Task GetUser_ExistingId_ReturnsOkWithUser()
{
    // Arrange
    var mockService = new Mock<IUserService>();
    mockService.Setup(s => s.GetByIdAsync(1))
        .ReturnsAsync(new UserDto { Id = 1, Name = "John" });
    var controller = new UsersController(mockService.Object);

    // Act
    var result = await controller.GetUser(1);

    // Assert
    var okResult = Assert.IsType<OkObjectResult>(result.Result);
    var user = Assert.IsType<UserDto>(okResult.Value);
    Assert.Equal("John", user.Name);
}

[Fact]
public async Task GetUser_NonExistingId_ReturnsNotFound()
{
    // Arrange
    var mockService = new Mock<IUserService>();
    mockService.Setup(s => s.GetByIdAsync(999))
        .ReturnsAsync((UserDto?)null);
    var controller = new UsersController(mockService.Object);

    // Act
    var result = await controller.GetUser(999);

    // Assert
    Assert.IsType<NotFoundResult>(result.Result);
}
```

---

## 테스트 데이터 생성

### Object Mother 패턴
```csharp
public static class UserMother
{
    public static User CreateDefault() => new User
    {
        Id = 1,
        Name = "John Doe",
        Email = "john@example.com",
        IsActive = true
    };

    public static User CreateInactive() => CreateDefault() with { IsActive = false };

    public static User CreateAdmin() => CreateDefault() with { Role = Role.Admin };
}

[Fact]
public void DeactivateUser_ActiveUser_SetsIsActiveToFalse()
{
    var user = UserMother.CreateDefault();

    user.Deactivate();

    Assert.False(user.IsActive);
}
```

### Builder 패턴
```csharp
public class UserBuilder
{
    private int _id = 1;
    private string _name = "John";
    private string _email = "john@example.com";
    private bool _isActive = true;

    public UserBuilder WithId(int id) { _id = id; return this; }
    public UserBuilder WithName(string name) { _name = name; return this; }
    public UserBuilder WithEmail(string email) { _email = email; return this; }
    public UserBuilder Inactive() { _isActive = false; return this; }

    public User Build() => new User
    {
        Id = _id,
        Name = _name,
        Email = _email,
        IsActive = _isActive
    };
}

[Fact]
public void Example_UsingBuilder()
{
    var user = new UserBuilder()
        .WithName("Jane")
        .WithEmail("jane@example.com")
        .Build();

    Assert.Equal("Jane", user.Name);
}
```

### Bogus (가짜 데이터 생성)
```csharp
using Bogus;

public class UserFaker : Faker<User>
{
    public UserFaker()
    {
        RuleFor(u => u.Id, f => f.IndexFaker + 1);
        RuleFor(u => u.Name, f => f.Name.FullName());
        RuleFor(u => u.Email, f => f.Internet.Email());
        RuleFor(u => u.CreatedAt, f => f.Date.Past());
    }
}

[Fact]
public void Example_UsingBogus()
{
    var faker = new UserFaker();
    var users = faker.Generate(10);

    Assert.Equal(10, users.Count);
    Assert.All(users, u => Assert.NotEmpty(u.Email));
}
```

---

## 예외 테스트

```csharp
[Fact]
public void Withdraw_NegativeAmount_ThrowsArgumentException()
{
    var account = new BankAccount(100);

    var exception = Assert.Throws<ArgumentException>(
        () => account.Withdraw(-50));

    Assert.Contains("negative", exception.Message.ToLower());
}

[Fact]
public async Task GetUser_InvalidId_ThrowsNotFoundException()
{
    var service = new UserService();

    await Assert.ThrowsAsync<NotFoundException>(
        () => service.GetByIdAsync(-1));
}
```

---

## 비동기 테스트

```csharp
[Fact]
public async Task FetchData_ValidUrl_ReturnsData()
{
    var service = new DataFetcher();

    var result = await service.FetchAsync("https://api.example.com/data");

    Assert.NotNull(result);
}

// Task 완료 대기 테스트
[Fact]
public async Task ProcessAsync_LongRunning_CompletesWithinTimeout()
{
    var processor = new DataProcessor();
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

    await processor.ProcessAsync(cts.Token);

    // 타임아웃 내에 완료되면 성공
}
```

---

## 테스트 구성

### Fact vs Theory

```csharp
// Fact: 단일 케이스 테스트
[Fact]
public void Add_TwoNumbers_ReturnsSum()
{
    Assert.Equal(4, Calculator.Add(2, 2));
}

// Theory: 여러 케이스를 데이터로 테스트
[Theory]
[InlineData(2, 2, 4)]
[InlineData(0, 0, 0)]
[InlineData(-1, 1, 0)]
[InlineData(int.MaxValue, 1, int.MinValue)] // 오버플로우
public void Add_VariousInputs_ReturnsExpected(int a, int b, int expected)
{
    Assert.Equal(expected, Calculator.Add(a, b));
}
```

### MemberData / ClassData

```csharp
public class CalculatorTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1, 2, 3 };
        yield return new object[] { -4, -6, -10 };
        yield return new object[] { -2, 2, 0 };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(CalculatorTestData))]
public void Add_ClassData_ReturnsExpected(int a, int b, int expected)
{
    Assert.Equal(expected, Calculator.Add(a, b));
}

// MemberData 사용
public static IEnumerable<object[]> AdditionData =>
    new List<object[]>
    {
        new object[] { 1, 2, 3 },
        new object[] { -4, -6, -10 }
    };

[Theory]
[MemberData(nameof(AdditionData))]
public void Add_MemberData_ReturnsExpected(int a, int b, int expected)
{
    Assert.Equal(expected, Calculator.Add(a, b));
}
```

---

## 테스트 격리

### 테스트별 새 인스턴스 (xUnit 기본)
```csharp
public class CalculatorTests
{
    private readonly Calculator _calculator = new();

    [Fact]
    public void Test1()
    {
        // 새 _calculator 인스턴스
    }

    [Fact]
    public void Test2()
    {
        // 또 다른 새 _calculator 인스턴스
    }
}
```

### IAsyncLifetime (비동기 Setup/Teardown)
```csharp
public class DatabaseTests : IAsyncLifetime
{
    private SqlConnection _connection;

    public async Task InitializeAsync()
    {
        _connection = new SqlConnection("...");
        await _connection.OpenAsync();
    }

    public async Task DisposeAsync()
    {
        await _connection.DisposeAsync();
    }

    [Fact]
    public async Task Query_ReturnsData()
    {
        var result = await _connection.QueryAsync("SELECT 1");
        Assert.NotEmpty(result);
    }
}
```

---

## 면접 질문

### Q1: 단위 테스트와 통합 테스트의 차이점은?
**A**: 단위 테스트는 격리된 환경에서 단일 컴포넌트를 테스트하고, 통합 테스트는 여러 컴포넌트가 함께 동작하는 것을 테스트합니다. 단위 테스트는 빠르고 Mock을 사용하며, 통합 테스트는 실제 의존성을 사용합니다.

### Q2: 테스트 커버리지는 얼마나 되어야 하나요?
**A**: 숫자(80%, 100%)보다 **비즈니스 로직의 핵심 경로**가 테스트되었는지가 중요합니다. 100% 커버리지도 모든 버그를 잡지 못하며, 낮은 커버리지도 중요한 로직을 잘 테스트할 수 있습니다.

### Q3: TDD(Test-Driven Development)란?
**A**: 테스트를 먼저 작성하고, 테스트를 통과하는 최소한의 코드를 작성한 후, 리팩토링하는 개발 방법론입니다. Red-Green-Refactor 사이클이라고도 합니다.

```
Red (실패하는 테스트 작성)
  ↓
Green (테스트 통과하는 코드 작성)
  ↓
Refactor (코드 정리)
  ↓
(반복)
```

### Q4: private 메서드를 어떻게 테스트하나요?
**A**: **테스트하지 않습니다**. private 메서드는 public API를 통해 간접적으로 테스트됩니다. private 메서드를 직접 테스트하고 싶다면, 그 로직을 별도 클래스로 추출하는 것이 더 좋은 설계입니다.
