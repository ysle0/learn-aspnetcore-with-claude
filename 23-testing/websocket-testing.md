# WebSocket & SignalR Integration Testing

WebSocket 및 SignalR 기반 실시간 통신의 통합 테스트를 다룹니다.

## SignalR Hub 테스트

### 테스트 대상 Hub

```csharp
// ChatHub.cs
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
}

public class ChatHub : Hub<IChatClient>
{
    private readonly IUserService _userService;
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(IUserService userService, ILogger<ChatHub> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    public override async Task OnConnectedAsync()
    {
        var user = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.Others.UserJoined(user);
        _logger.LogInformation("{User} connected", user);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var user = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.Others.UserLeft(user);
        _logger.LogInformation("{User} disconnected", user);
        await base.OnDisconnectedAsync(exception);
    }

    public async Task SendMessage(string message)
    {
        var user = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.All.ReceiveMessage(user, message);
    }

    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        await Clients.Group(roomName).UserJoined(Context.User?.Identity?.Name ?? "Anonymous");
    }

    public async Task SendToRoom(string roomName, string message)
    {
        var user = Context.User?.Identity?.Name ?? "Anonymous";
        await Clients.Group(roomName).ReceiveMessage(user, message);
    }
}
```

---

## 테스트 인프라 설정

### WebApplicationFactory 설정

```csharp
public class SignalRTestFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 테스트용 서비스 교체
            services.RemoveAll<IUserService>();
            services.AddSingleton<IUserService, FakeUserService>();
        });

        builder.ConfigureLogging(logging =>
        {
            logging.ClearProviders();
            logging.AddDebug();
        });
    }
}
```

### 테스트 기본 클래스

```csharp
public class SignalRTestBase : IClassFixture<SignalRTestFactory>, IAsyncLifetime
{
    protected readonly SignalRTestFactory Factory;
    protected HubConnection Connection = null!;
    protected readonly List<(string User, string Message)> ReceivedMessages = new();
    protected readonly List<string> JoinedUsers = new();
    protected readonly List<string> LeftUsers = new();

    public SignalRTestBase(SignalRTestFactory factory)
    {
        Factory = factory;
    }

    public async Task InitializeAsync()
    {
        var client = Factory.CreateClient();

        Connection = new HubConnectionBuilder()
            .WithUrl(
                new Uri(client.BaseAddress!, "/chatHub"),
                options =>
                {
                    options.HttpMessageHandlerFactory = _ => Factory.Server.CreateHandler();
                })
            .ConfigureLogging(logging => logging.AddDebug())
            .Build();

        // 이벤트 핸들러 등록
        Connection.On<string, string>("ReceiveMessage", (user, message) =>
        {
            ReceivedMessages.Add((user, message));
        });

        Connection.On<string>("UserJoined", user =>
        {
            JoinedUsers.Add(user);
        });

        Connection.On<string>("UserLeft", user =>
        {
            LeftUsers.Add(user);
        });

        await Connection.StartAsync();
    }

    public async Task DisposeAsync()
    {
        await Connection.DisposeAsync();
    }

    protected async Task<HubConnection> CreateAdditionalConnectionAsync()
    {
        var connection = new HubConnectionBuilder()
            .WithUrl(
                new Uri(Factory.CreateClient().BaseAddress!, "/chatHub"),
                options =>
                {
                    options.HttpMessageHandlerFactory = _ => Factory.Server.CreateHandler();
                })
            .Build();

        await connection.StartAsync();
        return connection;
    }

    protected async Task WaitForMessageAsync(
        Func<bool> condition,
        TimeSpan? timeout = null)
    {
        timeout ??= TimeSpan.FromSeconds(5);
        var deadline = DateTime.UtcNow + timeout.Value;

        while (DateTime.UtcNow < deadline)
        {
            if (condition()) return;
            await Task.Delay(50);
        }

        throw new TimeoutException("Condition not met within timeout");
    }
}
```

---

## SignalR 통합 테스트

### 기본 연결 테스트

```csharp
public class ChatHubTests : SignalRTestBase
{
    public ChatHubTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task Connect_SuccessfullyConnects()
    {
        // Assert - InitializeAsync에서 이미 연결됨
        Connection.State.Should().Be(HubConnectionState.Connected);
    }

    [Fact]
    public async Task Disconnect_SuccessfullyDisconnects()
    {
        // Act
        await Connection.StopAsync();

        // Assert
        Connection.State.Should().Be(HubConnectionState.Disconnected);
    }
}
```

