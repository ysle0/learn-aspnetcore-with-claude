# Test Frameworks 비교

.NET 생태계의 주요 테스트 프레임워크를 비교합니다.

## 프레임워크 비교 요약

| 특성 | xUnit | NUnit | MSTest |
|------|-------|-------|--------|
| **출시** | 2007 | 2002 | 2005 |
| **유지보수** | 커뮤니티 | 커뮤니티 | Microsoft |
| **ASP.NET Core 권장** | ✅ | ✅ | ✅ |
| **병렬 실행** | 기본 활성화 | 설정 필요 | 설정 필요 |
| **테스트 격리** | 클래스당 새 인스턴스 | 재사용 | 재사용 |
| **확장성** | 높음 | 높음 | 보통 |

---

## xUnit (권장)

ASP.NET Core 팀이 내부적으로 사용하는 프레임워크입니다.

### 설치
```xml
<PackageReference Include="xunit" Version="2.6.2" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.4" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
```

### 기본 문법
```csharp
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_TwoNumbers_ReturnsSum()
    {
        var calc = new Calculator();
        Assert.Equal(4, calc.Add(2, 2));
    }

    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(0, 0, 0)]
    [InlineData(-1, 1, 0)]
    public void Add_VariousInputs_ReturnsExpected(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.Equal(expected, calc.Add(a, b));
    }
}
```

### xUnit Assertions
```csharp
// 동등성
Assert.Equal(expected, actual);
Assert.NotEqual(expected, actual);

// 참조
Assert.Same(expected, actual);    // ReferenceEquals
Assert.NotSame(expected, actual);

// Null 체크
Assert.Null(obj);
Assert.NotNull(obj);

// Boolean
Assert.True(condition);
Assert.False(condition);

// 컬렉션
Assert.Empty(collection);
Assert.NotEmpty(collection);
Assert.Contains(item, collection);
Assert.DoesNotContain(item, collection);
Assert.Single(collection);
Assert.All(collection, item => Assert.NotNull(item));

// 타입
Assert.IsType<ExpectedType>(obj);
Assert.IsAssignableFrom<BaseType>(obj);

// 예외
Assert.Throws<ArgumentException>(() => method());
await Assert.ThrowsAsync<InvalidOperationException>(() => asyncMethod());

// 범위
Assert.InRange(value, low, high);

// 문자열
Assert.StartsWith("prefix", text);
Assert.EndsWith("suffix", text);
Assert.Contains("substring", text);
Assert.Matches(regex, text);
```

### Fixture (공유 리소스)
```csharp
// 테스트 클래스당 한 번 (IClassFixture)
public class DatabaseFixture : IDisposable
{
    public SqlConnection Connection { get; }

    public DatabaseFixture()
    {
        Connection = new SqlConnection("...");
        Connection.Open();
    }

    public void Dispose() => Connection.Dispose();
}

public class DatabaseTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;

    public DatabaseTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void Query_Works()
    {
        // _fixture.Connection 사용
    }
}

// 여러 클래스에서 공유 (ICollectionFixture)
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class Tests1
{
    public Tests1(DatabaseFixture fixture) { }
}

[Collection("Database")]
public class Tests2
{
    public Tests2(DatabaseFixture fixture) { }
}
```

### 병렬 실행 제어
```csharp
// 같은 Collection 내에서는 순차 실행
[Collection("Sequential")]
public class Test1 { }

[Collection("Sequential")]
public class Test2 { }

// xunit.runner.json으로 전역 설정
{
    "parallelizeAssembly": false,
    "parallelizeTestCollections": true,
    "maxParallelThreads": 4
}
```

---

## NUnit

가장 오래된 .NET 테스트 프레임워크입니다.

### 설치
```xml
<PackageReference Include="NUnit" Version="4.0.1" />
<PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
```

### 기본 문법
```csharp
using NUnit.Framework;

[TestFixture]
public class CalculatorTests
{
    private Calculator _calc;

    [SetUp]
    public void Setup()
    {
        _calc = new Calculator();
    }

    [TearDown]
    public void TearDown()
    {
        // 정리 로직
    }

    [Test]
    public void Add_TwoNumbers_ReturnsSum()
    {
        Assert.That(_calc.Add(2, 2), Is.EqualTo(4));
    }

    [TestCase(1, 2, 3)]
    [TestCase(0, 0, 0)]
    [TestCase(-1, 1, 0)]
    public void Add_VariousInputs_ReturnsExpected(int a, int b, int expected)
    {
        Assert.That(_calc.Add(a, b), Is.EqualTo(expected));
    }
}
```

