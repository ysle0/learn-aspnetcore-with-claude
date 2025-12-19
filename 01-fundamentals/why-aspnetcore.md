# ASP.NET Core를 왜 이렇게 만들었는가

## 개요

ASP.NET Core는 2016년에 출시된 Microsoft의 차세대 웹 프레임워크입니다. 기존 ASP.NET의 한계를 극복하고, 현대적인 웹 개발 요구사항을 충족하기 위해 **완전히 새로 설계**되었습니다.

---

## 역사적 배경

### ASP.NET의 문제점 (2002-2015)

```
┌─────────────────────────────────────────────────────────────┐
│                    ASP.NET (Classic)                        │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐                                           │
│  │  System.Web  │ ← 모든 것이 여기에 의존                    │
│  │  (거대한 DLL) │                                          │
│  └──────────────┘                                           │
│         ↓                                                   │
│  ┌──────────────┐                                           │
│  │     IIS      │ ← Windows Server 필수                     │
│  └──────────────┘                                           │
│         ↓                                                   │
│  ┌──────────────┐                                           │
│  │   Windows    │ ← 크로스 플랫폼 불가                       │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

**주요 문제점:**

1. **System.Web 의존성**: 200개 이상의 어셈블리가 System.Web에 강하게 결합
2. **IIS 종속**: 다른 웹 서버 사용 불가
3. **Windows 전용**: Linux, macOS에서 실행 불가
4. **느린 릴리스 주기**: .NET Framework와 함께 릴리스 (2-3년)
5. **무거운 프레임워크**: 필요 없는 기능도 모두 포함

### 시대적 요구사항 (2010년대)

| 트렌드 | 요구사항 |
|--------|---------|
| 클라우드 컴퓨팅 | Linux 서버 지원 필요 |
| 마이크로서비스 | 경량화, 빠른 시작 시간 |
| 컨테이너 (Docker) | 작은 이미지 크기 |
| 오픈소스 | 커뮤니티 기여, 투명성 |

---

## ASP.NET Core의 설계 철학

### 1. 모듈러 설계

```csharp
// 필요한 것만 추가 (NuGet 패키지)
var builder = WebApplication.CreateBuilder(args);

// 필요한 서비스만 등록
builder.Services.AddControllers();        // MVC 컨트롤러가 필요할 때만
builder.Services.AddAuthentication();     // 인증이 필요할 때만
builder.Services.AddCors();               // CORS가 필요할 때만
```

**기존 ASP.NET:**
```csharp
// 모든 것이 기본 포함 - 선택 불가
// System.Web.dll 하나에 모든 것이 포함
```

### 2. 크로스 플랫폼

```
ASP.NET Core 앱
      │
      ▼
┌─────────────────┐
│   .NET Runtime  │
└─────────────────┘
      │
      ├──── Windows
      ├──── Linux (Ubuntu, CentOS, Alpine...)
      └──── macOS
```

**어떻게 가능해졌나?**

- .NET Core 런타임이 각 OS별로 구현됨
- System.Web 의존성 완전 제거
- Kestrel: 순수 .NET으로 작성된 웹 서버

### 3. 고성능

```
TechEmpower Benchmark (Round 21)
─────────────────────────────────
ASP.NET Core    │████████████████████████│ 7,000,000+ req/s
Node.js         │██████████              │ 300,000 req/s
Spring Boot     │████████                │ 250,000 req/s
```

**성능 향상 요인:**

- Kestrel의 비동기 I/O
- Span<T>, Memory<T> 활용 (Zero-copy)
- System.Text.Json (고성능 JSON 직렬화)
- HTTP/2, HTTP/3 지원

### 4. 의존성 주입 내장

```csharp
// ASP.NET Core - DI가 프레임워크에 내장
public class MyController : ControllerBase
{
    private readonly IMyService _service;

    // 생성자 주입 - 프레임워크가 자동 처리
    public MyController(IMyService service)
    {
        _service = service;
    }
}

// 서비스 등록
builder.Services.AddScoped<IMyService, MyService>();
```

**기존 ASP.NET:**
```csharp
// 별도 DI 컨테이너 필요 (Ninject, Autofac, Unity 등)
// 설정이 복잡하고 일관성 없음
```

### 5. 미들웨어 파이프라인

```csharp
var app = builder.Build();

// 요청 처리 파이프라인을 명시적으로 구성
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

