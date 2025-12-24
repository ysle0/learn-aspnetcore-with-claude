# Testing in ASP.NET Core

ASP.NET Core 애플리케이션의 테스팅 전략과 구현 방법을 다룹니다.

## 📚 목차

### 기초
| 문서 | 설명 |
|------|------|
| [Unit Testing 기초](./unit-testing-fundamentals.md) | 단위 테스트의 원칙과 구조 |
| [Test Frameworks 비교](./test-frameworks.md) | xUnit, NUnit, MSTest 비교 |
| [Mocking과 Test Doubles](./mocking-test-doubles.md) | Moq, NSubstitute, Fake 패턴 |

### 통합 테스트 (Integration/Acceptance Testing)
| 문서 | 설명 |
|------|------|
| [Integration Testing 기초](./integration-testing-fundamentals.md) | WebApplicationFactory와 TestServer |
| [HTTP API 테스트](./http-api-testing.md) | REST API 인수 테스트 |
| [WebSocket 테스트](./websocket-testing.md) | SignalR 및 WebSocket 테스트 |
| [gRPC 테스트](./grpc-testing.md) | gRPC 서비스 테스트 |

### 고급 주제
| 문서 | 설명 |
|------|------|
| [Testing Best Practices](./best-practices.md) | 테스트 설계 원칙과 패턴 |

---

## 🎯 테스트 피라미드

```
                    ┌─────────┐
                    │   E2E   │  ← 느리고 비용 높음
                   ─┴─────────┴─
                  │ Integration │  ← WebApplicationFactory
                 ─┴─────────────┴─
                │   Unit Tests    │  ← 빠르고 격리됨
               ─┴─────────────────┴─
```

| 테스트 유형 | 범위 | 속도 | 신뢰도 |
|------------|------|------|--------|
| Unit | 단일 클래스/메서드 | ⚡ 매우 빠름 | 낮음 (격리됨) |
| Integration | 여러 컴포넌트 | 🔄 보통 | 높음 |
| E2E | 전체 시스템 | 🐢 느림 | 매우 높음 |

---

## 🔧 ASP.NET Core 테스트 도구

### Microsoft 공식
```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
```

### 테스트 프레임워크
```xml
<!-- xUnit (권장) -->
<PackageReference Include="xunit" Version="2.6.2" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.4" />

<!-- NUnit -->
<PackageReference Include="NUnit" Version="4.0.1" />
<PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
```

### Mocking 라이브러리
```xml
<PackageReference Include="Moq" Version="4.20.70" />
<!-- 또는 -->
<PackageReference Include="NSubstitute" Version="5.1.0" />
```

### 추가 도구
```xml
<PackageReference Include="FluentAssertions" Version="6.12.0" />
<PackageReference Include="Bogus" Version="35.0.1" />
<PackageReference Include="Testcontainers" Version="3.6.0" />
```

---

## 📖 핵심 개념 미리보기

### WebApplicationFactory
```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 테스트용 서비스 교체
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContext));

            if (descriptor != null)
                services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("TestDb"));
        });
    }
}
```

### 기본 통합 테스트 구조
```csharp
public class ApiTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ApiTests(CustomWebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Get_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/values");

        // Assert
        response.EnsureSuccessStatusCode();
    }
}
```

---

## 📚 참고 자료

### 공식 문서
- [ASP.NET Core에서 통합 테스트](https://learn.microsoft.com/aspnet/core/test/integration-tests)
- [ASP.NET Core에서 컨트롤러 테스트](https://learn.microsoft.com/aspnet/core/mvc/controllers/testing)

### 추천 도서
- **"Unit Testing Principles, Practices, and Patterns"** - Vladimir Khorikov
- **"The Art of Unit Testing"** - Roy Osherove
- **"xUnit Test Patterns"** - Gerard Meszaros
- **"Growing Object-Oriented Software, Guided by Tests"** - Steve Freeman

### 관련 섹션
- [00-best-practices](../00-best-practices/) - 테스트 가능한 코드 설계
- [04-dependency-injection](../04-dependency-injection/) - DI와 테스트 가능성