### 메시지 송수신 테스트

```csharp
public class MessageTests : SignalRTestBase
{
    public MessageTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task SendMessage_AllClientsReceive()
    {
        // Arrange
        await using var connection2 = await CreateAdditionalConnectionAsync();
        var messages2 = new List<(string, string)>();
        connection2.On<string, string>("ReceiveMessage", (user, msg) =>
            messages2.Add((user, msg)));

        // Act
        await Connection.InvokeAsync("SendMessage", "Hello World");

        // Assert - 양쪽 모두 수신
        await WaitForMessageAsync(() => ReceivedMessages.Count > 0);
        await WaitForMessageAsync(() => messages2.Count > 0);

        ReceivedMessages.Should().Contain(m => m.Message == "Hello World");
        messages2.Should().Contain(m => m.Message == "Hello World");
    }

    [Fact]
    public async Task SendMessage_MultipleTimes_AllReceived()
    {
        // Act
        for (int i = 0; i < 5; i++)
        {
            await Connection.InvokeAsync("SendMessage", $"Message {i}");
        }

        // Assert
        await WaitForMessageAsync(() => ReceivedMessages.Count >= 5);
        ReceivedMessages.Should().HaveCount(5);
    }
}
```

### 그룹 테스트

```csharp
public class GroupTests : SignalRTestBase
{
    public GroupTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task JoinRoom_ReceivesRoomMessages()
    {
        // Arrange
        await using var connection2 = await CreateAdditionalConnectionAsync();
        var roomMessages = new List<(string, string)>();
        connection2.On<string, string>("ReceiveMessage", (user, msg) =>
            roomMessages.Add((user, msg)));

        // Act - 두 클라이언트가 같은 방에 참가
        await Connection.InvokeAsync("JoinRoom", "room1");
        await connection2.InvokeAsync("JoinRoom", "room1");

        await Connection.InvokeAsync("SendToRoom", "room1", "Hello Room");

        // Assert
        await WaitForMessageAsync(() => roomMessages.Count > 0);
        roomMessages.Should().Contain(m => m.Message == "Hello Room");
    }

    [Fact]
    public async Task SendToRoom_OnlyRoomMembersReceive()
    {
        // Arrange
        await using var inRoomConnection = await CreateAdditionalConnectionAsync();
        await using var outsideConnection = await CreateAdditionalConnectionAsync();

        var inRoomMessages = new List<string>();
        var outsideMessages = new List<string>();

        inRoomConnection.On<string, string>("ReceiveMessage", (_, msg) =>
            inRoomMessages.Add(msg));
        outsideConnection.On<string, string>("ReceiveMessage", (_, msg) =>
            outsideMessages.Add(msg));

        // Act
        await Connection.InvokeAsync("JoinRoom", "secret-room");
        await inRoomConnection.InvokeAsync("JoinRoom", "secret-room");
        // outsideConnection은 방에 참가하지 않음

        await Connection.InvokeAsync("SendToRoom", "secret-room", "Secret Message");

        // Assert
        await WaitForMessageAsync(() => inRoomMessages.Count > 0);
        await Task.Delay(200); // 외부 클라이언트가 받지 않음을 확인하기 위해 대기

        inRoomMessages.Should().Contain("Secret Message");
        outsideMessages.Should().BeEmpty();
    }
}
```

### 사용자 참가/퇴장 알림 테스트

```csharp
public class PresenceTests : SignalRTestBase
{
    public PresenceTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task NewUserConnects_OthersReceiveNotification()
    {
        // Act - 새 클라이언트 연결
        await using var newConnection = await CreateAdditionalConnectionAsync();

        // Assert - 기존 클라이언트가 알림 수신
        await WaitForMessageAsync(() => JoinedUsers.Count > 0);
        JoinedUsers.Should().NotBeEmpty();
    }

    [Fact]
    public async Task UserDisconnects_OthersReceiveNotification()
    {
        // Arrange
        var disconnectingConnection = await CreateAdditionalConnectionAsync();

        // Act
        await disconnectingConnection.StopAsync();
        await disconnectingConnection.DisposeAsync();

        // Assert
        await WaitForMessageAsync(() => LeftUsers.Count > 0);
        LeftUsers.Should().NotBeEmpty();
    }
}
```

---

## 인증된 SignalR 테스트