```
요청 → [HTTPS] → [Static] → [Routing] → [Auth] → [AuthZ] → [Controller]
응답 ← [HTTPS] ← [Static] ← [Routing] ← [Auth] ← [AuthZ] ← [Controller]
```

---

## 버전별 주요 변화

| 버전 | 출시 | 주요 변경 |
|------|------|----------|
| ASP.NET Core 1.0 | 2016 | 최초 릴리스, 크로스 플랫폼 |
| ASP.NET Core 2.0 | 2017 | Razor Pages, API 개선 |
| ASP.NET Core 3.0 | 2019 | Blazor, gRPC, Worker Service |
| .NET 5 | 2020 | 통합 (.NET Core + .NET Framework) |
| .NET 6 | 2021 | **Minimal APIs**, Hot Reload |
| .NET 7 | 2022 | Rate Limiting, Output Caching |
| .NET 8 | 2023 | Native AOT, Blazor United |
| .NET 9 | 2024 | HybridCache, 성능 개선 |

### Minimal APIs의 등장 (.NET 6)

```csharp
// 이전: Controllers 필수
[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpGet]
    public IEnumerable<Weather> Get() => ...
}

// .NET 6+: Minimal API
app.MapGet("/weather", () => new[] { new Weather() });
```

---

## ASP.NET Core vs 다른 프레임워크

| 특성 | ASP.NET Core | Node.js (Express) | Spring Boot |
|------|-------------|-------------------|-------------|
| 언어 | C# | JavaScript | Java |
| 타입 | 정적 | 동적 | 정적 |
| 성능 | 매우 높음 | 중간 | 높음 |
| 메모리 | 효율적 | GC 의존 | 높은 사용량 |
| 크로스 플랫폼 | ✅ | ✅ | ✅ |
| 내장 DI | ✅ | ❌ | ✅ |
| 학습 곡선 | 중간 | 낮음 | 높음 |

---

## 예시: 동일한 API를 각 버전으로

### ASP.NET (Classic) - WebForms 시절

```csharp
// Global.asax
protected void Application_Start()
{
    GlobalConfiguration.Configure(WebApiConfig.Register);
}

// WebApiConfig.cs
public static void Register(HttpConfiguration config)
{
    config.Routes.MapHttpRoute(
        name: "DefaultApi",
        routeTemplate: "api/{controller}/{id}",
        defaults: new { id = RouteParameter.Optional }
    );
}

// ProductsController.cs
public class ProductsController : ApiController
{
    public IEnumerable<Product> GetAllProducts()
    {
        return _repository.GetAll();
    }
}
```

### ASP.NET Core - Controller 기반

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();

// ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IEnumerable<Product> GetAll() => _repository.GetAll();
}
```

### ASP.NET Core - Minimal API

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/products", () => _repository.GetAll());

app.Run();
```

---

## 면접 예상 질문

### Q1: ASP.NET과 ASP.NET Core의 가장 큰 차이점은?

**A:**
- **크로스 플랫폼**: ASP.NET Core는 Windows, Linux, macOS에서 실행 가능
- **모듈러 설계**: 필요한 기능만 NuGet으로 추가
- **고성능**: Kestrel 웹 서버, 비동기 처리 최적화
- **오픈소스**: GitHub에서 개발, 커뮤니티 기여 가능

### Q2: ASP.NET Core가 고성능인 이유는?

**A:**
1. **Kestrel**: libuv/IO Completion Ports 기반 비동기 I/O
2. **Span<T>/Memory<T>**: 메모리 할당 최소화
3. **System.Text.Json**: 고성능 JSON 직렬화
4. **HTTP/2, HTTP/3**: 멀티플렉싱, QUIC 프로토콜
5. **JIT/AOT 최적화**: .NET 런타임 성능 향상

### Q3: 왜 System.Web을 버렸나요?

**A:**
- Windows 전용 API에 강하게 결합
- IIS에 종속적인 설계
- 모놀리식 구조로 모듈화 불가능
- 크로스 플랫폼 지원을 위해 완전한 재설계 필요

---

## 참고 자료

- [ASP.NET Core 공식 문서](https://learn.microsoft.com/aspnet/core)
- [.NET Blog - ASP.NET Core 발표](https://devblogs.microsoft.com/dotnet/)
- [GitHub - dotnet/aspnetcore](https://github.com/dotnet/aspnetcore)
- [TechEmpower Benchmarks](https://www.techempower.com/benchmarks/)

---

## 다음 문서

→ [project-structure.md](./project-structure.md) - 프로젝트 구조 이해하기
