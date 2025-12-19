# 12. TAP Internals - 비동기 프로그래밍 내부

Task-based Asynchronous Pattern의 내부 동작을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [async-await.md](./async-await.md) | async/await 기초 | 표면 |
| [state-machine.md](./state-machine.md) | 상태 머신 내부 | 깊음 |
| [synchronization-context.md](./synchronization-context.md) | 동기화 컨텍스트 | 깊음 |
| [valuetask.md](./valuetask.md) | ValueTask 최적화 | 실용 |

---

## 핵심 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                    async/await 변환                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  소스 코드:                                                     │
│  async Task<int> GetDataAsync()                                │
│  {                                                              │
│      var data = await FetchAsync();                            │
│      return Process(data);                                      │
│  }                                                              │
│                                                                 │
│       │ 컴파일러 변환                                           │
│       ▼                                                         │
│                                                                 │
│  상태 머신 클래스:                                               │
│  struct GetDataAsync_StateMachine : IAsyncStateMachine          │
│  {                                                              │
│      int state;           // 현재 상태                          │
│      TaskAwaiter awaiter; // 대기자                             │
│      int result;          // 결과                               │
│                                                                 │
│      void MoveNext()                                           │
│      {                                                          │
│          switch(state)                                          │
│          {                                                      │
│              case 0: // 시작                                    │
│              case 1: // await 후                                │
│          }                                                      │
│      }                                                          │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 빠른 예제

### 기본 async/await

```csharp
public async Task<string> GetDataAsync()
{
    // Task가 완료되지 않으면 여기서 반환
    // 완료 시 MoveNext() 콜백으로 재개
    var data = await httpClient.GetStringAsync(url);

    return data.ToUpper();
}
```

### ConfigureAwait

```csharp
// 라이브러리 코드에서 권장
public async Task<string> GetDataAsync()
{
    var data = await httpClient.GetStringAsync(url)
        .ConfigureAwait(false);  // 원래 컨텍스트로 돌아가지 않음

    return data.ToUpper();
}
```

### ValueTask

```csharp
// 자주 동기적으로 완료되는 경우 최적화
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return ValueTask.FromResult(value);  // 할당 없음

    return new ValueTask<int>(LoadFromDbAsync(key));
}
```

---

## 면접 빈출 질문

1. **async/await 내부 동작은?**
2. **ConfigureAwait(false)를 사용하는 이유는?**
3. **ValueTask vs Task 차이는?**
4. **SynchronizationContext란?**

---

## 다음 단계

→ [13. Rate Limiting](../13-rate-limiting/) - 요청 제한