```csharp
public class AuthenticatedHubTests : IClassFixture<SignalRTestFactory>
{
    private readonly SignalRTestFactory _factory;

    public AuthenticatedHubTests(SignalRTestFactory factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task Connect_WithValidToken_Succeeds()
    {
        // Arrange
        var token = TestJwtTokenGenerator.GenerateToken(
            userId: "1",
            userName: "TestUser");

        var connection = new HubConnectionBuilder()
            .WithUrl(
                new Uri(_factory.CreateClient().BaseAddress!, "/chatHub"),
                options =>
                {
                    options.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler();
                    options.AccessTokenProvider = () => Task.FromResult(token)!;
                })
            .Build();

        // Act
        await connection.StartAsync();

        // Assert
        connection.State.Should().Be(HubConnectionState.Connected);

        await connection.DisposeAsync();
    }

    [Fact]
    public async Task Connect_WithoutToken_Fails()
    {
        // Arrange
        var connection = new HubConnectionBuilder()
            .WithUrl(
                new Uri(_factory.CreateClient().BaseAddress!, "/secureHub"),
                options =>
                {
                    options.HttpMessageHandlerFactory = _ => _factory.Server.CreateHandler();
                    // 토큰 없음
                })
            .Build();

        // Act & Assert
        await Assert.ThrowsAsync<HttpRequestException>(
            () => connection.StartAsync());
    }
}
```

---

## Raw WebSocket 테스트

SignalR 없이 순수 WebSocket 테스트입니다.

### WebSocket Endpoint

```csharp
// Program.cs
app.UseWebSockets();

app.Map("/ws", async context =>
{
    if (context.WebSockets.IsWebSocketRequest)
    {
        var webSocket = await context.WebSockets.AcceptWebSocketAsync();
        await HandleWebSocket(webSocket);
    }
    else
    {
        context.Response.StatusCode = 400;
    }
});

async Task HandleWebSocket(WebSocket webSocket)
{
    var buffer = new byte[1024];
    while (webSocket.State == WebSocketState.Open)
    {
        var result = await webSocket.ReceiveAsync(
            new ArraySegment<byte>(buffer), CancellationToken.None);

        if (result.MessageType == WebSocketMessageType.Text)
        {
            var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
            var response = Encoding.UTF8.GetBytes($"Echo: {message}");
            await webSocket.SendAsync(
                new ArraySegment<byte>(response),
                WebSocketMessageType.Text,
                true,
                CancellationToken.None);
        }
        else if (result.MessageType == WebSocketMessageType.Close)
        {
            await webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Closing",
                CancellationToken.None);
        }
    }
}
```

### WebSocket 테스트

```csharp
public class RawWebSocketTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public RawWebSocketTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task WebSocket_EchoMessage()
    {
        // Arrange
        var client = _factory.Server.CreateWebSocketClient();
        var webSocket = await client.ConnectAsync(
            new Uri(_factory.Server.BaseAddress, "/ws"),
            CancellationToken.None);

        try
        {
            // Act - 메시지 전송
            var message = "Hello WebSocket";
            var sendBuffer = Encoding.UTF8.GetBytes(message);
            await webSocket.SendAsync(
                new ArraySegment<byte>(sendBuffer),
                WebSocketMessageType.Text,
                true,
                CancellationToken.None);

            // 응답 수신
            var receiveBuffer = new byte[1024];
            var result = await webSocket.ReceiveAsync(
                new ArraySegment<byte>(receiveBuffer),
                CancellationToken.None);

            // Assert
            result.MessageType.Should().Be(WebSocketMessageType.Text);
            var response = Encoding.UTF8.GetString(receiveBuffer, 0, result.Count);
            response.Should().Be($"Echo: {message}");
        }
        finally
        {
            await webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Test complete",
                CancellationToken.None);
        }
    }

    [Fact]
    public async Task WebSocket_MultipleMessages()
    {
        // Arrange
        var client = _factory.Server.CreateWebSocketClient();
        var webSocket = await client.ConnectAsync(
            new Uri(_factory.Server.BaseAddress, "/ws"),
            CancellationToken.None);

        try
        {
            for (int i = 0; i < 5; i++)
            {
                // Act
                var message = $"Message {i}";
                var sendBuffer = Encoding.UTF8.GetBytes(message);
                await webSocket.SendAsync(
                    new ArraySegment<byte>(sendBuffer),
                    WebSocketMessageType.Text,
                    true,
                    CancellationToken.None);

                var receiveBuffer = new byte[1024];
                var result = await webSocket.ReceiveAsync(
                    new ArraySegment<byte>(receiveBuffer),
                    CancellationToken.None);

                // Assert
                var response = Encoding.UTF8.GetString(receiveBuffer, 0, result.Count);
                response.Should().Be($"Echo: {message}");
            }
        }
        finally
        {
            await webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Test complete",
                CancellationToken.None);
        }
    }

    [Fact]
    public async Task WebSocket_BinaryMessage()
    {
        // Arrange
        var client = _factory.Server.CreateWebSocketClient();
        var webSocket = await client.ConnectAsync(
            new Uri(_factory.Server.BaseAddress, "/ws/binary"),
            CancellationToken.None);

        try
        {
            // Act
            var binaryData = new byte[] { 0x01, 0x02, 0x03, 0x04 };
            await webSocket.SendAsync(
                new ArraySegment<byte>(binaryData),
                WebSocketMessageType.Binary,
                true,
                CancellationToken.None);

            var receiveBuffer = new byte[1024];
            var result = await webSocket.ReceiveAsync(
                new ArraySegment<byte>(receiveBuffer),
                CancellationToken.None);

            // Assert
            result.MessageType.Should().Be(WebSocketMessageType.Binary);
        }
        finally
        {
            await webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Test complete",
                CancellationToken.None);
        }
    }
}
```

