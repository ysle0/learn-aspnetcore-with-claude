# 17. .NET Aspire

.NET Aspire를 이용한 클라우드 네이티브 개발을 이해합니다.

---

## 개요

.NET Aspire는 분산 애플리케이션 개발을 단순화하는 프레임워크입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Aspire 구조                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  AppHost (오케스트레이터)                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  var builder = DistributedApplication.CreateBuilder();  │   │
│  │                                                          │   │
│  │  var redis = builder.AddRedis("cache");                 │   │
│  │  var postgres = builder.AddPostgres("db");              │   │
│  │                                                          │   │
│  │  builder.AddProject<Projects.WebApi>("api")             │   │
│  │      .WithReference(redis)                               │   │
│  │      .WithReference(postgres);                           │   │
│  │                                                          │   │
│  │  builder.Build().Run();                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ServiceDefaults (공통 설정)                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • OpenTelemetry (Traces, Metrics, Logs)                │   │
│  │  • Health Checks                                        │   │
│  │  • Service Discovery                                    │   │
│  │  • Resilience (Retry, Circuit Breaker)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 다음 단계

→ [18. AWS Integration](../18-aws-integration/) - AWS 통합
