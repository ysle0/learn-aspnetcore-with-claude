# 21. Game Engine Integration - 게임 엔진 연동

Unity3D, Unreal Engine과 ASP.NET Core 서버 연동을 이해합니다.

---

## Unity3D 연동

### SignalR 클라이언트

```csharp
// Unity에서 SignalR 사용
// NuGet: Microsoft.AspNetCore.SignalR.Client
public class GameNetworkManager : MonoBehaviour
{
    private HubConnection _connection;

    async void Start()
    {
        _connection = new HubConnectionBuilder()
            .WithUrl("https://server/gamehub")
            .WithAutomaticReconnect()
            .Build();

        _connection.On<Vector3>("PlayerMoved", OnPlayerMoved);

        await _connection.StartAsync();
    }

    public async Task SendMove(Vector3 position)
    {
        await _connection.InvokeAsync("Move", position);
    }
}
```

### MagicOnion (gRPC)

```csharp
// 공유 인터페이스
public interface IGameHub : IStreamingHub<IGameHub, IGameHubReceiver>
{
    Task JoinAsync(string roomId);
    Task MoveAsync(Vector3 position);
}

public interface IGameHubReceiver
{
    void OnPlayerJoined(string playerId);
    void OnPlayerMoved(string playerId, Vector3 position);
}
```

---

## Unreal Engine 연동

### gRPC (TurboLink)

```cpp
// Unreal에서 gRPC 사용
void AGameClient::SendMove(FVector Position)
{
    MoveRequest Request;
    Request.set_x(Position.X);
    Request.set_y(Position.Y);
    Request.set_z(Position.Z);

    auto* Call = new AsyncMoveCall();
    Call->ResponseReader = Stub->AsyncMove(&Call->Context, Request, &CompletionQueue);
    Call->ResponseReader->Finish(&Call->Reply, &Call->Status, (void*)Call);
}
```

### VaRest (HTTP)

```cpp
// HTTP REST API
void AGameClient::Login(FString Username, FString Password)
{
    TSharedRef<IHttpRequest> Request = Http->CreateRequest();
    Request->SetURL(TEXT("https://server/api/auth/login"));
    Request->SetVerb(TEXT("POST"));
    Request->SetHeader(TEXT("Content-Type"), TEXT("application/json"));
    Request->SetContentAsString(FString::Printf(
        TEXT("{\"username\":\"%s\",\"password\":\"%s\"}"),
        *Username, *Password));
    Request->OnProcessRequestComplete().BindUObject(this, &AGameClient::OnLoginComplete);
    Request->ProcessRequest();
}
```

---

## 다음 단계

→ [22. Reference Repositories](../22-reference-repos/) - 참고 레포지토리
