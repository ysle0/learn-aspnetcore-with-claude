# 프로젝트 구조 (csproj, slnx)

## 개요

ASP.NET Core 프로젝트는 **SDK 스타일** 프로젝트 파일(.csproj)을 사용합니다. 기존 .NET Framework의 복잡한 XML 구조에서 벗어나 간결하고 읽기 쉬운 형태로 진화했습니다.

---

## 기본 프로젝트 구조

```
MyWebApp/
├── MyWebApp.sln                    # 솔루션 파일
├── src/
│   └── MyWebApp/
│       ├── MyWebApp.csproj         # 프로젝트 파일
│       ├── Program.cs              # 앱 진입점
│       ├── appsettings.json        # 기본 설정
│       ├── appsettings.Development.json
│       ├── appsettings.Production.json
│       ├── Properties/
│       │   └── launchSettings.json # 개발 실행 설정
│       ├── Controllers/            # API 컨트롤러
│       ├── Models/                 # 도메인 모델
│       ├── Services/               # 비즈니스 로직
│       └── wwwroot/                # 정적 파일
│           ├── css/
│           ├── js/
│           └── images/
└── tests/
    └── MyWebApp.Tests/
        └── MyWebApp.Tests.csproj
```

---

## .csproj 파일 상세

### 최소 구성

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

### 주요 속성 설명

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <!-- SDK: 프로젝트 타입 결정 -->
  <!-- Microsoft.NET.Sdk       : 콘솔/라이브러리 -->
  <!-- Microsoft.NET.Sdk.Web   : 웹 애플리케이션 -->
  <!-- Microsoft.NET.Sdk.Worker: 백그라운드 서비스 -->
  <!-- Microsoft.NET.Sdk.Razor : Razor 클래스 라이브러리 -->

  <PropertyGroup>
    <!-- 대상 프레임워크 -->
    <TargetFramework>net9.0</TargetFramework>

    <!-- 멀티 타겟팅 -->
    <TargetFrameworks>net8.0;net9.0</TargetFrameworks>

    <!-- Nullable 참조 타입 활성화 -->
    <Nullable>enable</Nullable>

    <!-- 암시적 using 지시문 -->
    <ImplicitUsings>enable</ImplicitUsings>

    <!-- 출력 타입 -->
    <OutputType>Exe</OutputType>

    <!-- 어셈블리 이름 -->
    <AssemblyName>MyWebApp</AssemblyName>

    <!-- 루트 네임스페이스 -->
    <RootNamespace>MyWebApp</RootNamespace>

    <!-- 버전 정보 -->
    <Version>1.0.0</Version>
    <AssemblyVersion>1.0.0.0</AssemblyVersion>

    <!-- Docker 지원 -->
    <DockerDefaultTargetOS>Linux</DockerDefaultTargetOS>

    <!-- AOT 컴파일 -->
    <PublishAot>true</PublishAot>
  </PropertyGroup>

  <!-- NuGet 패키지 참조 -->
  <ItemGroup>
    <PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.0" />
  </ItemGroup>

  <!-- 프로젝트 참조 -->
  <ItemGroup>
    <ProjectReference Include="..\MyWebApp.Core\MyWebApp.Core.csproj" />
  </ItemGroup>

</Project>
```

### ImplicitUsings 활성화 시 자동 포함되는 네임스페이스

```csharp
// Microsoft.NET.Sdk.Web 사용 시 자동 포함
global using global::Microsoft.AspNetCore.Builder;
global using global::Microsoft.AspNetCore.Hosting;
global using global::Microsoft.AspNetCore.Http;
global using global::Microsoft.AspNetCore.Routing;
global using global::Microsoft.Extensions.Configuration;
global using global::Microsoft.Extensions.DependencyInjection;
global using global::Microsoft.Extensions.Hosting;
global using global::Microsoft.Extensions.Logging;
global using global::System;
global using global::System.Collections.Generic;
global using global::System.IO;
global using global::System.Linq;
global using global::System.Net.Http;
global using global::System.Threading;
global using global::System.Threading.Tasks;
```

---

## .slnx (새로운 솔루션 파일 형식)

### 기존 .sln vs 새로운 .slnx

**.sln (기존)**
```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 17
VisualStudioVersion = 17.0.31903.59
MinimumVisualStudioVersion = 10.0.40219.1
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "MyApp", "MyApp\MyApp.csproj", "{GUID}"
EndProject
Global
    GlobalSection(SolutionConfigurationPlatforms) = preSolution
        Debug|Any CPU = Debug|Any CPU
        Release|Any CPU = Release|Any CPU
    EndGlobalSection
    ...
EndGlobal
```

**.slnx (새로운 XML 기반 - .NET 9+)**
```xml
<Solution>
  <Project Path="src/MyApp/MyApp.csproj" />
  <Project Path="tests/MyApp.Tests/MyApp.Tests.csproj" />

  <Folder Name="/docs/">
    <File Path="README.md" />
  </Folder>

  <Configuration Name="Debug" />
  <Configuration Name="Release" />
