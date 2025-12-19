# Kestrel 내부 동작

## 개요

Kestrel은 ASP.NET Core의 핵심 웹 서버로, **비동기 I/O**와 **이벤트 기반 아키텍처**를 사용하여 높은 성능을 달성합니다. 이 문서에서는 Kestrel이 요청을 처리하는 내부 구조를 살펴봅니다.

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kestrel 내부 구조                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Transport Layer                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │Socket Trans.│  │ Quic Trans. │  │ Named Pipe  │         │   │
│  │  │  (TCP/UDP)  │  │   (HTTP/3)  │  │  (Windows)  │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Connection Layer                          │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │              ConnectionContext                       │   │   │
│  │  │  - Transport (읽기/쓰기 파이프)                      │   │   │
│  │  │  - LocalEndPoint / RemoteEndPoint                    │   │   │
│  │  │  - ConnectionId                                      │   │   │
│  │  │  - Features                                          │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Protocol Layer                           │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐               │   │
│  │  │ HTTP/1.1  │  │  HTTP/2   │  │  HTTP/3   │               │   │
│  │  │  Parser   │  │  Parser   │  │  Parser   │               │   │
│  │  └───────────┘  └───────────┘  └───────────┘               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 Application Layer                           │   │
│  │                 (미들웨어 파이프라인)                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Transport Layer

### 진화 과정

```
.NET Core 1.x ~ 2.0        .NET Core 2.1+           .NET 5+
─────────────────────────────────────────────────────────────────
     libuv                  관리형 소켓              System.IO.Pipelines
  (Node.js와 동일)         (직접 구현)               + 관리형 소켓

  장점: 검증된 성능          장점:                    장점:
  단점: 네이티브 의존성       - 순수 .NET             - Zero-copy
                             - 디버깅 용이            - 고성능 버퍼 관리
                             - 메모리 제어            - 백프레셔 지원
```

### SocketTransport 동작 방식

```csharp
// 단순화된 소켓 수신 로직 (실제 코드는 더 복잡)
public class SocketListener
{
    private readonly Socket _listenSocket;

    public async Task AcceptAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            // 새 연결 수락 (비동기)
            var acceptSocket = await _listenSocket.AcceptAsync(cancellationToken);

            // 각 연결을 별도 Task로 처리 (블로킹하지 않음)
            _ = HandleConnectionAsync(acceptSocket, cancellationToken);
        }
    }

    private async Task HandleConnectionAsync(Socket socket, CancellationToken ct)
    {
        // System.IO.Pipelines를 사용한 데이터 처리
        var pipe = new Pipe();
        var reading = FillPipeAsync(socket, pipe.Writer, ct);
        var processing = ReadPipeAsync(pipe.Reader, ct);

        await Task.WhenAll(reading, processing);
    }
}
```

---

## System.IO.Pipelines

Kestrel의 핵심 성능 비결은 **System.IO.Pipelines**입니다.

### 기존 Stream 방식의 문제

```csharp
// 기존 Stream 방식 - 비효율적
byte[] buffer = new byte[4096]; // 매번 새 버퍼 할당
int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
// 버퍼 복사, GC 압력 증가
```

### Pipelines 방식

```csharp
// Pipelines - 효율적
public async Task ProcessDataAsync(PipeReader reader)
{
    while (true)
    {
        // Zero-copy: 버퍼를 복사하지 않고 직접 접근
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        // 데이터 처리
        ProcessBuffer(buffer);

        // 처리한 만큼만 "소비" 표시 - 메모리 재사용
        reader.AdvanceTo(buffer.End);

        if (result.IsCompleted)
            break;
    }
}
```

### Pipelines의 장점

