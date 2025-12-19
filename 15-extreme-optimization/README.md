# 15. Extreme Optimization - 극한 최적화

Span<T>, Memory<T>, unsafe 코드를 이용한 고성능 최적화를 이해합니다.

---

## 핵심 개념

### Span<T>

```csharp
// 힙 할당 없이 배열/문자열 슬라이싱
ReadOnlySpan<char> span = "Hello World".AsSpan();
ReadOnlySpan<char> hello = span[..5];     // "Hello"
ReadOnlySpan<char> world = span[6..];     // "World"

// 스택 할당
Span<int> numbers = stackalloc int[100];
numbers[0] = 42;
```

### Memory<T>

```csharp
// 비동기 메서드에서 사용 가능 (Span은 불가)
Memory<byte> buffer = new byte[1024];
await stream.ReadAsync(buffer);
```

### ArrayPool<T>

```csharp
// 배열 재사용으로 GC 압력 감소
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(1024);
try
{
    // buffer 사용
}
finally
{
    pool.Return(buffer);
}
```

### unsafe 코드

```csharp
unsafe
{
    int* p = stackalloc int[10];
    p[0] = 42;
}
```

---

## 다음 단계

→ [16. Reflection Alternatives](../16-reflection-alternatives/) - 리플렉션 대안
