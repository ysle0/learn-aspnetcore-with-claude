# Mocking과 Test Doubles

테스트에서 의존성을 대체하는 다양한 패턴과 도구를 다룹니다.

## Test Doubles 종류

Gerard Meszaros의 "xUnit Test Patterns"에서 정의한 테스트 더블 유형입니다.

| 유형 | 설명 | 사용 시점 |
|------|------|----------|
| **Dummy** | 전달만 되고 사용되지 않음 | 파라미터 채우기 |
| **Stub** | 미리 정해진 값 반환 | 간접 입력 제공 |
| **Spy** | 호출 기록 | 간접 출력 검증 |
| **Mock** | 기대값 검증 | 행위 검증 |
| **Fake** | 실제 동작하는 가벼운 구현 | 복잡한 의존성 대체 |

```
┌─────────────────────────────────────────────────────────┐
│                    Test Doubles                          │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│  Dummy   │   Stub   │   Spy    │   Mock   │    Fake     │
│          │          │          │          │             │
│ 사용안함  │ 값 반환   │ 호출기록  │ 행위검증  │ 실제구현    │
│          │          │          │          │ (간소화)    │
└──────────┴──────────┴──────────┴──────────┴─────────────┘
        간단 ◄────────────────────────────────► 복잡
```

---

## Moq (가장 인기 있는 Mocking 라이브러리)

### 설치
```xml
<PackageReference Include="Moq" Version="4.20.70" />
```

### 기본 사용법

```csharp
public interface IUserRepository
{
    User GetById(int id);
    Task<User> GetByIdAsync(int id);
    IEnumerable<User> GetAll();
    void Save(User user);
    Task SaveAsync(User user);
}

[Fact]
public void GetUser_ExistingId_ReturnsUser()
{
    // Arrange - Mock 생성
    var mockRepo = new Mock<IUserRepository>();

    // Stub 설정 - 특정 입력에 특정 출력 반환
    mockRepo.Setup(r => r.GetById(1))
        .Returns(new User { Id = 1, Name = "John" });

    var service = new UserService(mockRepo.Object);

    // Act
    var user = service.GetUser(1);

    // Assert
    Assert.Equal("John", user.Name);
}
```

### Setup 패턴

```csharp
// 특정 값 반환
mock.Setup(x => x.GetById(1)).Returns(new User { Id = 1 });

// 람다로 동적 반환
mock.Setup(x => x.GetById(It.IsAny<int>()))
    .Returns((int id) => new User { Id = id });

// 비동기 메서드
mock.Setup(x => x.GetByIdAsync(1))
    .ReturnsAsync(new User { Id = 1 });

// 예외 발생
mock.Setup(x => x.GetById(-1))
    .Throws<ArgumentException>();

mock.Setup(x => x.GetByIdAsync(-1))
    .ThrowsAsync(new NotFoundException());

// 순차적 반환
mock.SetupSequence(x => x.GetById(It.IsAny<int>()))
    .Returns(new User { Id = 1 })
    .Returns(new User { Id = 2 })
    .Throws<InvalidOperationException>();

// 콜백 실행
mock.Setup(x => x.Save(It.IsAny<User>()))
    .Callback<User>(u => Console.WriteLine($"Saved: {u.Name}"));

// 콜백 + 반환
mock.Setup(x => x.GetById(It.IsAny<int>()))
    .Callback<int>(id => Console.WriteLine($"Getting: {id}"))
    .Returns(new User());
```

### Argument Matching

```csharp
// 모든 값
mock.Setup(x => x.GetById(It.IsAny<int>())).Returns(new User());

// 특정 조건
mock.Setup(x => x.GetById(It.Is<int>(id => id > 0))).Returns(new User());

// 범위
mock.Setup(x => x.GetById(It.IsInRange(1, 100, Moq.Range.Inclusive)))
    .Returns(new User());

// 정규식 (문자열)
mock.Setup(x => x.GetByEmail(It.IsRegex(@".*@example\.com")))
    .Returns(new User());

// Null이 아닌 값
mock.Setup(x => x.GetByName(It.IsNotNull<string>()))
    .Returns(new User());
```

### Verify (행위 검증)

```csharp
[Fact]
public void CreateOrder_ValidOrder_SavesAndSendsEmail()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();
    var service = new OrderService(mockRepo.Object, mockEmail.Object);

    service.CreateOrder(new Order());

    // 호출 횟수 검증
    mockRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
    mockEmail.Verify(e => e.Send(It.IsAny<string>(), It.IsAny<string>()), Times.Once);

    // 호출되지 않음 검증
    mockRepo.Verify(r => r.Delete(It.IsAny<Order>()), Times.Never);
}

// Times 옵션
Times.Never         // 0번
Times.Once          // 정확히 1번
Times.Exactly(3)    // 정확히 3번
Times.AtLeast(2)    // 최소 2번
Times.AtMost(5)     // 최대 5번
Times.Between(2, 5, Moq.Range.Inclusive)  // 2~5번
```

