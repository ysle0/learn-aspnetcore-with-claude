# 16. Reflection Alternatives - 리플렉션 대안

Expression Trees, Source Generators, IL Emit을 이용한 리플렉션 대안을 이해합니다.

---

## 성능 비교

| 방식 | 상대 성능 | 사용 시점 |
|------|----------|----------|
| 직접 호출 | 1x | 컴파일 타임에 알려진 경우 |
| Source Generator | 1x | 컴파일 타임 코드 생성 |
| Expression.Compile | 2-3x | 런타임 한 번 컴파일 |
| IL Emit | 2-3x | 동적 메서드 생성 |
| Reflection | 10-100x | 최후의 수단 |

---

## 예시

### Expression Trees

```csharp
// 프로퍼티 getter 캐싱
public static Func<T, TValue> CreateGetter<T, TValue>(string propertyName)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(param, propertyName);
    var convert = Expression.Convert(property, typeof(TValue));
    return Expression.Lambda<Func<T, TValue>>(convert, param).Compile();
}

// 사용
var getId = CreateGetter<User, int>("Id");
int id = getId(user);  // Reflection보다 100배 빠름
```

---

## 다음 단계

→ [17. Aspire](../17-aspire/) - .NET Aspire