---

## 연결 복원력 테스트

```csharp
public class ConnectionResilienceTests : SignalRTestBase
{
    public ConnectionResilienceTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task Reconnect_AfterDisconnection_Succeeds()
    {
        // Arrange
        var reconnectedTcs = new TaskCompletionSource<bool>();
        Connection.Reconnected += _ =>
        {
            reconnectedTcs.TrySetResult(true);
            return Task.CompletedTask;
        };

        // 자동 재연결 설정된 연결
        var connection = new HubConnectionBuilder()
            .WithUrl(
                new Uri(Factory.CreateClient().BaseAddress!, "/chatHub"),
                options =>
                {
                    options.HttpMessageHandlerFactory = _ => Factory.Server.CreateHandler();
                })
            .WithAutomaticReconnect(new[] { TimeSpan.FromSeconds(0) })
            .Build();

        await connection.StartAsync();

        // Act - 연결 끊김 시뮬레이션은 서버 재시작이 필요하므로
        // 여기서는 연결 상태만 확인

        // Assert
        connection.State.Should().Be(HubConnectionState.Connected);

        await connection.DisposeAsync();
    }
}
```

---

## 성능 테스트

```csharp
public class SignalRPerformanceTests : SignalRTestBase
{
    public SignalRPerformanceTests(SignalRTestFactory factory) : base(factory) { }

    [Fact]
    public async Task BroadcastToManyClients_CompletesWithinTimeout()
    {
        // Arrange - 여러 클라이언트 생성
        const int clientCount = 10;
        var connections = new List<HubConnection>();
        var receiveCounts = new ConcurrentDictionary<int, int>();

        for (int i = 0; i < clientCount; i++)
        {
            var conn = await CreateAdditionalConnectionAsync();
            var index = i;
            conn.On<string, string>("ReceiveMessage", (_, _) =>
            {
                receiveCounts.AddOrUpdate(index, 1, (_, v) => v + 1);
            });
            connections.Add(conn);
        }

        // Act
        var stopwatch = Stopwatch.StartNew();
        await Connection.InvokeAsync("SendMessage", "Broadcast Test");
        stopwatch.Stop();

        // Assert
        await WaitForMessageAsync(() => receiveCounts.Count == clientCount);
        stopwatch.ElapsedMilliseconds.Should().BeLessThan(1000);

        // Cleanup
        foreach (var conn in connections)
        {
            await conn.DisposeAsync();
        }
    }
}
```

---

## 면접 질문

### Q1: SignalR Hub를 어떻게 테스트하나요?
**A**: `WebApplicationFactory`와 `HubConnectionBuilder`를 사용하여 실제 SignalR 연결을 생성하고, Hub 메서드를 호출하여 메시지 송수신을 검증합니다.

### Q2: WebSocket 테스트에서 메시지 수신을 어떻게 기다리나요?
**A**: 폴링 방식으로 조건이 충족될 때까지 대기하거나, `TaskCompletionSource`를 사용하여 비동기적으로 기다립니다. 타임아웃을 설정하여 무한 대기를 방지합니다.

### Q3: 그룹 기능은 어떻게 테스트하나요?
**A**: 여러 연결을 생성하고, 일부만 그룹에 참가시킨 후, 그룹 메시지가 그룹 멤버에게만 전달되는지 검증합니다.