### Properties

```csharp
public interface IConfig
{
    string ConnectionString { get; set; }
    int Timeout { get; }
}

// 프로퍼티 설정
mock.Setup(c => c.ConnectionString).Returns("Server=...");
mock.Setup(c => c.Timeout).Returns(30);

// 프로퍼티 추적 (SetupProperty)
mock.SetupProperty(c => c.ConnectionString, "initial");
mock.Object.ConnectionString = "new value";  // 변경 가능

// 모든 프로퍼티 추적
mock.SetupAllProperties();
```

### Strict vs Loose Mock

```csharp
// Loose (기본) - 설정되지 않은 호출은 기본값 반환
var looseMock = new Mock<IService>(MockBehavior.Loose);

// Strict - 설정되지 않은 호출은 예외 발생
var strictMock = new Mock<IService>(MockBehavior.Strict);
strictMock.Setup(s => s.Method()).Returns("value");
// 설정되지 않은 메서드 호출 시 MockException 발생
```

---

## NSubstitute (대안)

더 간결한 문법을 제공합니다.

### 설치
```xml
<PackageReference Include="NSubstitute" Version="5.1.0" />
```

### 기본 사용법

```csharp
using NSubstitute;

// 생성
var repo = Substitute.For<IUserRepository>();

// Setup
repo.GetById(1).Returns(new User { Id = 1, Name = "John" });

// 비동기
repo.GetByIdAsync(1).Returns(new User { Id = 1 });

// Argument matching
repo.GetById(Arg.Any<int>()).Returns(new User());
repo.GetById(Arg.Is<int>(x => x > 0)).Returns(new User());

// 예외
repo.GetById(-1).Returns(x => throw new ArgumentException());

// 콜백
repo.When(r => r.Save(Arg.Any<User>()))
    .Do(ci => Console.WriteLine($"Saved: {ci.Arg<User>().Name}"));

// Verify
repo.Received().Save(Arg.Any<User>());
repo.Received(1).GetById(1);
repo.DidNotReceive().Delete(Arg.Any<User>());
```

### Moq vs NSubstitute 비교

```csharp
// Moq
var mock = new Mock<IService>();
mock.Setup(s => s.GetValue()).Returns(42);
var result = mock.Object.GetValue();
mock.Verify(s => s.GetValue(), Times.Once);

// NSubstitute
var sub = Substitute.For<IService>();
sub.GetValue().Returns(42);
var result = sub.GetValue();
sub.Received().GetValue();
```

---

## Fake 구현 패턴

실제 동작하는 가벼운 구현을 만드는 패턴입니다.

### In-Memory Repository Fake

```csharp
public class FakeUserRepository : IUserRepository
{
    private readonly Dictionary<int, User> _users = new();
    private int _nextId = 1;

    public User GetById(int id) =>
        _users.TryGetValue(id, out var user) ? user : null;

    public Task<User> GetByIdAsync(int id) =>
        Task.FromResult(GetById(id));

    public IEnumerable<User> GetAll() => _users.Values;

    public void Save(User user)
    {
        if (user.Id == 0)
            user.Id = _nextId++;
        _users[user.Id] = user;
    }

    public Task SaveAsync(User user)
    {
        Save(user);
        return Task.CompletedTask;
    }

    public void Delete(User user) => _users.Remove(user.Id);

    // 테스트 헬퍼
    public void Clear() => _users.Clear();
    public int Count => _users.Count;
}
```

### 사용 예시

```csharp
[Fact]
public void CreateUser_ValidUser_SavesSuccessfully()
{
    // Fake는 실제 동작하므로 상태 검증 가능
    var fakeRepo = new FakeUserRepository();
    var service = new UserService(fakeRepo);

    service.CreateUser("John", "john@example.com");

    Assert.Equal(1, fakeRepo.Count);
    var user = fakeRepo.GetAll().First();
    Assert.Equal("John", user.Name);
}
```

---

## Mock vs Fake 선택 기준

| 상황 | 권장 | 이유 |
|------|------|------|
| 외부 API 호출 | Mock | 호출 여부만 검증 |
| 이메일 발송 | Mock | 부수 효과 방지 |
| 데이터베이스 | Fake | 상태 기반 테스트 |
| 캐시 | Fake | 실제 캐싱 동작 검증 |
| 복잡한 로직 검증 | Fake | 통합적 동작 검증 |
| 간단한 의존성 | Mock | 빠른 설정 |

### Vladimir Khorikov의 권장 사항

> "Mocks and stubs are for unmanaged dependencies (out-of-process dependencies you don't control). For managed dependencies (in-process dependencies), use the real implementation or a fake."

