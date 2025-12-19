# 20. Game Server - 게임 서버

ASP.NET Core 기반 게임 서버 개발을 이해합니다.

---

## 관련 기술

| 기술 | 설명 | 용도 |
|------|------|------|
| Orleans | Virtual Actor 모델 | 분산 게임 상태 |
| MagicOnion | gRPC + MessagePack | 실시간 통신 |
| SignalR | 웹소켓 | 웹 기반 게임 |
| NetCoreServer | 고성능 소켓 | 저지연 통신 |
| LiteNetLib | UDP | 액션 게임 |

---

## Orleans 예시

```csharp
// Grain 인터페이스
public interface IPlayerGrain : IGrainWithStringKey
{
    Task<PlayerState> GetStateAsync();
    Task JoinGameAsync(string gameId);
    Task MoveAsync(Position position);
}

// Grain 구현
public class PlayerGrain : Grain, IPlayerGrain
{
    private readonly IPersistentState<PlayerState> _state;

    public PlayerGrain(
        [PersistentState("player")] IPersistentState<PlayerState> state)
    {
        _state = state;
    }

    public Task<PlayerState> GetStateAsync() => Task.FromResult(_state.State);

    public async Task JoinGameAsync(string gameId)
    {
        _state.State.CurrentGameId = gameId;
        await _state.WriteStateAsync();
    }
}
```

---

## 다음 단계

→ [21. Game Engine Integration](../21-game-engine-integration/) - 게임 엔진 연동
