# Testing Best Practices

"Unit Testing Principles, Practices, and Patterns" (Vladimir Khorikov), "The Art of Unit Testing" (Roy Osherove), "xUnit Test Patterns" (Gerard Meszaros) 등의 명저에서 제시하는 테스트 설계 원칙을 다룹니다.

## 1. 좋은 단위 테스트의 4대 원칙

Vladimir Khorikov의 4가지 핵심 원칙입니다.

| 원칙 | 설명 |
|------|------|
| **회귀 방지** | 버그를 발견하는 능력 |
| **리팩토링 내성** | 거짓 양성(false positive) 최소화 |
| **빠른 피드백** | 테스트 실행 속도 |
| **유지보수성** | 테스트 이해 및 수정 용이성 |

### 회귀 방지 vs 리팩토링 내성

```csharp
// ❌ 리팩토링 내성이 낮은 테스트 - 구현 세부사항에 의존
[Fact]
public void CalculateDiscount_CallsRepositoryWithCorrectId()
{
    var mockRepo = new Mock<IProductRepository>();
    var service = new PricingService(mockRepo.Object);

    service.CalculateDiscount(productId: 1);

    // 내부 구현(저장소 호출)을 검증 - 리팩토링하면 깨짐
    mockRepo.Verify(r => r.GetById(1), Times.Once);
}

// ✅ 리팩토링 내성이 높은 테스트 - 결과(출력)를 검증
[Fact]
public void CalculateDiscount_BulkOrder_Returns10PercentDiscount()
{
    var service = new PricingService(new FakeProductRepository());

    var discount = service.CalculateDiscount(productId: 1, quantity: 100);

    Assert.Equal(0.10m, discount);
}
```

---

## 2. 테스트 격리: Classical vs London School

### Classical School (Detroit/Chicago)
- 단위 = 동작의 단위
- 공유 의존성만 격리 (데이터베이스, 파일 시스템)
- Fake 선호

```csharp
// Classical: 협력 객체는 실제 구현 사용
[Fact]
public void Order_Total_CalculatesCorrectly()
{
    var cart = new ShoppingCart();
    cart.AddItem(new Product("Apple", 1.5m), 3);
    cart.AddItem(new Product("Banana", 0.5m), 2);

    var order = new Order(cart);  // 실제 ShoppingCart 사용

    Assert.Equal(5.5m, order.Total);
}
```

### London School
- 단위 = 클래스
- 모든 의존성을 격리
- Mock 선호

```csharp
// London: 모든 의존성을 Mock
[Fact]
public void Order_Total_CallsCartTotal()
{
    var mockCart = new Mock<IShoppingCart>();
    mockCart.Setup(c => c.Total).Returns(5.5m);

    var order = new Order(mockCart.Object);

    Assert.Equal(5.5m, order.Total);
    mockCart.Verify(c => c.Total, Times.Once);
}
```

### 권장 사항
> Classical School이 더 나은 리팩토링 내성을 제공합니다. Mock은 **외부 의존성**에만 사용하세요.

---

## 3. 관리 가능한 의존성 vs 비관리 의존성

```
┌─────────────────────────────────────────────────────────────┐
│                        의존성 분류                           │
├──────────────────────────────┬──────────────────────────────┤
│     관리 가능 (Managed)       │    비관리 (Unmanaged)        │
├──────────────────────────────┼──────────────────────────────┤
│ - 인메모리 컬렉션             │ - 데이터베이스                │
│ - 도메인 서비스               │ - 외부 API                   │
│ - 헬퍼 클래스                 │ - 메시지 큐                  │
│                              │ - 이메일 서비스               │
├──────────────────────────────┼──────────────────────────────┤
│ → 실제 구현 또는 Fake 사용    │ → Mock 사용                  │
└──────────────────────────────┴──────────────────────────────┘
```

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;      // 비관리 (Mock)
    private readonly IEmailService _emailService;       // 비관리 (Mock)
    private readonly PriceCalculator _calculator;       // 관리 (실제 사용)
    private readonly DiscountPolicy _discountPolicy;    // 관리 (실제 사용)

    // ...
}

[Fact]
public void CreateOrder_ValidOrder_SendsEmail()
{
    // 비관리 의존성만 Mock
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();

    // 관리 의존성은 실제 구현 사용
    var calculator = new PriceCalculator();
    var policy = new StandardDiscountPolicy();

    var service = new OrderService(
        mockRepo.Object,
        mockEmail.Object,
        calculator,  // 실제 구현
        policy);     // 실제 구현

    service.CreateOrder(new OrderRequest { /* ... */ });

    mockEmail.Verify(e => e.Send(It.IsAny<string>()), Times.Once);
}
```

---

## 4. 테스트 가능한 설계

### 의존성 주입 사용
```csharp
// ❌ 테스트 불가능 - 의존성이 하드코딩됨
public class ReportGenerator
{
    public string Generate()
    {
        var data = new SqlDatabase().GetData();  // 하드코딩
        var formatter = new PdfFormatter();       // 하드코딩
        return formatter.Format(data);
    }
}

