# 02. Server Infrastructure - 서버 인프라

ASP.NET Core의 웹 서버인 Kestrel의 동작 원리와 IIS, 리버스 프록시 구성을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [kestrel/overview.md](./kestrel/overview.md) | Kestrel 개요와 설정 | 표면 |
| [kestrel/internals.md](./kestrel/internals.md) | Kestrel 내부 동작 | 깊음 |
| [iis-integration.md](./iis-integration.md) | IIS 연동 | 중간 |
| [reverse-proxy.md](./reverse-proxy.md) | 리버스 프록시 패턴 | 실용 |

---

## 핵심 요약

### ASP.NET Core 서버 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                        요청 흐름                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  클라이언트                                                      │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────┐                                            │
│  │  리버스 프록시   │  (Nginx, IIS, Apache, 클라우드 LB)         │
│  │  (선택사항)      │                                           │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │     Kestrel     │  ← 기본 웹 서버                             │
│  │   (HTTP 서버)   │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │   미들웨어       │                                            │
│  │   파이프라인     │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │   앱 로직       │  (Controllers, Endpoints)                  │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Kestrel이란?

Kestrel은 ASP.NET Core의 **기본 크로스 플랫폼 웹 서버**입니다.

| 특성 | 설명 |
|------|------|
| 성능 | 수백만 req/sec 처리 가능 |
| 프로토콜 | HTTP/1.1, HTTP/2, HTTP/3 (QUIC) |
| 보안 | TLS/HTTPS 지원 |
| 플랫폼 | Windows, Linux, macOS |

### 기본 Kestrel 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    // HTTP
    options.ListenLocalhost(5000);

    // HTTPS
    options.ListenLocalhost(5001, listenOptions =>
    {
        listenOptions.UseHttps();
    });

    // 성능 튜닝
    options.Limits.MaxConcurrentConnections = 100;
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB
});

var app = builder.Build();
app.MapGet("/", () => "Hello from Kestrel!");
app.Run();
```

---

## 서버 옵션 비교

| 기능 | Kestrel | IIS In-Process | HTTP.sys |
|------|---------|----------------|----------|
| 크로스 플랫폼 | ✅ | ❌ | ❌ |
| 성능 | 최고 | 높음 | 높음 |
| Windows 인증 | 미들웨어 | ✅ 내장 | ✅ 내장 |
| 포트 공유 | ❌ | ❌ | ✅ |
| HTTP/2 | ✅ | ✅ | ✅ |
| HTTP/3 | ✅ | ❌ | ❌ |
| 직접 인터넷 노출 | 가능 | N/A | 가능 |

---

## 권장 구성

### 개발 환경
```
Kestrel (단독)
```

### 컨테이너/클라우드
```
Load Balancer/Ingress → Kestrel
```

### Windows Server (온프레미스)
```
IIS (In-Process) 또는 IIS → Kestrel (Out-of-Process)
```

### Linux 서버 (온프레미스)
```
Nginx → Kestrel
```

---

## 면접 빈출 질문

1. **Kestrel과 IIS의 차이점은?**
2. **Kestrel을 프로덕션에서 단독으로 사용해도 되나요?**
3. **HTTP/2와 HTTP/3의 차이점은?**
4. **리버스 프록시를 사용하는 이유는?**

---

## 다음 단계

→ [03. Request Pipeline](../03-request-pipeline/) - 미들웨어 파이프라인 이해하기