### NUnit Assertions (Constraint Model)
```csharp
// 기본
Assert.That(actual, Is.EqualTo(expected));
Assert.That(actual, Is.Not.EqualTo(unexpected));

// Null
Assert.That(obj, Is.Null);
Assert.That(obj, Is.Not.Null);

// Boolean
Assert.That(condition, Is.True);
Assert.That(condition, Is.False);

// 비교
Assert.That(value, Is.GreaterThan(5));
Assert.That(value, Is.LessThanOrEqualTo(10));
Assert.That(value, Is.InRange(1, 10));

// 컬렉션
Assert.That(collection, Is.Empty);
Assert.That(collection, Is.Not.Empty);
Assert.That(collection, Has.Count.EqualTo(3));
Assert.That(collection, Contains.Item("value"));
Assert.That(collection, Has.All.GreaterThan(0));
Assert.That(collection, Is.Ordered);

// 문자열
Assert.That(text, Does.StartWith("prefix"));
Assert.That(text, Does.Contain("substring"));
Assert.That(text, Does.Match("regex"));

// 타입
Assert.That(obj, Is.TypeOf<ExpectedType>());
Assert.That(obj, Is.InstanceOf<BaseType>());

// 예외
Assert.Throws<ArgumentException>(() => method());
Assert.ThrowsAsync<InvalidOperationException>(async () => await asyncMethod());

// 복합 조건
Assert.That(value, Is.GreaterThan(5).And.LessThan(10));
Assert.That(text, Is.Not.Null.And.Not.Empty);
```

### Setup/Teardown 계층
```csharp
[TestFixture]
public class Tests
{
    [OneTimeSetUp]
    public void OneTimeSetUp()
    {
        // 모든 테스트 전 한 번
    }

    [SetUp]
    public void SetUp()
    {
        // 각 테스트 전
    }

    [TearDown]
    public void TearDown()
    {
        // 각 테스트 후
    }

    [OneTimeTearDown]
    public void OneTimeTearDown()
    {
        // 모든 테스트 후 한 번
    }
}
```

---

## MSTest

Microsoft 공식 테스트 프레임워크입니다.

### 설치
```xml
<PackageReference Include="MSTest.TestFramework" Version="3.1.1" />
<PackageReference Include="MSTest.TestAdapter" Version="3.1.1" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
```

### 기본 문법
```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CalculatorTests
{
    private Calculator _calc;

    [TestInitialize]
    public void Initialize()
    {
        _calc = new Calculator();
    }

    [TestCleanup]
    public void Cleanup()
    {
        // 정리 로직
    }

    [TestMethod]
    public void Add_TwoNumbers_ReturnsSum()
    {
        Assert.AreEqual(4, _calc.Add(2, 2));
    }

    [DataTestMethod]
    [DataRow(1, 2, 3)]
    [DataRow(0, 0, 0)]
    [DataRow(-1, 1, 0)]
    public void Add_VariousInputs_ReturnsExpected(int a, int b, int expected)
    {
        Assert.AreEqual(expected, _calc.Add(a, b));
    }
}
```

### MSTest Assertions
```csharp
// 동등성
Assert.AreEqual(expected, actual);
Assert.AreNotEqual(expected, actual);

// 참조
Assert.AreSame(expected, actual);
Assert.AreNotSame(expected, actual);

// Null
Assert.IsNull(obj);
Assert.IsNotNull(obj);

// Boolean
Assert.IsTrue(condition);
Assert.IsFalse(condition);

// 타입
Assert.IsInstanceOfType(obj, typeof(ExpectedType));

// 예외
Assert.ThrowsException<ArgumentException>(() => method());
await Assert.ThrowsExceptionAsync<InvalidOperationException>(() => asyncMethod());

// 문자열
StringAssert.Contains(text, "substring");
StringAssert.StartsWith(text, "prefix");
StringAssert.Matches(text, regex);

// 컬렉션
CollectionAssert.Contains(collection, item);
CollectionAssert.AreEqual(expected, actual);
CollectionAssert.AreEquivalent(expected, actual);
```

---

## FluentAssertions (권장 추가 도구)

모든 프레임워크와 함께 사용 가능한 Fluent API입니다.

### 설치
```xml
<PackageReference Include="FluentAssertions" Version="6.12.0" />
```

