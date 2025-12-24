# gRPC Integration Testing

ASP.NET Core gRPC 서비스의 통합 테스트를 다룹니다.

## 테스트 환경 설정

### 필요 패키지

```xml
<PackageReference Include="Grpc.Net.Client" Version="2.59.0" />
<PackageReference Include="Grpc.Tools" Version="2.60.0" PrivateAssets="All" />
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
<PackageReference Include="xunit" Version="2.6.2" />
<PackageReference Include="FluentAssertions" Version="6.12.0" />
```

### Proto 파일 정의

```protobuf
// Protos/greet.proto
syntax = "proto3";

option csharp_namespace = "GrpcService";

package greet;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc SayHelloServerStream (HelloRequest) returns (stream HelloReply);
  rpc SayHelloClientStream (stream HelloRequest) returns (HelloReply);
  rpc SayHelloBidirectional (stream HelloRequest) returns (stream HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

### 테스트 대상 서비스

```csharp
// GreeterService.cs
public class GreeterService : Greeter.GreeterBase
{
    private readonly ILogger<GreeterService> _logger;

    public GreeterService(ILogger<GreeterService> logger)
    {
        _logger = logger;
    }

    public override Task<HelloReply> SayHello(
        HelloRequest request,
        ServerCallContext context)
    {
        _logger.LogInformation("Greeting {Name}", request.Name);

        return Task.FromResult(new HelloReply
        {
            Message = $"Hello {request.Name}"
        });
    }

    public override async Task SayHelloServerStream(
        HelloRequest request,
        IServerStreamWriter<HelloReply> responseStream,
        ServerCallContext context)
    {
        for (int i = 1; i <= 5; i++)
        {
            await responseStream.WriteAsync(new HelloReply
            {
                Message = $"Hello {request.Name} #{i}"
            });
            await Task.Delay(100);
        }
    }

    public override async Task<HelloReply> SayHelloClientStream(
        IAsyncStreamReader<HelloRequest> requestStream,
        ServerCallContext context)
    {
        var names = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync())
        {
            names.Add(request.Name);
        }

        return new HelloReply
        {
            Message = $"Hello {string.Join(", ", names)}"
        };
    }

    public override async Task SayHelloBidirectional(
        IAsyncStreamReader<HelloRequest> requestStream,
        IServerStreamWriter<HelloReply> responseStream,
        ServerCallContext context)
    {
        await foreach (var request in requestStream.ReadAllAsync())
        {
            await responseStream.WriteAsync(new HelloReply
            {
                Message = $"Hello {request.Name}"
            });
        }
    }
}
```

---

## GrpcTestFixture 설정

### WebApplicationFactory 확장

```csharp
public class GrpcTestFixture<TStartup> : WebApplicationFactory<TStartup>
    where TStartup : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 테스트용 서비스 구성
        });
    }

    public GrpcChannel CreateGrpcChannel()
    {
        var client = CreateDefaultClient();

        return GrpcChannel.ForAddress(client.BaseAddress!, new GrpcChannelOptions
        {
            HttpHandler = Server.CreateHandler()
        });
    }

    public T CreateGrpcClient<T>() where T : ClientBase<T>
    {
        var channel = CreateGrpcChannel();

        // Activator로 클라이언트 생성
        return (T)Activator.CreateInstance(typeof(T), channel)!;
    }
}
```

### HTTP/2 지원 설정

```csharp
public class GrpcIntegrationTestFixture : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureKestrel(options =>
        {
            // HTTP/2 전용 엔드포인트
            options.ListenLocalhost(5001, o => o.Protocols = HttpProtocols.Http2);
        });

        builder.ConfigureServices(services =>
        {
            // 테스트용 서비스 교체
        });
    }

    public GrpcChannel CreateChannel()
    {
        var httpHandler = new HttpClientHandler();

        // 개발 인증서 무시
        httpHandler.ServerCertificateCustomValidationCallback =
            HttpClientHandler.DangerousAcceptAnyServerCertificateValidator;

        return GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
        {
            HttpHandler = httpHandler
        });
    }
}
```

### In-Memory 테스트 (권장)

```csharp
public class GrpcInMemoryFixture : IAsyncLifetime
{
    private readonly WebApplicationFactory<Program> _factory;
    private GrpcChannel? _channel;

