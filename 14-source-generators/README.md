# 14. Source Generators - 소스 생성기

C# Source Generator를 이용한 컴파일 타임 코드 생성을 이해합니다.

---

## 개요

Source Generator는 컴파일 시점에 추가 소스 코드를 생성하는 .NET 기능입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Source Generator 흐름                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  소스 코드 (.cs)                                                │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────┐                                           │
│  │    컴파일러     │                                            │
│  │   (Roslyn)      │                                            │
│  └────────┬────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │   구문 분석     │ ──► │ Source Generator │                   │
│  │   (Syntax)      │     │  (코드 생성)     │                   │
│  └─────────────────┘     └────────┬────────┘                   │
│                                   │                             │
│                                   ▼                             │
│                          ┌─────────────────┐                   │
│                          │  생성된 코드    │                   │
│                          │  (.g.cs)        │                   │
│                          └────────┬────────┘                   │
│                                   │                             │
│                                   ▼                             │
│                          ┌─────────────────┐                   │
│                          │   최종 어셈블리  │                   │
│                          └─────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 활용 예시

- `System.Text.Json` - JSON 직렬화
- `LoggerMessage.Define` - 고성능 로깅
- `Regex.GeneratedRegex` - 정규식 컴파일
- AutoMapper, Mapperly - 객체 매핑

```csharp
// 예: Regex Source Generator
[GeneratedRegex(@"\d+")]
private static partial Regex NumberPattern();

// 예: 고성능 로깅
[LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} processed")]
static partial void LogOrderProcessed(ILogger logger, int orderId);
```

---

## 다음 단계

→ [15. Extreme Optimization](../15-extreme-optimization/) - 극한 최적화
