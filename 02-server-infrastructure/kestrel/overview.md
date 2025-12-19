# Kestrel 개요

## 개요

Kestrel은 ASP.NET Core에 포함된 **크로스 플랫폼 웹 서버**입니다. 기본적으로 모든 ASP.NET Core 프로젝트에서 사용되며, 고성능과 낮은 메모리 사용량이 특징입니다.

---

## 핵심 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kestrel 특징                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🚀 고성능          │  수백만 req/sec 처리                       │
│  🌍 크로스 플랫폼   │  Windows, Linux, macOS                    │
│  🔒 보안            │  TLS 1.2/1.3, HTTPS                       │
│  📡 프로토콜        │  HTTP/1.1, HTTP/2, HTTP/3 (QUIC)          │
│  ⚡ 비동기 I/O     │  libuv → 관리형 소켓 (현재)                │
│  🧩 모듈러          │  필요한 기능만 로드                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 기본 사용법

### 1. 기본 설정 (코드)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    // 특정 포트에서 수신
    options.ListenLocalhost(5000);  // HTTP
    options.ListenLocalhost(5001, listenOptions =>
    {
        listenOptions.UseHttps();   // HTTPS
    });
});

var app = builder.Build();
app.MapGet("/", () => "Hello Kestrel!");
app.Run();
```

### 2. appsettings.json으로 설정

```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://localhost:5000"
      },
      "Https": {
        "Url": "https://localhost:5001"
      },
      "HttpsFromPem": {
        "Url": "https://*:5002",
        "Certificate": {
          "Path": "/path/to/cert.pem",
          "KeyPath": "/path/to/key.pem"
        }
      }
    }
  }
}
```

### 3. 환경 변수로 설정

```bash
# URL 설정
export ASPNETCORE_URLS="http://localhost:5000;https://localhost:5001"

# 또는 Kestrel 섹션 직접 설정
export Kestrel__Endpoints__Http__Url="http://localhost:5000"
```

---

## 엔드포인트 구성

### 다양한 바인딩 방식

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    // 1. localhost만 (개발용)
    options.ListenLocalhost(5000);

    // 2. 모든 IP에서 수신
    options.ListenAnyIP(80);

    // 3. 특정 IP
    options.Listen(IPAddress.Parse("192.168.1.100"), 5000);

    // 4. Unix 소켓 (Linux)
    options.ListenUnixSocket("/tmp/kestrel.sock");

    // 5. 명명된 파이프 (Windows)
    options.ListenNamedPipe("MyPipe");
});
```

### HTTPS 구성

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(443, listenOptions =>
    {
        // 1. PFX 파일
        listenOptions.UseHttps("certificate.pfx", "password");

        // 2. PEM 파일 (Linux에서 주로 사용)
        listenOptions.UseHttps(options =>
        {
            options.ServerCertificate = X509Certificate2.CreateFromPemFile(
                "cert.pem", "key.pem");
        });

        // 3. 인증서 저장소에서 로드 (Windows)
        listenOptions.UseHttps(StoreName.My, "MyCertSubject",
            allowInvalid: false, StoreLocation.CurrentUser);
    });
});
```

### HTTP/2 및 HTTP/3 설정

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps();

        // HTTP/2 활성화 (HTTPS에서 기본 활성화)
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;
    });

    options.ListenAnyIP(5002, listenOptions =>
    {
        listenOptions.UseHttps();

        // HTTP/3 활성화 (QUIC)
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
    });
});
```

---