    public GrpcInMemoryFixture()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // 필요한 서비스 Mock 교체
                });
            });
    }

    public GrpcChannel Channel => _channel!;

    public T CreateClient<T>() where T : ClientBase<T>
    {
        return (T)Activator.CreateInstance(typeof(T), Channel)!;
    }

    public async Task InitializeAsync()
    {
        // 팩토리 초기화
        var client = _factory.CreateDefaultClient();

        _channel = GrpcChannel.ForAddress(client.BaseAddress!, new GrpcChannelOptions
        {
            HttpHandler = _factory.Server.CreateHandler()
        });

        await Task.CompletedTask;
    }

    public async Task DisposeAsync()
    {
        _channel?.Dispose();
        await _factory.DisposeAsync();
    }
}
```

---

## Unary RPC 테스트

```csharp
public class UnaryRpcTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public UnaryRpcTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHello_ValidName_ReturnsGreeting()
    {
        // Arrange
        var request = new HelloRequest { Name = "World" };

        // Act
        var response = await _client.SayHelloAsync(request);

        // Assert
        response.Message.Should().Be("Hello World");
    }

    [Fact]
    public async Task SayHello_EmptyName_ReturnsGreeting()
    {
        // Arrange
        var request = new HelloRequest { Name = "" };

        // Act
        var response = await _client.SayHelloAsync(request);

        // Assert
        response.Message.Should().Be("Hello ");
    }

    [Fact]
    public async Task SayHello_WithDeadline_CompletesBeforeDeadline()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };
        var deadline = DateTime.UtcNow.AddSeconds(5);

        // Act
        var response = await _client.SayHelloAsync(
            request,
            deadline: deadline);

        // Assert
        response.Should().NotBeNull();
    }

    [Fact]
    public async Task SayHello_WithCancellation_CanBeCancelled()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };
        var cts = new CancellationTokenSource();
        cts.Cancel(); // 즉시 취소

        // Act & Assert
        await Assert.ThrowsAsync<RpcException>(async () =>
            await _client.SayHelloAsync(request, cancellationToken: cts.Token));
    }
}
```

---

## Server Streaming 테스트

```csharp
public class ServerStreamingTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public ServerStreamingTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHelloServerStream_ReceivesAllMessages()
    {
        // Arrange
        var request = new HelloRequest { Name = "World" };
        var messages = new List<string>();

        // Act
        using var call = _client.SayHelloServerStream(request);

        await foreach (var response in call.ResponseStream.ReadAllAsync())
        {
            messages.Add(response.Message);
        }

        // Assert
        messages.Should().HaveCount(5);
        messages.Should().AllSatisfy(m => m.Should().StartWith("Hello World"));
        messages[0].Should().EndWith("#1");
        messages[4].Should().EndWith("#5");
    }

    [Fact]
    public async Task SayHelloServerStream_CanCancelMidStream()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };
        var cts = new CancellationTokenSource();
        var receivedCount = 0;

        // Act
        using var call = _client.SayHelloServerStream(request);

        try
        {
            await foreach (var response in call.ResponseStream.ReadAllAsync(cts.Token))
            {
                receivedCount++;
                if (receivedCount >= 2)
                {
                    cts.Cancel();
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 예상된 예외
        }

        // Assert
        receivedCount.Should().BeGreaterOrEqualTo(2);
        receivedCount.Should().BeLessThan(5);
    }

    [Fact]
    public async Task SayHelloServerStream_WithHeaders_ReceivesHeaders()
    {
        // Arrange
        var request = new HelloRequest { Name = "World" };
        var headers = new Metadata
        {
            { "custom-header", "test-value" }
        };

        // Act
        using var call = _client.SayHelloServerStream(request, headers);
        var responseHeaders = await call.ResponseHeadersAsync;

        // Assert - 서버가 헤더를 에코하도록 구현되어 있다면
        // responseHeaders.Should().Contain(h => h.Key == "custom-header");

        // 스트림 소비
        await foreach (var _ in call.ResponseStream.ReadAllAsync()) { }
    }
}
```

---

## Client Streaming 테스트

```csharp
public class ClientStreamingTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public ClientStreamingTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHelloClientStream_SendsMultipleRequests_ReceivesCombinedResponse()
    {
        // Arrange
        var names = new[] { "Alice", "Bob", "Charlie" };

        // Act
        using var call = _client.SayHelloClientStream();

        foreach (var name in names)
        {
            await call.RequestStream.WriteAsync(new HelloRequest { Name = name });
        }

        await call.RequestStream.CompleteAsync();

        var response = await call;

        // Assert
        response.Message.Should().Be("Hello Alice, Bob, Charlie");
    }

    [Fact]
    public async Task SayHelloClientStream_EmptyStream_ReturnsEmptyGreeting()
    {
        // Act
        using var call = _client.SayHelloClientStream();
        await call.RequestStream.CompleteAsync();

        var response = await call;

        // Assert
        response.Message.Should().Be("Hello ");
    }

    [Fact]
    public async Task SayHelloClientStream_ManyMessages_HandlesAll()
    {
        // Arrange
        const int messageCount = 100;

        // Act
        using var call = _client.SayHelloClientStream();

        for (int i = 0; i < messageCount; i++)
        {
            await call.RequestStream.WriteAsync(
                new HelloRequest { Name = $"User{i}" });
        }

        await call.RequestStream.CompleteAsync();

        var response = await call;

        // Assert
        response.Message.Should().Contain("User0");
        response.Message.Should().Contain("User99");
    }
}
```

---

## Bidirectional Streaming 테스트

```csharp
public class BidirectionalStreamingTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public BidirectionalStreamingTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHelloBidirectional_EchosMessages()
    {
        // Arrange
        var names = new[] { "Alice", "Bob", "Charlie" };
        var responses = new List<string>();

        // Act
        using var call = _client.SayHelloBidirectional();

        // 병렬로 읽기/쓰기
        var readTask = Task.Run(async () =>
        {
            await foreach (var response in call.ResponseStream.ReadAllAsync())
            {
                responses.Add(response.Message);
            }
        });

        foreach (var name in names)
        {
            await call.RequestStream.WriteAsync(new HelloRequest { Name = name });
            await Task.Delay(50); // 서버가 처리할 시간
        }

        await call.RequestStream.CompleteAsync();
        await readTask;

        // Assert
        responses.Should().HaveCount(3);
        responses[0].Should().Be("Hello Alice");
        responses[1].Should().Be("Hello Bob");
        responses[2].Should().Be("Hello Charlie");
    }

    [Fact]
    public async Task SayHelloBidirectional_InterleavedSendReceive()
    {
        // Arrange & Act
        using var call = _client.SayHelloBidirectional();

        // 보내고 바로 받기
        await call.RequestStream.WriteAsync(new HelloRequest { Name = "First" });

        // 응답 읽기 시작
        var moveNext = call.ResponseStream.MoveNext(CancellationToken.None);

        // 더 보내기
        await call.RequestStream.WriteAsync(new HelloRequest { Name = "Second" });

        // 첫 번째 응답 확인
        await moveNext;
        var first = call.ResponseStream.Current;
        first.Message.Should().Be("Hello First");

        // 두 번째 응답
        await call.ResponseStream.MoveNext(CancellationToken.None);
        var second = call.ResponseStream.Current;
        second.Message.Should().Be("Hello Second");

        await call.RequestStream.CompleteAsync();
    }
}
```

---

## 에러 처리 테스트

```csharp
public class GrpcErrorTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public GrpcErrorTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHello_InvalidArgument_ThrowsRpcException()
    {
        // Arrange - 서버가 특정 이름에 대해 InvalidArgument를 반환하도록 구현되어 있다고 가정
        var request = new HelloRequest { Name = "INVALID" };

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => _client.SayHelloAsync(request).ResponseAsync);

        exception.StatusCode.Should().Be(StatusCode.InvalidArgument);
    }

    [Fact]
    public async Task SayHello_NotFound_ThrowsRpcException()
    {
        // Arrange
        var request = new HelloRequest { Name = "NOTFOUND" };

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => _client.SayHelloAsync(request).ResponseAsync);

        exception.StatusCode.Should().Be(StatusCode.NotFound);
    }

    [Fact]
    public async Task SayHello_DeadlineExceeded_ThrowsRpcException()
    {
        // Arrange - 매우 짧은 deadline
        var request = new HelloRequest { Name = "SlowOperation" };
        var deadline = DateTime.UtcNow.AddMilliseconds(1);

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => _client.SayHelloAsync(request, deadline: deadline).ResponseAsync);

        exception.StatusCode.Should().Be(StatusCode.DeadlineExceeded);
    }
}
```

---

## 인증 테스트

```csharp
public class AuthenticatedGrpcTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly GrpcInMemoryFixture _fixture;

    public AuthenticatedGrpcTests(GrpcInMemoryFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public async Task SecureMethod_WithValidToken_Succeeds()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken();
        var headers = new Metadata
        {
            { "Authorization", $"Bearer {token}" }
        };

        var client = _fixture.CreateClient<SecureService.SecureServiceClient>();

        // Act
        var response = await client.GetSecureDataAsync(
            new SecureRequest(),
            headers);

        // Assert
        response.Should().NotBeNull();
    }

    [Fact]
    public async Task SecureMethod_WithoutToken_ThrowsUnauthenticated()
    {
        // Arrange
        var client = _fixture.CreateClient<SecureService.SecureServiceClient>();

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => client.GetSecureDataAsync(new SecureRequest()).ResponseAsync);

        exception.StatusCode.Should().Be(StatusCode.Unauthenticated);
    }

    [Fact]
    public async Task AdminMethod_WithUserRole_ThrowsPermissionDenied()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken(roles: new[] { "User" });
        var headers = new Metadata
        {
            { "Authorization", $"Bearer {token}" }
        };

        var client = _fixture.CreateClient<SecureService.SecureServiceClient>();

        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => client.AdminOnlyMethodAsync(new AdminRequest(), headers).ResponseAsync);

        exception.StatusCode.Should().Be(StatusCode.PermissionDenied);
    }
}
```

---

## Metadata/Headers 테스트

```csharp
public class MetadataTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public MetadataTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHello_WithCustomHeaders_ServerReceivesHeaders()
    {
        // Arrange
        var headers = new Metadata
        {
            { "x-custom-header", "custom-value" },
            { "x-request-id", Guid.NewGuid().ToString() }
        };

        // Act
        using var call = _client.SayHelloAsync(
            new HelloRequest { Name = "Test" },
            headers);

        var response = await call;

        // Assert
        response.Should().NotBeNull();
    }

    [Fact]
    public async Task SayHello_GetResponseHeaders()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };

        // Act
        using var call = _client.SayHelloAsync(request);
        var responseHeaders = await call.ResponseHeadersAsync;
        var response = await call;

        // Assert
        response.Should().NotBeNull();
        // 서버가 특정 헤더를 반환하도록 구현되어 있다면 확인
    }

    [Fact]
    public async Task SayHello_GetTrailers()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };

        // Act
        using var call = _client.SayHelloAsync(request);
        var response = await call;
        var trailers = call.GetTrailers();

        // Assert
        // 서버가 trailer를 반환하도록 구현되어 있다면 확인
        trailers.Should().NotBeNull();
    }
}
```

---

## 성능 테스트

```csharp
public class GrpcPerformanceTests : IClassFixture<GrpcInMemoryFixture>
{
    private readonly Greeter.GreeterClient _client;

