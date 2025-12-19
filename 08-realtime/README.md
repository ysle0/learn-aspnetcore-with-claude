# 08. Realtime - 실시간 통신

SignalR, gRPC, WebSocket을 이용한 실시간 통신을 이해합니다.

---

## 이 섹션에서 다루는 내용

| 문서 | 설명 | 깊이 |
|------|------|------|
| [signalr.md](./signalr.md) | SignalR 실시간 통신 | 중간 |
| [grpc.md](./grpc.md) | gRPC 서비스 | 중간 |
| [websocket.md](./websocket.md) | WebSocket 기초 | 표면 |

---

## 기술 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                    실시간 통신 기술                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SignalR                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 양방향 실시간 통신                                    │   │
│  │  • 자동 전송 방식 협상 (WebSocket → SSE → Long Polling)  │   │
│  │  • 그룹/사용자 관리                                      │   │
│  │  • Hub 기반 RPC 패턴                                     │   │
│  │  • 용도: 채팅, 알림, 라이브 업데이트                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  gRPC                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • HTTP/2 기반 고성능 RPC                               │   │
│  │  • Protocol Buffers (작은 페이로드)                      │   │
│  │  • 스트리밍 지원 (Unary, Server, Client, Bidirectional) │   │
│  │  • 강력한 타입 계약 (.proto)                             │   │
│  │  • 용도: 마이크로서비스 간 통신, 게임 서버              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  WebSocket                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 저수준 양방향 통신                                    │   │
│  │  • 직접 프로토콜 구현 필요                               │   │
│  │  • 최소 오버헤드                                         │   │
│  │  • 용도: 커스텀 프로토콜, 저지연 요구사항               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 빠른 시작

### SignalR

```csharp
// Hub 정의
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
}

// 등록
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chathub");
```

### gRPC

```protobuf
// greeter.proto
syntax = "proto3";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

```csharp
// 서비스 구현
public class GreeterService : Greeter.GreeterBase
{
    public override Task<HelloReply> SayHello(
        HelloRequest request, ServerCallContext context)
    {
        return Task.FromResult(new HelloReply
        {
            Message = $"Hello {request.Name}"
        });
    }
}

// 등록
builder.Services.AddGrpc();
app.MapGrpcService<GreeterService>();
```

---

## 기능 비교

| 기능 | SignalR | gRPC | WebSocket |
|------|---------|------|-----------|
| 프로토콜 | HTTP/1.1, WS | HTTP/2 | WebSocket |
| 직렬화 | JSON/MessagePack | Protobuf | 커스텀 |
| 브라우저 지원 | ✅ | ❌ (gRPC-Web) | ✅ |
| 스트리밍 | ✅ | ✅ | ✅ |
| 자동 재연결 | ✅ | ❌ | ❌ |
| 그룹 관리 | ✅ | ❌ | ❌ |

---

## 면접 빈출 질문

1. **SignalR의 전송 방식 협상이란?**
2. **gRPC의 4가지 통신 패턴은?**
3. **gRPC-Web이 필요한 이유는?**
4. **SignalR 스케일아웃 방법은?**

---

## 다음 단계

→ [09. Background Services](../09-background-services/) - 백그라운드 작업
