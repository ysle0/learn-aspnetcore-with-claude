# 09. Background Services - 백그라운드 작업

IHostedService와 Worker Service를 이용한 백그라운드 작업을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [hosted-services.md](./hosted-services.md) | IHostedService 기초 | 표면 |
| [background-service.md](./background-service.md) | BackgroundService 패턴 | 중간 |
| [graceful-shutdown.md](./graceful-shutdown.md) | 안전한 종료 | 깊음 |
| [worker-service.md](./worker-service.md) | Worker Service 프로젝트 | 실용 |

---

## 핵심 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hosted Service 생명주기                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  앱 시작                                                        │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────┐                   │
│  │  StartAsync() 호출                       │                   │
│  │  (등록 순서대로 순차 실행)               │                   │
│  └─────────────────────────────────────────┘                   │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────┐                   │
│  │  ExecuteAsync() 실행                     │                   │
│  │  (백그라운드에서 계속 실행)              │                   │
│  └─────────────────────────────────────────┘                   │
│     │                                                           │
│     │  SIGTERM / Ctrl+C / app.StopAsync()                      │
│     ▼                                                           │
│  ┌─────────────────────────────────────────┐                   │
│  │  StopAsync() 호출                        │                   │
│  │  (등록 역순으로 순차 실행)               │                   │
│  │  (ShutdownTimeout 내에 완료해야 함)      │                   │
│  └─────────────────────────────────────────┘                   │
│     │                                                           │
│     ▼                                                           │
│  앱 종료                                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 빠른 시작

### BackgroundService

```csharp
public class TimerService : BackgroundService
{
    private readonly ILogger<TimerService> _logger;

    public TimerService(ILogger<TimerService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Worker running at: {time}", DateTimeOffset.Now);

            await Task.Delay(1000, stoppingToken);
        }
    }
}

// 등록
builder.Services.AddHostedService<TimerService>();
```

### Scoped 서비스 사용

```csharp
public class ScopedWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public ScopedWorker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var dbContext = scope.ServiceProvider
                .GetRequiredService<AppDbContext>();

            // DbContext 사용
            await dbContext.ProcessPendingJobsAsync(stoppingToken);

            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

---

## Graceful Shutdown

```csharp
// 종료 타임아웃 설정
builder.Host.ConfigureHostOptions(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// 안전한 종료 처리
public class GracefulWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                await ProcessBatchAsync(stoppingToken);
            }
        }
        catch (OperationCanceledException)
        {
            // 정상 종료
            _logger.LogInformation("Worker stopping gracefully");
        }
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Worker stopping...");

        // 진행 중인 작업 완료 대기
        await SaveStateAsync();

        await base.StopAsync(cancellationToken);
    }
}
```

---

## 면접 빈출 질문

1. **IHostedService vs BackgroundService 차이는?**
2. **Scoped 서비스를 어떻게 사용하나요?**
3. **Graceful Shutdown이란?**
4. **ShutdownTimeout의 역할은?**

---

## 다음 단계

→ [10. Security](../10-security/) - 보안
