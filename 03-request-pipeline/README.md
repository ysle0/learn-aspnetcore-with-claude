# 03. Request Pipeline - 요청 파이프라인

ASP.NET Core의 핵심인 미들웨어 파이프라인의 동작 원리와 활용법을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [middleware-fundamentals.md](./middleware-fundamentals.md) | 미들웨어 기초 개념 | 표면 |
| [middleware-internals.md](./middleware-internals.md) | 미들웨어 내부 동작 | 깊음 |
| [built-in-middleware.md](./built-in-middleware.md) | 내장 미들웨어 가이드 | 중간 |
| [custom-middleware.md](./custom-middleware.md) | 커스텀 미들웨어 작성 | 실용 |

---

## 핵심 요약

### 미들웨어 파이프라인이란?

미들웨어는 HTTP 요청과 응답을 처리하는 **컴포넌트 체인**입니다. 각 미들웨어는 요청을 다음 컴포넌트에 전달하거나, 직접 응답을 생성할 수 있습니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                      요청 파이프라인                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Request ──►                                    ──► Response   │
│              │                                  │               │
│              ▼                                  │               │
│   ┌──────────────────┐                          │               │
│   │   Middleware 1   │──────────────────────────┤               │
│   │  (Exception)     │◄─────────────────────────┤               │
│   └────────┬─────────┘                          │               │
│            │                                    │               │
│            ▼                                    │               │
│   ┌──────────────────┐                          │               │
│   │   Middleware 2   │──────────────────────────┤               │
│   │    (Logging)     │◄─────────────────────────┤               │
│   └────────┬─────────┘                          │               │
│            │                                    │               │
│            ▼                                    │               │
│   ┌──────────────────┐                          │               │
│   │   Middleware 3   │──────────────────────────┤               │
│   │    (Routing)     │◄─────────────────────────┤               │
│   └────────┬─────────┘                          │               │
│            │                                    │               │
│            ▼                                    │               │
│   ┌──────────────────┐                          │               │
│   │   Middleware 4   │                          │               │
│   │   (Endpoint)     │──────────────────────────┘               │
│   └──────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기본 미들웨어 예제

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 미들웨어 1: 요청 로깅
app.Use(async (context, next) =>
{
    Console.WriteLine($"Request: {context.Request.Path}");
    await next(context);  // 다음 미들웨어 호출
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});

// 미들웨어 2: 인증 확인
app.Use(async (context, next) =>
{
    if (!context.User.Identity?.IsAuthenticated ?? true)
    {
        context.Response.StatusCode = 401;
        return;  // 파이프라인 단축 (Short-circuit)
    }
    await next(context);
});

// 종단 미들웨어
app.MapGet("/", () => "Hello World!");

app.Run();
```

---

## 미들웨어 순서

### 권장 순서

```csharp
var app = builder.Build();

// 1. 예외 처리 (가장 먼저)
app.UseExceptionHandler("/Error");

// 2. HSTS (HTTPS 강제)
app.UseHsts();

// 3. HTTPS 리다이렉션
app.UseHttpsRedirection();

// 4. 정적 파일 (빠른 응답)
app.UseStaticFiles();

// 5. 라우팅
app.UseRouting();

// 6. CORS (인증 전에)
app.UseCors();

// 7. 인증
app.UseAuthentication();

// 8. 인가
app.UseAuthorization();

// 9. 세션
app.UseSession();

// 10. 엔드포인트 매핑
app.MapControllers();

app.Run();
```

### 순서가 중요한 이유

```
┌─────────────────────────────────────────────────────────────────┐
│                    잘못된 순서 예시                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ❌ 잘못된 순서:                                                 │
│     UseAuthorization() → UseAuthentication()                    │
│     결과: 인증되지 않은 사용자를 인가하려고 시도                   │
│                                                                 │
│  ❌ 잘못된 순서:                                                 │
│     UseRouting() → UseCors()                                    │
│     결과: CORS 헤더가 응답에 포함되지 않을 수 있음                │
│                                                                 │
│  ❌ 잘못된 순서:                                                 │
│     UseStaticFiles() → UseExceptionHandler()                    │
│     결과: 정적 파일 오류가 처리되지 않음                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 미들웨어 유형

### 1. Use - 체인 미들웨어

```csharp
app.Use(async (context, next) =>
{
    // 요청 처리 전 로직
    await next(context);
    // 응답 처리 후 로직
});
```

### 2. Map - 분기 미들웨어

```csharp
app.Map("/api", apiApp =>
{
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    apiApp.Run(async context =>
    {
        await context.Response.WriteAsync("API Endpoint");
    });
});

app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin Area");
    });
});
```

### 3. MapWhen - 조건부 분기

```csharp
app.MapWhen(
    context => context.Request.Query.ContainsKey("debug"),
    debugApp =>
    {
        debugApp.Run(async context =>
        {
            await context.Response.WriteAsync("Debug Mode");
        });
    });
```

### 4. Run - 종단 미들웨어

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Final Response");
    // next()가 없음 - 파이프라인 종료
});
```

### 5. UseWhen - 조건부 실행 (다시 합류)

```csharp
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    appBuilder =>
    {
        appBuilder.UseMiddleware<ApiLoggingMiddleware>();
    });
// 조건과 관계없이 이후 미들웨어로 계속 진행
```

---

## 주요 내장 미들웨어

| 미들웨어 | 역할 | 순서 |
|----------|------|------|
| `UseExceptionHandler` | 예외 처리 | 1 |
| `UseHsts` | HTTP Strict Transport Security | 2 |
| `UseHttpsRedirection` | HTTPS 리다이렉트 | 3 |
| `UseStaticFiles` | 정적 파일 제공 | 4 |
| `UseRouting` | 라우팅 결정 | 5 |
| `UseCors` | Cross-Origin 요청 | 6 |
| `UseAuthentication` | 인증 | 7 |
| `UseAuthorization` | 인가 | 8 |
| `UseSession` | 세션 관리 | 9 |
| `UseResponseCaching` | 응답 캐싱 | 가변 |
| `UseRateLimiter` | 요청 제한 | 인증 후 |

---

## 면접 빈출 질문

1. **미들웨어와 필터의 차이점은?**
2. **미들웨어 순서가 왜 중요한가요?**
3. **Use vs Run vs Map의 차이는?**
4. **미들웨어에서 DI를 어떻게 사용하나요?**
5. **Short-circuit(파이프라인 단축)이란?**

---

## 다음 단계

→ [04. Dependency Injection](../04-dependency-injection/) - DI 컨테이너 이해하기