```
┌─────────────────────────────────────────────────────────────────┐
│                    System.IO.Pipelines 장점                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Zero-copy (제로 카피)                                       │
│     - 데이터를 복사하지 않고 직접 버퍼 참조                      │
│     - Memory<T>, Span<T> 활용                                  │
│                                                                 │
│  2. 버퍼 풀링 (Buffer Pooling)                                  │
│     - ArrayPool<byte>에서 버퍼 재사용                           │
│     - GC 압력 감소                                              │
│                                                                 │
│  3. 백프레셔 (Backpressure)                                     │
│     - 느린 소비자가 빠른 생산자를 제어                          │
│     - 메모리 폭발 방지                                          │
│                                                                 │
│  4. 비동기 완료 (Async Completion)                              │
│     - await 가능한 읽기/쓰기                                    │
│     - ThreadPool 스레드 효율적 사용                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## HTTP 파서

### HTTP/1.1 파싱

```
HTTP/1.1 요청 구조:
─────────────────────────────────────────
GET /api/users HTTP/1.1\r\n           ← 요청 라인
Host: example.com\r\n                  ← 헤더
Content-Type: application/json\r\n
Content-Length: 13\r\n
\r\n                                   ← 헤더 끝
{"id": 123}                            ← 본문
```

```csharp
// 단순화된 HTTP/1.1 파서 로직
public struct HttpParser
{
    public bool ParseRequestLine(ReadOnlySpan<byte> line, out HttpMethod method,
                                  out string path, out HttpVersion version)
    {
        // "GET /api/users HTTP/1.1" 파싱
        int methodEnd = line.IndexOf((byte)' ');
        method = ParseMethod(line.Slice(0, methodEnd));

        int pathStart = methodEnd + 1;
        int pathEnd = line.Slice(pathStart).IndexOf((byte)' ');
        path = Encoding.ASCII.GetString(line.Slice(pathStart, pathEnd));

        version = ParseVersion(line.Slice(pathStart + pathEnd + 1));

        return true;
    }

    // 핵심: Span<T>을 사용하여 힙 할당 없이 파싱
    private HttpMethod ParseMethod(ReadOnlySpan<byte> method)
    {
        // 길이와 첫 바이트로 빠른 비교
        if (method.Length == 3 && method[0] == 'G')
            return HttpMethod.Get;
        if (method.Length == 4 && method[0] == 'P')
            return method[1] == 'O' ? HttpMethod.Post : HttpMethod.Put;
        // ...
    }
}
```

### HTTP/2 프레임 처리

```
HTTP/2 프레임 구조:
─────────────────────────────────────────
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+---------------+
|R|                 Stream Identifier (31)      |
+=+=============+===============================+
|                   Frame Payload               |
+-----------------------------------------------+
```

```csharp
// HTTP/2 스트림 멀티플렉싱
public class Http2Connection
{
    private readonly ConcurrentDictionary<int, Http2Stream> _streams = new();

    public async Task ProcessFrameAsync(Http2Frame frame)
    {
        switch (frame.Type)
        {
            case Http2FrameType.Headers:
                // 새 스트림 생성
                var stream = new Http2Stream(frame.StreamId);
                _streams.TryAdd(frame.StreamId, stream);
                await ProcessHeadersAsync(stream, frame);
                break;

            case Http2FrameType.Data:
                // 기존 스트림에 데이터 추가
                if (_streams.TryGetValue(frame.StreamId, out var existingStream))
                {
                    await existingStream.ReceiveDataAsync(frame.Payload);
                }
                break;

            case Http2FrameType.RstStream:
                // 스트림 종료
                _streams.TryRemove(frame.StreamId, out _);
                break;
        }
    }
}
```

---

## 연결 관리

### Connection Pool

```csharp
// 연결 풀 (단순화)
public class ConnectionPool
{
    private readonly ConcurrentBag<KestrelConnection> _pool = new();
    private readonly SemaphoreSlim _maxConnections;

    public ConnectionPool(int maxConnections)
    {
        _maxConnections = new SemaphoreSlim(maxConnections, maxConnections);
    }

    public async Task<KestrelConnection> GetConnectionAsync()
    {
        // 최대 연결 수 제한
        await _maxConnections.WaitAsync();

        if (_pool.TryTake(out var connection))
            return connection;

        return new KestrelConnection();
    }

    public void Return(KestrelConnection connection)
    {
        if (connection.IsReusable)
        {
            _pool.Add(connection);
        }
        _maxConnections.Release();
    }
}
```

### Keep-Alive 처리

```
Keep-Alive 연결 흐름:
─────────────────────────────────────────

클라이언트              Kestrel
    │                     │
    │──── 요청 1 ────────▶│
    │◀─── 응답 1 ─────────│
    │                     │
    │     (연결 유지)     │  ← KeepAliveTimeout 대기
    │                     │
    │──── 요청 2 ────────▶│
    │◀─── 응답 2 ─────────│
    │                     │
    │     (타임아웃)      │  ← KeepAliveTimeout 초과
    │                     │
    │◀─── 연결 종료 ──────│
```

---

## 스레드 모델

### I/O 스레드 vs Worker 스레드

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kestrel 스레드 모델                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  I/O Threads (소켓 처리)                                        │
│  ─────────────────────                                          │
│  - 소켓 읽기/쓰기                                               │
│  - 수가 적음 (CPU 코어 수 기반)                                 │
│  - 블로킹 금지!                                                 │
│                                                                 │
│  ThreadPool (앱 로직)                                           │
│  ────────────────────                                           │
│  - 미들웨어, 컨트롤러 실행                                       │
│  - 필요에 따라 동적 확장                                         │
│  - 블로킹 가능 (하지만 비권장)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 요청 처리 흐름

```csharp
// 1. 소켓에서 데이터 수신 (I/O 스레드)
await socket.ReceiveAsync(buffer);  // 비동기, 논블로킹

