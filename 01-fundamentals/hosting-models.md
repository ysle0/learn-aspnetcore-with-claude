# 호스팅 모델

## 개요

ASP.NET Core 앱은 다양한 방식으로 호스팅될 수 있습니다. 핵심 구성 요소는 **Kestrel**(웹 서버)과 **Host**(앱 생명주기 관리)입니다.

---

## 호스팅 옵션

```
┌─────────────────────────────────────────────────────────────────┐
│                     호스팅 옵션                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Kestrel 단독          2. Kestrel + 리버스 프록시            │
│  ┌─────────────┐          ┌─────────────┐  ┌─────────────┐     │
│  │   Kestrel   │          │  Nginx/IIS  │→│   Kestrel   │     │
│  │   (직접)    │          │  (프록시)   │  │   (백엔드)  │     │
│  └─────────────┘          └─────────────┘  └─────────────┘     │
│                                                                 │
│  3. IIS In-Process        4. HTTP.sys (Windows)                 │
│  ┌─────────────────────┐  ┌─────────────┐                      │
│  │        IIS          │  │  HTTP.sys   │                      │
│  │  ┌───────────────┐  │  │ (커널 모드) │                      │
│  │  │  ASP.NET Core │  │  └─────────────┘                      │
│  │  │   (In-Proc)   │  │                                       │
│  │  └───────────────┘  │                                       │
│  └─────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Kestrel 단독 실행

가장 단순한 형태. 개발 환경이나 컨테이너에서 주로 사용.

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

// 기본: http://localhost:5000, https://localhost:5001
app.Run();
```

### Kestrel 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

// Kestrel 설정
builder.WebHost.ConfigureKestrel(options =>
{
    // 포트 설정
    options.ListenLocalhost(5000);
    options.ListenLocalhost(5001, listenOptions =>
    {
        listenOptions.UseHttps();
    });

    // 모든 IP에서 수신
    options.ListenAnyIP(80);
    options.ListenAnyIP(443, listenOptions =>
    {
        listenOptions.UseHttps("certificate.pfx", "password");
    });

    // 성능 설정
    options.Limits.MaxConcurrentConnections = 100;
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
});
```

### appsettings.json으로 설정

```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://localhost:5000"
      },
      "Https": {
        "Url": "https://localhost:5001",
        "Certificate": {
          "Path": "certificate.pfx",
          "Password": "password"
        }
      }
    },
    "Limits": {
      "MaxConcurrentConnections": 100,
      "MaxRequestBodySize": 10485760
    }
  }
}
```

---

## 2. Kestrel + 리버스 프록시

프로덕션 환경에서 권장되는 구성.

```
인터넷 → [Nginx/IIS/Apache] → [Kestrel] → ASP.NET Core 앱
           (리버스 프록시)      (백엔드)
```

### 리버스 프록시를 사용하는 이유

| 기능 | Kestrel 단독 | + 리버스 프록시 |
|------|-------------|-----------------|
| SSL 종료 | 가능 | ✅ 더 효율적 |
| 로드 밸런싱 | ❌ | ✅ |
| DDoS 방어 | 기본만 | ✅ 강력 |
| 정적 파일 캐싱 | 가능 | ✅ 더 효율적 |
| URL 재작성 | 미들웨어 | ✅ |
| gzip 압축 | 미들웨어 | ✅ |

### Nginx 설정 예시

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Forwarded Headers 미들웨어

```csharp
var builder = WebApplication.CreateBuilder(args);

// 프록시 뒤에서 원본 클라이언트 정보 획득
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders =
        ForwardedHeaders.XForwardedFor |
        ForwardedHeaders.XForwardedProto;

    // 신뢰할 프록시 설정 (프로덕션에서 필수)
    options.KnownProxies.Add(IPAddress.Parse("10.0.0.1"));
});

var app = builder.Build();

// 파이프라인 최상단에 배치
app.UseForwardedHeaders();

app.MapGet("/", (HttpContext ctx) =>
{
    // 프록시 통과 후에도 원본 IP 확인 가능
    var clientIp = ctx.Connection.RemoteIpAddress;
    return $"Client IP: {clientIp}";
});

app.Run();
```

---

## 3. IIS 호스팅

### In-Process vs Out-of-Process

```
In-Process (권장)
┌────────────────────────────────────────┐
│                  IIS                   │
│  ┌──────────────────────────────────┐  │
│  │         w3wp.exe 프로세스        │  │
│  │  ┌────────────────────────────┐  │  │
│  │  │     ASP.NET Core 앱        │  │  │
│  │  │  (IIS 프로세스 내부 실행)  │  │  │
│  │  └────────────────────────────┘  │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘

Out-of-Process
┌────────────────────────────────────────┐
│                  IIS                   │
│  ┌────────────────────────────────┐    │
│  │  ASP.NET Core Module (ANCM)   │    │
│  │         (프록시 역할)          │    │
│  └─────────────┬──────────────────┘    │
└────────────────┼───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│     별도 dotnet.exe 프로세스          │
│  ┌────────────────────────────────┐    │
│  │  Kestrel + ASP.NET Core 앱    │    │
│  └────────────────────────────────┘    │
└────────────────────────────────────────┘
```

### 성능 비교

| 항목 | In-Process | Out-of-Process |
|------|-----------|----------------|
| 요청 처리량 | 더 빠름 (프록시 없음) | 프록시 오버헤드 |
| 프로세스 격리 | IIS와 공유 | 완전 분리 |
| 메모리 사용 | 효율적 | 추가 프로세스 |
| 앱 풀 재활용 | 영향 받음 | 독립적 |

### web.config 설정

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*"
             modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>

      <!-- In-Process (기본값) -->
      <aspNetCore processPath="dotnet"
                  arguments=".\MyApp.dll"
                  stdoutLogEnabled="false"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess">
        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT"
                               value="Production" />
        </environmentVariables>
      </aspNetCore>

      <!-- Out-of-Process -->
      <!--
      <aspNetCore processPath="dotnet"
                  arguments=".\MyApp.dll"
                  stdoutLogEnabled="true"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="outofprocess" />
      -->
    </system.webServer>
  </location>
</configuration>
```