## 성능 제한 설정

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    // 연결 제한
    options.Limits.MaxConcurrentConnections = 100;
    options.Limits.MaxConcurrentUpgradedConnections = 100; // WebSocket 등

    // 요청 제한
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB (기본: 30MB)
    options.Limits.MaxRequestHeaderCount = 100;           // 기본: 100
    options.Limits.MaxRequestHeadersTotalSize = 32 * 1024; // 32KB
    options.Limits.MaxRequestLineSize = 8 * 1024;          // 8KB

    // 타임아웃
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
    options.Limits.RequestHeadersTimeout = TimeSpan.FromSeconds(30);

    // HTTP/2 설정
    options.Limits.Http2.MaxStreamsPerConnection = 100;
    options.Limits.Http2.HeaderTableSize = 4096;
    options.Limits.Http2.MaxFrameSize = 16 * 1024;
    options.Limits.Http2.MaxRequestHeaderFieldSize = 8 * 1024;
    options.Limits.Http2.InitialConnectionWindowSize = 128 * 1024;
    options.Limits.Http2.InitialStreamWindowSize = 96 * 1024;

    // HTTP/3 설정
    options.Limits.Http3.MaxRequestHeaderFieldSize = 16 * 1024;
});
```

### appsettings.json으로 제한 설정

```json
{
  "Kestrel": {
    "Limits": {
      "MaxConcurrentConnections": 100,
      "MaxRequestBodySize": 10485760,
      "KeepAliveTimeout": "00:02:00",
      "RequestHeadersTimeout": "00:00:30",
      "Http2": {
        "MaxStreamsPerConnection": 100,
        "InitialConnectionWindowSize": 131072
      }
    }
  }
}
```

---

## 엔드포인트별 설정

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    // 일반 API 엔드포인트
    options.Listen(IPAddress.Any, 5000, listenOptions =>
    {
        listenOptions.UseHttps();
    });

    // 파일 업로드 전용 엔드포인트 (큰 요청 허용)
    options.Listen(IPAddress.Any, 5001, listenOptions =>
    {
        listenOptions.UseHttps();
        listenOptions.KestrelServerOptions.Limits.MaxRequestBodySize = 100 * 1024 * 1024; // 100MB
    });
});

// 또는 특정 엔드포인트에서만 제한 해제
app.MapPost("/upload", async (HttpRequest request) =>
{
    // 이 엔드포인트에서만 제한 해제
    request.Body.SetMaxRequestBodySize(null);
    // ...
}).DisableAntiforgery();
```

---

## SNI (Server Name Indication)

여러 도메인을 하나의 IP에서 호스팅:

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(443, listenOptions =>
    {
        listenOptions.UseHttps(httpsOptions =>
        {
            httpsOptions.ServerCertificateSelector = (context, host) =>
            {
                return host switch
                {
                    "api.example.com" => LoadCertificate("api.pfx"),
                    "www.example.com" => LoadCertificate("www.pfx"),
                    _ => LoadCertificate("default.pfx")
                };
            };
        });
    });
});
```

---

## 클라이언트 인증서

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps(httpsOptions =>
        {
            httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
            httpsOptions.ClientCertificateValidation = (cert, chain, errors) =>
            {
                // 커스텀 검증 로직
                return cert.Issuer == "CN=MyCA";
            };
        });
    });
});

// 컨트롤러에서 클라이언트 인증서 접근
app.MapGet("/secure", (HttpContext context) =>
{
    var clientCert = context.Connection.ClientCertificate;
    return $"Client: {clientCert?.Subject}";
});
```

---

## 연결 미들웨어

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000, listenOptions =>
    {
        // 연결 수준에서 실행되는 미들웨어
        listenOptions.Use(async (context, next) =>
        {
            // 연결이 수립될 때 실행
            var remoteEndpoint = context.RemoteEndPoint;
            Console.WriteLine($"Connection from: {remoteEndpoint}");

            await next();

            // 연결이 종료될 때 실행
            Console.WriteLine($"Connection closed: {remoteEndpoint}");
        });
    });
});
```

---

## 진단 및 로깅

```csharp
builder.WebHost.ConfigureKestrel((context, options) =>
{
    // Kestrel 진단 활성화
    options.AddServerHeader = false; // Server 헤더 제거 (보안)

    options.ConfigureEndpointDefaults(listenOptions =>
    {
        listenOptions.UseConnectionLogging(); // 연결 로깅
    });
});

// appsettings.json에서 로그 레벨 설정
// "Microsoft.AspNetCore.Server.Kestrel": "Debug"
```

---

## 면접 예상 질문

### Q1: Kestrel의 기본 요청 본문 크기 제한은?

**A:** 기본값은 약 **30MB** (28.6MB = 30,000,000 bytes)입니다. `MaxRequestBodySize`로 변경 가능하며, `null`로 설정하면 제한이 없어집니다.

### Q2: Kestrel에서 HTTP/2를 사용하려면?

**A:** HTTPS가 활성화되면 HTTP/2가 기본으로 활성화됩니다. HTTP에서 HTTP/2를 사용하려면 `HttpProtocols.Http2`를 명시적으로 설정해야 합니다.

### Q3: KeepAliveTimeout과 RequestHeadersTimeout의 차이는?

**A:**
- **KeepAliveTimeout**: 클라이언트가 Keep-Alive 연결에서 다음 요청을 보낼 때까지 대기하는 시간
- **RequestHeadersTimeout**: 요청 헤더를 받는 데 허용되는 시간

---

## 참고 자료

- [Kestrel 웹 서버 - Microsoft Learn](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel)
- [Kestrel 엔드포인트 구성](https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel/endpoints)

---

## 다음 문서

→ [internals.md](./internals.md) - Kestrel 내부 동작 이해하기