</Solution>
```

### .slnx 장점

| 특성 | .sln | .slnx |
|------|------|-------|
| 형식 | 커스텀 텍스트 | XML |
| 가독성 | 낮음 | 높음 |
| 병합 충돌 | 잦음 | 적음 |
| 편집 | VS 필요 | 텍스트 에디터 OK |
| GUID 관리 | 필수 | 자동 |

---

## Program.cs 진화

### .NET 5 이전 (Startup.cs 분리)

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Startup.cs
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

### .NET 6+ (통합된 Program.cs)

```csharp
// Program.cs - 모든 것이 하나의 파일에
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록 (이전의 ConfigureServices)
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// 미들웨어 구성 (이전의 Configure)
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Top-Level Statements

```csharp
// Main 메서드 없이 바로 코드 작성
// 컴파일러가 자동으로 Main 메서드 생성
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();

// 컴파일러가 생성하는 실제 코드
// internal class Program
// {
//     private static void Main(string[] args) { ... }
// }
```

---

## appsettings.json 구조

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",

  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;..."
  },

  "AppSettings": {
    "ApiKey": "your-api-key",
    "MaxPageSize": 100
  }
}
```

### 환경별 설정 오버라이드

```
설정 로드 순서 (나중이 우선):
1. appsettings.json
2. appsettings.{Environment}.json
3. 환경 변수
4. 명령줄 인수
5. User Secrets (개발 환경)
```

```csharp
// 환경 변수 예시
// ConnectionStrings__DefaultConnection="Server=prod;..."
// AppSettings__ApiKey="prod-api-key"

// 설정 바인딩
builder.Services.Configure<AppSettings>(
    builder.Configuration.GetSection("AppSettings"));
```

---

## launchSettings.json

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "Docker": {
      "commandName": "Docker",
      "launchBrowser": true,
      "launchUrl": "{Scheme}://{ServiceHost}:{ServicePort}",
      "publishAllPorts": true
    }
  }
}
```

---

## 디렉토리 구조 패턴

### Feature-based (기능별)

```
src/MyApp/
├── Features/
│   ├── Users/
│   │   ├── UsersController.cs
│   │   ├── UserService.cs
│   │   ├── UserDto.cs
│   │   └── UserRepository.cs
│   ├── Products/
│   │   ├── ProductsController.cs
│   │   ├── ProductService.cs
│   │   └── ...
│   └── Orders/
│       └── ...
└── Shared/
    ├── Exceptions/
    └── Extensions/
```

### Layer-based (계층별)

```
src/MyApp/
├── Controllers/
│   ├── UsersController.cs
│   └── ProductsController.cs
├── Services/
│   ├── IUserService.cs
│   ├── UserService.cs
│   └── ...
├── Repositories/
│   └── ...
├── Models/
│   ├── Entities/
│   └── DTOs/
└── Infrastructure/
    └── ...
```

### Clean Architecture

```
src/
├── MyApp.Domain/           # 엔티티, 값 객체, 도메인 서비스
├── MyApp.Application/      # Use cases, DTOs, 인터페이스
├── MyApp.Infrastructure/   # DB, 외부 서비스 구현
└── MyApp.WebApi/           # Controllers, 미들웨어
```

---

## 면접 예상 질문

### Q1: SDK 스타일 프로젝트의 장점은?

**A:**
- 간결한 XML 구조 (수백 줄 → 수십 줄)
- 와일드카드 파일 포함 (*.cs 자동 포함)
- 멀티 타겟팅 쉬움
- NuGet 패키지 참조가 직관적
- Git 병합 충돌 감소

### Q2: appsettings.json의 로드 순서는?

**A:**
1. `appsettings.json` (기본)
2. `appsettings.{Environment}.json` (환경별)
3. 환경 변수
4. 명령줄 인수
5. User Secrets (개발 환경에서만)

나중에 로드되는 것이 이전 값을 오버라이드합니다.

### Q3: Program.cs가 통합된 이유는?

**A:**
- Startup.cs 분리가 불필요한 복잡성 추가
- Top-level statements로 보일러플레이트 제거
- 작은 앱에서 파일 개수 감소
- 설정 흐름이 한 파일에서 보여 이해하기 쉬움

---

## 참고 자료

- [SDK 스타일 프로젝트](https://learn.microsoft.com/dotnet/core/project-sdk/overview)
- [ASP.NET Core 프로젝트 구조](https://learn.microsoft.com/aspnet/core/fundamentals/)
- [.slnx 파일 형식](https://learn.microsoft.com/visualstudio/ide/solution-explorer)

---

## 다음 문서

→ [hosting-models.md](./hosting-models.md) - 호스팅 모델 이해하기
