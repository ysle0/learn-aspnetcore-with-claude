# 06. Data Access - 데이터 액세스

다양한 데이터 액세스 기술과 패턴을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [ef-core.md](./ef-core.md) | Entity Framework Core | 중간 |
| [dapper.md](./dapper.md) | Dapper 마이크로 ORM | 실용 |
| [repository-pattern.md](./repository-pattern.md) | Repository 패턴 | 실용 |
| [connection-pooling.md](./connection-pooling.md) | 연결 풀링 | 깊음 |

---

## 핵심 비교

### EF Core vs Dapper

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORM 비교                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Entity Framework Core                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Full ORM (풍부한 기능)                               │   │
│  │  • LINQ to SQL 변환                                     │   │
│  │  • 변경 추적 (Change Tracking)                          │   │
│  │  • 마이그레이션 지원                                     │   │
│  │  • 관계 자동 로딩 (Lazy/Eager)                          │   │
│  │  • 성능: 느림 (추상화 오버헤드)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Dapper                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Micro ORM (최소 추상화)                              │   │
│  │  • Raw SQL 작성                                         │   │
│  │  • 빠른 매핑 (Object Mapping)                           │   │
│  │  • 간단한 API                                           │   │
│  │  • 성능: 빠름 (ADO.NET에 가까움)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 성능 비교 (상대적)

| 작업 | ADO.NET | Dapper | EF Core |
|------|---------|--------|---------|
| 단순 SELECT | 1x | 1.1x | 2-3x |
| INSERT | 1x | 1.1x | 2x |
| 복잡한 쿼리 | 1x | 1.2x | 3-5x |
| 메모리 사용 | 최소 | 적음 | 많음 |

---

## 빠른 시작

### EF Core

```csharp
// DbContext
public class AppDbContext : DbContext
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
}

// 등록
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// 사용
public class UserService(AppDbContext context)
{
    public async Task<List<User>> GetAllAsync()
        => await context.Users.ToListAsync();

    public async Task<User?> GetByIdAsync(int id)
        => await context.Users.FindAsync(id);

    public async Task CreateAsync(User user)
    {
        context.Users.Add(user);
        await context.SaveChangesAsync();
    }
}
```

### Dapper

```csharp
// 등록
builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(connectionString));

// 사용
public class UserRepository(IDbConnection db)
{
    public async Task<IEnumerable<User>> GetAllAsync()
        => await db.QueryAsync<User>("SELECT * FROM Users");

    public async Task<User?> GetByIdAsync(int id)
        => await db.QuerySingleOrDefaultAsync<User>(
            "SELECT * FROM Users WHERE Id = @Id",
            new { Id = id });

    public async Task<int> CreateAsync(User user)
        => await db.ExecuteAsync(
            "INSERT INTO Users (Name, Email) VALUES (@Name, @Email)",
            user);
}
```

---

## 연결 풀링

```csharp
// SQL Server 연결 문자열에서 풀링 설정
var connectionString = new SqlConnectionStringBuilder
{
    DataSource = "server",
    InitialCatalog = "database",
    MinPoolSize = 5,
    MaxPoolSize = 100,
    ConnectionTimeout = 30,
    Pooling = true
}.ToString();
```

---

## 면접 빈출 질문

1. **EF Core와 Dapper의 차이점은?**
2. **N+1 문제란? 어떻게 해결하나요?**
3. **연결 풀링이란?**
4. **DbContext의 생명주기는?**
5. **Repository 패턴을 사용하는 이유는?**

---

## 다음 단계

→ [07. Caching](../07-caching/) - 캐싱 전략