    public GrpcPerformanceTests(GrpcInMemoryFixture fixture)
    {
        _client = fixture.CreateClient<Greeter.GreeterClient>();
    }

    [Fact]
    public async Task SayHello_ManyRequests_CompletesQuickly()
    {
        // Arrange
        const int requestCount = 100;
        var stopwatch = Stopwatch.StartNew();

        // Act
        var tasks = Enumerable.Range(0, requestCount)
            .Select(i => _client.SayHelloAsync(
                new HelloRequest { Name = $"User{i}" }).ResponseAsync);

        await Task.WhenAll(tasks);
        stopwatch.Stop();

        // Assert
        stopwatch.ElapsedMilliseconds.Should().BeLessThan(5000);
    }

    [Fact]
    public async Task ServerStream_LargeStream_CompletesWithinTimeout()
    {
        // Arrange
        var request = new HelloRequest { Name = "Test" };
        var messageCount = 0;
        var stopwatch = Stopwatch.StartNew();

        // Act
        using var call = _client.SayHelloServerStream(request);

        await foreach (var _ in call.ResponseStream.ReadAllAsync())
        {
            messageCount++;
        }

        stopwatch.Stop();

        // Assert
        messageCount.Should().BeGreaterThan(0);
        stopwatch.ElapsedMilliseconds.Should().BeLessThan(2000);
    }
}
```

---

## 면접 질문

### Q1: gRPC 서비스를 어떻게 통합 테스트하나요?
**A**: `WebApplicationFactory`를 사용하여 테스트 서버를 생성하고, `GrpcChannel.ForAddress`에 테스트 서버의 `HttpHandler`를 전달하여 실제 네트워크 없이 gRPC 클라이언트로 테스트합니다.

### Q2: gRPC의 4가지 통신 패턴을 테스트할 때 주의점은?
**A**:
- **Unary**: 일반 async/await 사용
- **Server Streaming**: `ResponseStream.ReadAllAsync()`로 모든 메시지 수신
- **Client Streaming**: `RequestStream.CompleteAsync()` 호출 필수
- **Bidirectional**: 읽기/쓰기를 병렬로 처리, 데드락 주의

### Q3: gRPC 에러 처리는 어떻게 테스트하나요?
**A**: `RpcException`을 catch하고 `StatusCode`를 검증합니다. `StatusCode.InvalidArgument`, `StatusCode.NotFound`, `StatusCode.Unauthenticated` 등 다양한 상태 코드를 테스트합니다.