// ✅ 테스트 가능 - 의존성 주입
public class ReportGenerator
{
    private readonly IDataSource _dataSource;
    private readonly IFormatter _formatter;

    public ReportGenerator(IDataSource dataSource, IFormatter formatter)
    {
        _dataSource = dataSource;
        _formatter = formatter;
    }

    public string Generate()
    {
        var data = _dataSource.GetData();
        return _formatter.Format(data);
    }
}
```

### 순수 함수 분리
```csharp
// ✅ 순수 함수는 테스트하기 쉬움
public static class PriceCalculator
{
    public static decimal CalculateTotal(IEnumerable<OrderItem> items, decimal taxRate)
    {
        var subtotal = items.Sum(i => i.Price * i.Quantity);
        return subtotal * (1 + taxRate);
    }
}

[Theory]
[InlineData(100, 0.1, 110)]
[InlineData(200, 0.2, 240)]
public void CalculateTotal_AppliesTax(decimal subtotal, decimal tax, decimal expected)
{
    var items = new[] { new OrderItem { Price = subtotal, Quantity = 1 } };

    var total = PriceCalculator.CalculateTotal(items, tax);

    Assert.Equal(expected, total);
}
```

### Humble Object 패턴
테스트하기 어려운 로직을 분리합니다.

```csharp
// ❌ 테스트 어려움 - UI와 로직이 섞임
public class OrderController : Controller
{
    public IActionResult Create(OrderFormModel model)
    {
        // 유효성 검사, 비즈니스 로직, 데이터베이스 접근이 모두 섞임
        if (model.Items.Count == 0) return BadRequest();
        var discount = CalculateDiscount(model);
        // ...
    }
}

// ✅ Humble Object - 컨트롤러는 단순히 위임만
public class OrderController : Controller
{
    private readonly IOrderService _service;

    public IActionResult Create(OrderFormModel model)
    {
        var result = _service.CreateOrder(model.ToRequest());
        return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
    }
}

// 비즈니스 로직은 별도 서비스에서 테스트
public class OrderService : IOrderService
{
    public Result<Order> CreateOrder(CreateOrderRequest request)
    {
        if (!request.Items.Any())
            return Result.Failure<Order>("Items required");

        var discount = CalculateDiscount(request);
        // ...
    }
}
```

---

## 5. 테스트 안티패턴

### 1. 과도한 Setup
```csharp
// ❌ 복잡한 Setup은 테스트 이해를 방해
[Fact]
public void ComplexTest()
{
    var config = new ConfigurationBuilder()
        .AddInMemoryCollection(new Dictionary<string, string>
        {
            ["Setting1"] = "Value1",
            ["Setting2"] = "Value2",
            // ... 20줄의 설정
        })
        .Build();

    var services = new ServiceCollection();
    services.AddSingleton<IConfiguration>(config);
    // ... 10줄의 서비스 등록

    var provider = services.BuildServiceProvider();
    var service = provider.GetRequiredService<IMyService>();

    var result = service.DoSomething();  // 실제 테스트는 이것뿐

    Assert.NotNull(result);
}

// ✅ Builder 패턴으로 단순화
[Fact]
public void SimpleTest()
{
    var service = new MyServiceBuilder()
        .WithDefaultConfig()
        .Build();

    var result = service.DoSomething();

    Assert.NotNull(result);
}
```

### 2. 테스트에서 로직 사용
```csharp
// ❌ 테스트에 조건문 - 무엇을 테스트하는지 불명확
[Fact]
public void CalculatePrice_VaryingInputs()
{
    var items = GetTestItems();

    var total = calculator.Calculate(items);

    var expected = 0m;
    foreach (var item in items)
    {
        if (item.IsDiscounted)
            expected += item.Price * 0.9m;
        else
            expected += item.Price;
    }

    Assert.Equal(expected, total);  // 버그가 테스트에도 있으면?
}

// ✅ 명시적인 기대값
[Fact]
public void CalculatePrice_WithDiscount_AppliesDiscount()
{
    var items = new[]
    {
        new Item { Price = 100m, IsDiscounted = true }
    };

    var total = calculator.Calculate(items);

    Assert.Equal(90m, total);  // 명시적 기대값
}
```

### 3. 비공개 메서드 테스트
```csharp
// ❌ private 메서드 직접 테스트
[Fact]
public void ValidateEmail_InvalidEmail_ReturnsFalse()
{
    var service = new UserService();
    var method = typeof(UserService)
        .GetMethod("ValidateEmail", BindingFlags.NonPublic | BindingFlags.Instance);

    var result = (bool)method.Invoke(service, new[] { "invalid" });

    Assert.False(result);
}

// ✅ public API를 통해 간접 테스트
[Fact]
public void CreateUser_InvalidEmail_ReturnsValidationError()
{
    var service = new UserService();

    var result = service.CreateUser(new CreateUserDto { Email = "invalid" });

    Assert.False(result.IsSuccess);
    Assert.Contains("email", result.Error.ToLower());
}
```

### 4. 테스트 간 상태 공유
```csharp
// ❌ 정적 상태 공유 - 테스트 순서에 따라 결과 다름
public class SharedStateTests
{
    private static int _counter = 0;

