# 11. Logging - 로깅

ASP.NET Core 로깅 시스템과 구조화된 로깅을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [fundamentals.md](./fundamentals.md) | 로깅 기초 | 표면 |
| [structured-logging.md](./structured-logging.md) | 구조화된 로깅 | 중간 |
| [serilog.md](./serilog.md) | Serilog 통합 | 실용 |

---

## 빠른 시작

```csharp
public class OrderService(ILogger<OrderService> logger)
{
    public void ProcessOrder(int orderId)
    {
        logger.LogInformation("Processing order {OrderId}", orderId);

        try
        {
            // 처리 로직
            logger.LogDebug("Order {OrderId} validated", orderId);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to process order {OrderId}", orderId);
            throw;
        }
    }
}
```

### 로그 레벨

```csharp
logger.LogTrace("가장 상세한 정보");
logger.LogDebug("디버깅 정보");
logger.LogInformation("일반 정보");
logger.LogWarning("경고");
logger.LogError(ex, "오류");
logger.LogCritical("심각한 오류");
```

---

## 다음 단계

→ [12. TAP Internals](../12-tap-internals/) - 비동기 프로그래밍