// 2. HTTP 파싱 (I/O 스레드)
var request = parser.Parse(buffer);

// 3. 앱 로직 실행 (ThreadPool)
await Task.Run(async () =>
{
    // 미들웨어 파이프라인 실행
    await middleware.InvokeAsync(httpContext);
});

// 4. 응답 전송 (I/O 스레드)
await socket.SendAsync(response);
```

---

## 메모리 관리

### 버퍼 풀링

```csharp
// Kestrel이 사용하는 메모리 풀
public class MemoryPoolFactory
{
    // PinnedBlockMemoryPool - 고정 메모리 블록
    // SlabMemoryPool - 큰 메모리를 작은 블록으로 분할

    public static MemoryPool<byte> Create()
    {
        // .NET 6+: PinnedBlockMemoryPool 사용
        // GC가 이동시키지 않는 고정 메모리
        return new PinnedBlockMemoryPool();
    }
}

// 사용 예
using var owner = pool.Rent(4096);
Memory<byte> buffer = owner.Memory;
// 사용 후 자동 반환 (using)
```

### 문자열 최적화

```csharp
// 헤더 파싱 시 문자열 인터닝
public class HeaderNames
{
    // 자주 사용되는 헤더는 미리 인터닝
    public static readonly string ContentType = "Content-Type";
    public static readonly string ContentLength = "Content-Length";
    public static readonly string Connection = "Connection";

    // 커스텀 헤더만 새 문자열 생성
}

// StringSegment를 사용한 부분 문자열 (할당 없음)
public readonly struct StringSegment
{
    public string Buffer { get; }
    public int Offset { get; }
    public int Length { get; }

    // 필요할 때만 ToString() 호출
}
```

---

## 성능 벤치마크 결과

### TechEmpower Benchmarks (Round 21)

```
Plaintext (req/sec):
─────────────────────────────────────────
ASP.NET Core    │████████████████████████│ 7,000,000+
Actix (Rust)    │███████████████████████ │ 6,800,000
Drogon (C++)    │██████████████████████  │ 6,500,000
Node.js         │██████                  │ 1,200,000

JSON (req/sec):
─────────────────────────────────────────
ASP.NET Core    │████████████████████████│ 1,500,000
Actix (Rust)    │███████████████████████ │ 1,400,000
Go (stdlib)     │████████████            │   600,000
Node.js         │███████                 │   300,000
```

### 최적화 요인

1. **Span<T>/Memory<T>**: 힙 할당 최소화
2. **System.IO.Pipelines**: 제로 카피 버퍼 관리
3. **ValueTask**: Task 할당 방지
4. **ref struct**: 스택 할당
5. **JIT 최적화**: 핫 경로 인라이닝

---

## 면접 예상 질문

### Q1: Kestrel이 고성능인 이유는?

**A:**
1. **System.IO.Pipelines**: 제로 카피 버퍼 관리, 백프레셔 지원
2. **Span<T>/Memory<T>**: 힙 할당 없이 데이터 처리
3. **비동기 I/O**: ThreadPool 스레드 효율적 사용
4. **버퍼 풀링**: ArrayPool을 통한 메모리 재사용
5. **HTTP/2 멀티플렉싱**: 하나의 연결에서 다중 스트림

### Q2: System.IO.Pipelines의 장점은?

**A:**
- **Zero-copy**: 데이터를 복사하지 않고 직접 버퍼 참조
- **버퍼 풀링**: GC 압력 감소
- **백프레셔**: 생산자-소비자 간 흐름 제어
- **비동기 완료**: await 가능한 API

### Q3: libuv에서 관리형 소켓으로 바꾼 이유는?

**A:**
- 네이티브 의존성 제거 (순수 .NET)
- 디버깅 용이성
- .NET 생태계 도구 활용 가능
- System.IO.Pipelines로 동등 이상의 성능 달성

---

## 참고 자료

- [Kestrel 소스 코드 (GitHub)](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel)
- [System.IO.Pipelines 소개](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/)
- [TechEmpower Benchmarks](https://www.techempower.com/benchmarks/)
- [High-Performance .NET - Steve Gordon](https://www.stevejgordon.co.uk/)

---

## 다음 문서

→ [iis-integration.md](../iis-integration.md) - IIS 연동 이해하기