    [Fact]
    public void Test1()
    {
        _counter++;
        Assert.Equal(1, _counter);  // Test2가 먼저 실행되면 실패
    }

    [Fact]
    public void Test2()
    {
        _counter++;
        Assert.Equal(1, _counter);  // Test1이 먼저 실행되면 실패
    }
}

// ✅ 각 테스트가 독립적
public class IsolatedTests
{
    [Fact]
    public void Test1()
    {
        var counter = 0;
        counter++;
        Assert.Equal(1, counter);
    }

    [Fact]
    public void Test2()
    {
        var counter = 0;
        counter++;
        Assert.Equal(1, counter);
    }
}
```

---

## 6. 테스트 명명과 구조

### 명명 규칙

| 패턴 | 예시 |
|------|------|
| `[Method]_[Scenario]_[Expected]` | `Withdraw_InsufficientFunds_ThrowsException` |
| `[Action]_When_[Condition]` | `ThrowsException_When_AmountExceedsBalance` |
| `Should_[Expected]_When_[Condition]` | `Should_SendEmail_When_OrderCreated` |

### 구조화된 테스트

```csharp
public class UserServiceTests
{
    public class CreateUser
    {
        [Fact]
        public void WithValidData_CreatesUser() { }

        [Fact]
        public void WithDuplicateEmail_ReturnsError() { }

        [Fact]
        public void WithInvalidEmail_ReturnsValidationError() { }
    }

    public class DeleteUser
    {
        [Fact]
        public void ExistingUser_DeletesSuccessfully() { }

        [Fact]
        public void NonExistingUser_ReturnsNotFound() { }
    }
}
```

---

## 7. 테스트 커버리지 가이드

### 커버리지 목표

| 코드 유형 | 권장 커버리지 |
|----------|--------------|
| 도메인 로직 | 90%+ |
| 비즈니스 서비스 | 80%+ |
| 컨트롤러 | 통합 테스트로 대체 |
| 인프라 코드 | 통합 테스트로 대체 |
| 설정/부트스트랩 | 낮음 |

### 커버리지보다 중요한 것

> "100% 커버리지는 0% 버그를 의미하지 않는다"

```csharp
// 100% 커버리지지만 버그 있음
[Fact]
public void Add_TwoNumbers()
{
    var result = Calculator.Add(2, 2);
    Assert.NotNull(result);  // 값 검증 없음!
}

// 커버리지는 낮지만 의미 있는 테스트
[Theory]
[InlineData(2, 2, 4)]
[InlineData(-1, 1, 0)]
[InlineData(int.MaxValue, 1, int.MinValue)]  // 오버플로우 케이스
public void Add_EdgeCases(int a, int b, int expected)
{
    Assert.Equal(expected, Calculator.Add(a, b));
}
```

---

## 8. 테스트 작성 체크리스트

### 작성 전
- [ ] 무엇을 테스트하는가? (요구사항/버그)
- [ ] 단위 테스트인가, 통합 테스트인가?
- [ ] 경계 조건은 무엇인가?

### 작성 중
- [ ] AAA 패턴을 따르는가?
- [ ] 테스트 이름이 의도를 설명하는가?
- [ ] 하나의 논리적 개념만 테스트하는가?
- [ ] Mock을 과도하게 사용하지 않았는가?

### 작성 후
- [ ] 테스트가 빠르게 실행되는가?
- [ ] 테스트가 독립적인가?
- [ ] 리팩토링해도 테스트가 깨지지 않는가?
- [ ] 테스트가 문서 역할을 하는가?

---

## 9. 추천 도서

| 도서 | 저자 | 핵심 내용 |
|------|------|----------|
| Unit Testing Principles, Practices, and Patterns | Vladimir Khorikov | 단위 테스트 설계 원칙 |
| The Art of Unit Testing | Roy Osherove | 테스트 작성 기법 |
| xUnit Test Patterns | Gerard Meszaros | 테스트 패턴 카탈로그 |
| Growing Object-Oriented Software, Guided by Tests | Freeman & Pryce | TDD 실천법 |
| Working Effectively with Legacy Code | Michael Feathers | 레거시 코드 테스트 |

---

## 면접 질문

### Q1: Mock과 Fake의 차이점과 각각 언제 사용하나요?
**A**:
- **Mock**: 호출 검증이 목적, 외부 의존성(이메일, 결제)에 사용
- **Fake**: 실제 동작하는 가벼운 구현, 내부 의존성(저장소)에 사용

Mock은 행위 검증, Fake는 상태 검증에 적합합니다.

### Q2: 테스트 커버리지 100%가 좋은 것인가요?
**A**: 아닙니다. 커버리지는 테스트되지 않은 코드를 찾는 데 유용하지만, 테스트의 품질을 보장하지 않습니다. 중요한 것은 비즈니스 로직의 핵심 경로와 경계 조건이 테스트되었는지입니다.

### Q3: 테스트가 실패할 때 어떻게 디버깅하나요?
**A**:
1. 테스트 이름에서 의도 파악
2. 실패 메시지 분석
3. Arrange 데이터 확인
4. 단일 테스트 격리 실행
5. 실제 동작과 기대값 비교
