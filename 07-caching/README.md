# 07. Caching - 캐싱

ASP.NET Core의 캐싱 전략과 HybridCache를 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [memory-cache.md](./memory-cache.md) | In-Memory 캐싱 | 표면 |
| [distributed-cache.md](./distributed-cache.md) | 분산 캐싱 (Redis) | 중간 |
| [hybrid-cache.md](./hybrid-cache.md) | HybridCache (.NET 9+) | 실용 |
| [output-cache.md](./output-cache.md) | 출력 캐싱 | 중간 |

---

## 캐싱 계층

```
┌─────────────────────────────────────────────────────────────────┐
│                    캐싱 계층 구조                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  클라이언트                                                     │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │  Browser Cache (Cache-Control)          │  L1               │
│  └─────────────────────────────────────────┘                   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │  CDN Cache (CloudFlare, CloudFront)     │  L2               │
│  └─────────────────────────────────────────┘                   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │  Output Cache / Response Cache          │  L3               │
│  └─────────────────────────────────────────┘                   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │  Application Cache (Memory/Distributed) │  L4               │
│  └─────────────────────────────────────────┘                   │
│      │                                                          │
│      ▼                                                          │
│  ┌─────────────────────────────────────────┐                   │
│  │  Database                               │  Source           │
│  └─────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 빠른 시작

### IMemoryCache

```csharp
builder.Services.AddMemoryCache();

public class ProductService(IMemoryCache cache, IProductRepository repo)
{
    public async Task<Product?> GetByIdAsync(int id)
    {
        var key = $"product:{id}";

        return await cache.GetOrCreateAsync(key, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            entry.SlidingExpiration = TimeSpan.FromMinutes(1);
            return await repo.GetByIdAsync(id);
        });
    }
}
```

### IDistributedCache (Redis)

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp:";
});

public class SessionService(IDistributedCache cache)
{
    public async Task SetAsync(string key, object value)
    {
        var json = JsonSerializer.Serialize(value);
        await cache.SetStringAsync(key, json, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
        });
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var json = await cache.GetStringAsync(key);
        return json is null ? default : JsonSerializer.Deserialize<T>(json);
    }
}
```

### HybridCache (.NET 9+)

```csharp
builder.Services.AddHybridCache();

public class CatalogService(HybridCache cache, ICatalogRepository repo)
{
    public async Task<Product?> GetProductAsync(int id, CancellationToken ct)
    {
        return await cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await repo.GetByIdAsync(id, token),
            cancellationToken: ct);
    }
}
```

---

## 캐시 비교

| 기능 | IMemoryCache | IDistributedCache | HybridCache |
|------|-------------|------------------|-------------|
| 로컬 캐시 | ✅ | ❌ | ✅ |
| 분산 캐시 | ❌ | ✅ | ✅ |
| Stampede 방지 | ❌ | ❌ | ✅ |
| 직렬화 | 불필요 | 필요 | 자동 |
| 태그 기반 무효화 | ❌ | ❌ | ✅ |

---

## 면접 빈출 질문

1. **Cache Stampede란? 어떻게 방지하나요?**
2. **IMemoryCache vs IDistributedCache 차이는?**
3. **HybridCache의 장점은?**
4. **캐시 무효화 전략은?**

---

## 다음 단계

→ [08. Realtime](../08-realtime/) - 실시간 통신