### .csproj에서 호스팅 모델 지정

```xml
<PropertyGroup>
  <!-- In-Process (기본값) -->
  <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>

  <!-- Out-of-Process -->
  <!-- <AspNetCoreHostingModel>OutOfProcess</AspNetCoreHostingModel> -->
</PropertyGroup>
```

---

## 4. HTTP.sys (Windows 전용)

커널 모드 드라이버로 동작하는 고성능 웹 서버.

```csharp
var builder = WebApplication.CreateBuilder(args);

// HTTP.sys 사용
builder.WebHost.UseHttpSys(options =>
{
    options.Authentication.Schemes =
        AuthenticationSchemes.NTLM | AuthenticationSchemes.Negotiate;
    options.Authentication.AllowAnonymous = true;
    options.MaxConnections = 100;
    options.MaxRequestBodySize = 30_000_000; // 30MB
    options.UrlPrefixes.Add("http://localhost:5000");
});

var app = builder.Build();
app.MapGet("/", () => "Hello from HTTP.sys!");
app.Run();
```

### HTTP.sys vs Kestrel

| 기능 | Kestrel | HTTP.sys |
|------|---------|----------|
| 플랫폼 | 크로스 플랫폼 | Windows 전용 |
| Windows 인증 | 미들웨어 필요 | 내장 지원 |
| 포트 공유 | ❌ | ✅ |
| 커널 모드 캐싱 | ❌ | ✅ |
| 직접 인터넷 노출 | 비권장 | 가능 |

---

## Generic Host vs WebApplication

### Generic Host (범용)

```csharp
// 콘솔 앱, 백그라운드 서비스용
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<MyBackgroundService>();
    })
    .Build();

await host.RunAsync();
```

### WebApplication (웹 특화)

```csharp
// 웹 앱용 - 더 간결한 API
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.MapGet("/", () => "Hello!");
app.Run();
```

### 내부 관계

```
WebApplicationBuilder
       │
       │ 내부적으로 사용
       ▼
HostApplicationBuilder (Generic Host)
       │
       │ 웹 기능 추가
       ▼
WebApplication
```

---

## 앱 생명주기

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 앱 생명주기 이벤트
app.Lifetime.ApplicationStarted.Register(() =>
{
    Console.WriteLine("앱 시작됨");
});

app.Lifetime.ApplicationStopping.Register(() =>
{
    Console.WriteLine("앱 종료 중...");
    // 정리 작업
});

app.Lifetime.ApplicationStopped.Register(() =>
{
    Console.WriteLine("앱 종료됨");
});

app.Run();
```

### 시작/종료 순서

```
앱 시작:
1. Host 생성
2. 서비스 등록 (ConfigureServices)
3. 파이프라인 구성 (Configure)
4. IHostedService.StartAsync() 호출
5. ApplicationStarted 이벤트

앱 종료 (Ctrl+C 또는 SIGTERM):
1. ApplicationStopping 이벤트
2. IHostedService.StopAsync() 호출 (역순)
3. ApplicationStopped 이벤트
4. Host Dispose
```

---

## 환경 설정

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 환경 확인
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else if (app.Environment.IsStaging())
{
    // 스테이징 설정
}
else if (app.Environment.IsProduction())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

// 커스텀 환경 확인
if (app.Environment.IsEnvironment("Testing"))
{
    // 테스트 환경 설정
}
```

### 환경 변수 설정 방법

```bash
# 환경 변수
export ASPNETCORE_ENVIRONMENT=Production

# 명령줄
dotnet run --environment Production

# launchSettings.json
{
  "profiles": {
    "Development": {
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

---

## 면접 예상 질문

### Q1: In-Process와 Out-of-Process 호스팅의 차이는?

**A:**
- **In-Process**: IIS 워커 프로세스(w3wp.exe) 내부에서 앱 실행. 프록시 없이 직접 처리하므로 더 빠름.
- **Out-of-Process**: 별도 dotnet.exe 프로세스에서 Kestrel 실행. IIS는 프록시 역할만 함. 프로세스 격리됨.

### Q2: 프로덕션에서 Kestrel만 사용하면 안 되나요?

**A:**
가능하지만 권장하지 않습니다:
- 리버스 프록시가 제공하는 DDoS 방어, 로드 밸런싱, SSL 오프로드 기능이 없음
- 컨테이너 환경에서는 로드 밸런서/인그레스가 이 역할을 대신함

### Q3: UseForwardedHeaders()는 왜 필요한가요?

**A:**
프록시 뒤에서 실행될 때 원본 클라이언트 IP와 프로토콜(HTTP/HTTPS) 정보가 손실됩니다. X-Forwarded-For, X-Forwarded-Proto 헤더를 해석하여 원본 정보를 복원합니다.

---

## 참고 자료

- [ASP.NET Core 호스팅 모델](https://learn.microsoft.com/aspnet/core/fundamentals/servers/)
- [Kestrel 웹 서버](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel)
- [IIS에서 호스팅](https://learn.microsoft.com/aspnet/core/host-and-deploy/iis/)

---

## 다음 섹션

→ [02. Server Infrastructure](../02-server-infrastructure/) - Kestrel 내부 동작 이해하기
