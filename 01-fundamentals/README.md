# 01. Fundamentals - ASP.NET Core 기초

ASP.NET Core의 탄생 배경, 설계 철학, 그리고 프로젝트 구조를 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [why-aspnetcore.md](./why-aspnetcore.md) | ASP.NET Core를 왜 이렇게 만들었는가 | 표면 → 역사 |
| [project-structure.md](./project-structure.md) | 프로젝트 구조 (csproj, slnx) | 표면 → MSBuild |
| [hosting-models.md](./hosting-models.md) | 호스팅 모델 | 표면 → 프로세스 |

---

## 핵심 요약

### ASP.NET Core란?

ASP.NET Core는 Microsoft가 개발한 **크로스 플랫폼**, **고성능**, **오픈소스** 웹 프레임워크입니다.

```
ASP.NET (2002) → ASP.NET MVC (2009) → ASP.NET Core (2016) → .NET 9 (2024)
     ↓                   ↓                    ↓
  WebForms          MVC 패턴           크로스플랫폼
  Windows Only      Windows Only       Windows/Linux/macOS
```

### 왜 ASP.NET Core를 만들었나?

| 기존 ASP.NET | ASP.NET Core |
|-------------|--------------|
| Windows 전용 | 크로스 플랫폼 |
| System.Web 의존 | 모듈러 설계 |
| IIS 필수 | Kestrel 내장 |
| 무거운 프레임워크 | 경량화 |

### 프로젝트 구조 한눈에 보기

```
MyApp/
├── MyApp.sln              # 솔루션 파일 (또는 .slnx)
├── MyApp.csproj           # 프로젝트 파일
├── Program.cs             # 진입점
├── appsettings.json       # 설정
├── appsettings.Development.json
├── Properties/
│   └── launchSettings.json
└── wwwroot/               # 정적 파일
```

### 최소 ASP.NET Core 앱

```csharp
// Program.cs - .NET 6+ Minimal API
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello, ASP.NET Core!");

app.Run();
```

---

## 면접 빈출 질문

1. **ASP.NET과 ASP.NET Core의 차이점은?**
2. **ASP.NET Core가 크로스 플랫폼을 지원하는 이유는?**
3. **Program.cs에서 WebApplication.CreateBuilder()는 무엇을 하나요?**
4. **.csproj 파일의 역할은?**

---

## 다음 단계

→ [02. Server Infrastructure](../02-server-infrastructure/) - Kestrel과 서버 인프라 이해하기