```csharp
// ❌ 안 좋은 예 - 모든 것을 Mock
var mockLogger = new Mock<ILogger>();
var mockConfig = new Mock<IConfiguration>();
var mockCache = new Mock<ICache>();

// ✅ 좋은 예 - 외부 의존성만 Mock
var mockEmailService = new Mock<IEmailService>();  // 외부 API
var mockPaymentGateway = new Mock<IPaymentGateway>();  // 외부 API
var fakeCache = new FakeMemoryCache();  // 내부 의존성은 Fake
```

---

## Auto Mocking Containers

의존성이 많은 클래스 테스트를 단순화합니다.

### AutoMoq (Moq용)

```xml
<PackageReference Include="Moq.AutoMock" Version="3.5.0" />
```

```csharp
using Moq.AutoMock;

[Fact]
public void CreateOrder_WithAutoMock()
{
    // 모든 의존성을 자동으로 Mock 생성
    var mocker = new AutoMocker();

    // 특정 Mock 설정
    mocker.GetMock<IUserRepository>()
        .Setup(r => r.GetById(1))
        .Returns(new User { Id = 1 });

    // 테스트 대상 생성 (의존성 자동 주입)
    var service = mocker.CreateInstance<OrderService>();

    var order = service.CreateOrder(1, new[] { new OrderItem() });

    mocker.GetMock<IOrderRepository>()
        .Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
}
```

---

## HttpClient Mocking

### MockHttp 사용

```xml
<PackageReference Include="RichardSzalay.MockHttp" Version="7.0.0" />
```

```csharp
using RichardSzalay.MockHttp;

[Fact]
public async Task GetWeather_ReturnsData()
{
    var mockHttp = new MockHttpMessageHandler();

    mockHttp.When("https://api.weather.com/current")
        .Respond("application/json", """
        {
            "temperature": 25,
            "condition": "sunny"
        }
        """);

    var httpClient = new HttpClient(mockHttp);
    var service = new WeatherService(httpClient);

    var weather = await service.GetCurrentWeatherAsync();

    Assert.Equal(25, weather.Temperature);
}
```

### 수동 Handler 구현

```csharp
public class FakeHttpMessageHandler : HttpMessageHandler
{
    private readonly Dictionary<string, HttpResponseMessage> _responses = new();

    public void SetupResponse(string url, HttpResponseMessage response)
    {
        _responses[url] = response;
    }

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        if (_responses.TryGetValue(request.RequestUri!.ToString(), out var response))
        {
            return Task.FromResult(response);
        }

        return Task.FromResult(new HttpResponseMessage(HttpStatusCode.NotFound));
    }
}
```

---

## 시간 Mocking

### TimeProvider (.NET 8+)

```csharp
public class TokenService
{
    private readonly TimeProvider _timeProvider;

    public TokenService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }

    public bool IsTokenExpired(Token token)
    {
        return _timeProvider.GetUtcNow() > token.ExpiresAt;
    }
}

[Fact]
public void IsTokenExpired_ExpiredToken_ReturnsTrue()
{
    var fakeTime = new FakeTimeProvider(
        new DateTimeOffset(2024, 1, 15, 12, 0, 0, TimeSpan.Zero));

    var service = new TokenService(fakeTime);
    var token = new Token
    {
        ExpiresAt = new DateTimeOffset(2024, 1, 15, 10, 0, 0, TimeSpan.Zero)
    };

    Assert.True(service.IsTokenExpired(token));
}
```

### 커스텀 IClock 인터페이스 (이전 버전)

```csharp
public interface IClock
{
    DateTime UtcNow { get; }
    DateTimeOffset OffsetUtcNow { get; }
}

public class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
    public DateTimeOffset OffsetUtcNow => DateTimeOffset.UtcNow;
}

public class FakeClock : IClock
{
    public DateTime UtcNow { get; set; } = DateTime.UtcNow;
    public DateTimeOffset OffsetUtcNow { get; set; } = DateTimeOffset.UtcNow;

    public void Advance(TimeSpan duration) => UtcNow = UtcNow.Add(duration);
}
```

---

## 면접 질문

### Q1: Mock과 Stub의 차이점은?
**A**:
- **Stub**: 미리 정해진 값을 반환 (입력 대체)
- **Mock**: 호출 여부와 방식을 검증 (출력 검증)

```csharp
// Stub - 값 반환에 집중
mock.Setup(s => s.GetUser(1)).Returns(new User());

// Mock - 호출 검증에 집중
mock.Verify(s => s.SendEmail(It.IsAny<string>()), Times.Once);
```

### Q2: 언제 Mock을 사용하고 언제 Fake를 사용하나요?
**A**:
- **Mock**: 외부 시스템(이메일, 결제, 외부 API) - 호출 여부만 중요
- **Fake**: 내부 시스템(캐시, 저장소) - 실제 동작 검증 필요

### Q3: Over-mocking이란?
**A**: 너무 많은 의존성을 Mock하여 테스트가 구현 세부사항에 의존하게 되는 것입니다. 리팩토링하면 테스트가 깨지고, 실제 버그를 발견하지 못할 수 있습니다. 해결책은 행위가 아닌 결과를 테스트하는 것입니다.
