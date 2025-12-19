# 13. Rate Limiting - 요청 제한

ASP.NET Core의 요청 제한 기능을 이해합니다.

---

## 빠른 시작 (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    // 고정 윈도우
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 10;
    });

    // 슬라이딩 윈도우
    options.AddSlidingWindowLimiter("sliding", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.SegmentsPerWindow = 6;
    });

    // 토큰 버킷
    options.AddTokenBucketLimiter("token", opt =>
    {
        opt.TokenLimit = 100;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 10;
    });

    // 거부 시 응답
    options.OnRejected = async (context, _) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        await context.HttpContext.Response.WriteAsync("Too many requests");
    };
});

app.UseRateLimiter();

app.MapGet("/api/data", () => "Data")
    .RequireRateLimiting("fixed");
```

---

## 다음 단계

→ [14. Source Generators](../14-source-generators/) - 소스 생성기
