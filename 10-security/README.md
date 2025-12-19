# 10. Security - 보안

ASP.NET Core의 인증, 인가, 보안 기능을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [authentication.md](./authentication.md) | 인증 기초 | 표면 |
| [jwt.md](./jwt.md) | JWT 토큰 인증 | 중간 |
| [authorization.md](./authorization.md) | 인가 정책 | 중간 |
| [data-protection.md](./data-protection.md) | 데이터 보호 API | 깊음 |
| [owasp.md](./owasp.md) | OWASP Top 10 대응 | 실용 |

---

## 핵심 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                    인증 vs 인가                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  인증 (Authentication): "당신은 누구인가?"                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 사용자 신원 확인                                      │   │
│  │  • 로그인 처리                                           │   │
│  │  • JWT, Cookie, OAuth                                   │   │
│  │  • UseAuthentication()                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  인가 (Authorization): "당신은 무엇을 할 수 있는가?"             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 권한/역할 확인                                        │   │
│  │  • 리소스 접근 제어                                      │   │
│  │  • 정책 기반, 역할 기반                                  │   │
│  │  • UseAuthorization()                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 빠른 시작

### JWT 인증

```csharp
// 등록
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://myapp.com",
            ValidAudience = "https://myapp.com",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("your-256-bit-secret"))
        };
    });

builder.Services.AddAuthorization();

// 미들웨어
app.UseAuthentication();
app.UseAuthorization();

// 보호된 엔드포인트
app.MapGet("/protected", () => "Secret")
    .RequireAuthorization();
```

### 정책 기반 인가

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("Over18", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c =>
                c.Type == "Age" && int.Parse(c.Value) >= 18)));

    options.AddPolicy("CanEditContent", policy =>
        policy.RequireClaim("Permission", "content:edit"));
});

// 사용
[Authorize(Policy = "AdminOnly")]
public class AdminController : ControllerBase { }

app.MapGet("/admin", () => "Admin area")
    .RequireAuthorization("AdminOnly");
```

---

## OWASP Top 10 대응

| 취약점 | ASP.NET Core 보호 |
|--------|------------------|
| Injection | EF Core 파라미터화, Dapper 파라미터 |
| XSS | 자동 HTML 인코딩, CSP |
| CSRF | AntiForgery 토큰 |
| 인증 결함 | Identity, JWT, 2FA |
| 민감 데이터 노출 | Data Protection API, HTTPS |
| XXE | DTD 처리 비활성화 (기본) |
| 접근 제어 | [Authorize], 정책 기반 인가 |

---

## 면접 빈출 질문

1. **Authentication vs Authorization 차이는?**
2. **JWT의 구조와 장단점은?**
3. **CSRF 공격과 방어 방법은?**
4. **Data Protection API란?**
5. **Policy-based Authorization의 장점은?**

---

## 다음 단계

→ [11. Logging](../11-logging/) - 로깅과 모니터링