### 사용 예시
```csharp
using FluentAssertions;

[Fact]
public void FluentAssertions_Examples()
{
    // 기본 타입
    var value = 5;
    value.Should().Be(5);
    value.Should().BeGreaterThan(3);
    value.Should().BeInRange(1, 10);

    // 문자열
    var text = "Hello World";
    text.Should().StartWith("Hello");
    text.Should().Contain("World");
    text.Should().HaveLength(11);
    text.Should().MatchRegex(@"Hello \w+");

    // 컬렉션
    var list = new[] { 1, 2, 3 };
    list.Should().HaveCount(3);
    list.Should().Contain(2);
    list.Should().BeInAscendingOrder();
    list.Should().OnlyContain(x => x > 0);
    list.Should().BeEquivalentTo(new[] { 3, 2, 1 }); // 순서 무관

    // 객체
    var user = new User { Name = "John", Age = 30 };
    user.Should().BeEquivalentTo(new { Name = "John", Age = 30 });
    user.Name.Should().NotBeNullOrEmpty();

    // 예외
    Action act = () => throw new ArgumentException("Invalid");
    act.Should().Throw<ArgumentException>()
        .WithMessage("*Invalid*");

    // 비동기
    Func<Task> asyncAct = async () => throw new InvalidOperationException();
    await asyncAct.Should().ThrowAsync<InvalidOperationException>();

    // 실행 시간
    Action slowAction = () => Thread.Sleep(100);
    slowAction.ExecutionTime().Should().BeLessThan(TimeSpan.FromSeconds(1));

    // DateTime
    var date = DateTime.Now;
    date.Should().BeAfter(DateTime.MinValue);
    date.Should().BeSameDateAs(DateTime.Today);
    date.Should().BeCloseTo(DateTime.Now, TimeSpan.FromSeconds(1));
}
```

### Object Graph 비교
```csharp
[Fact]
public void ComplexObject_Comparison()
{
    var expected = new Order
    {
        Id = 1,
        Customer = new Customer { Name = "John" },
        Items = new[] { new OrderItem { ProductId = 100 } }
    };

    var actual = GetOrder();

    // 깊은 비교
    actual.Should().BeEquivalentTo(expected);

    // 일부 필드 제외
    actual.Should().BeEquivalentTo(expected, options => options
        .Excluding(o => o.Id)
        .Excluding(o => o.CreatedAt));

    // 컬렉션 순서 무시
    actual.Items.Should().BeEquivalentTo(expected.Items,
        options => options.WithoutStrictOrdering());
}
```

---

## 프레임워크 선택 가이드

### xUnit 선택 시
- ✅ ASP.NET Core 프로젝트 (공식 권장)
- ✅ 새 프로젝트
- ✅ 병렬 테스트가 중요한 경우
- ✅ 테스트 격리가 중요한 경우

### NUnit 선택 시
- ✅ 기존 NUnit 프로젝트 마이그레이션
- ✅ 풍부한 Constraint 문법 선호
- ✅ 테스트 순서 제어가 필요한 경우

### MSTest 선택 시
- ✅ Visual Studio 통합이 최우선
- ✅ Microsoft 지원이 필요한 엔터프라이즈
- ✅ 기존 MSTest 프로젝트

---

## 속성 비교표

| xUnit | NUnit | MSTest | 설명 |
|-------|-------|--------|------|
| `[Fact]` | `[Test]` | `[TestMethod]` | 테스트 메서드 |
| `[Theory]` | `[TestCase]` | `[DataTestMethod]` | 파라미터화 테스트 |
| `[InlineData]` | `[TestCase]` | `[DataRow]` | 테스트 데이터 |
| Constructor | `[SetUp]` | `[TestInitialize]` | 테스트 전 실행 |
| `IDisposable` | `[TearDown]` | `[TestCleanup]` | 테스트 후 실행 |
| `IClassFixture<>` | `[OneTimeSetUp]` | `[ClassInitialize]` | 클래스당 한 번 |
| `[Collection]` | `[Category]` | `[TestCategory]` | 테스트 그룹화 |
| `[Trait]` | `[Property]` | `[TestProperty]` | 메타데이터 |
| `[Skip]` | `[Ignore]` | `[Ignore]` | 테스트 건너뛰기 |

---

## 면접 질문

### Q1: xUnit에서 테스트마다 새 인스턴스가 생성되는 이유는?
**A**: 테스트 격리를 보장하기 위해서입니다. 각 테스트가 독립적으로 실행되어 테스트 순서나 다른 테스트의 상태에 영향받지 않습니다.

### Q2: Fact와 Theory의 차이점은?
**A**: `[Fact]`는 단일 케이스 테스트이고, `[Theory]`는 `[InlineData]` 등을 통해 여러 입력값으로 같은 테스트를 반복 실행합니다.

### Q3: FluentAssertions를 사용하는 이유는?
**A**:
1. 읽기 쉬운 Fluent 문법
2. 실패 시 더 명확한 오류 메시지
3. 복잡한 객체 비교 기능
4. 모든 테스트 프레임워크와 호환
