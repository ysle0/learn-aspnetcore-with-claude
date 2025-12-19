# ASP.NET Core 공부하기

ASP.NET Core의 표면적인 개념부터 프레임워크 내부 동작까지, 면접을 위한 종합적인 학습 자료입니다.

> **Target**: .NET 9 / ASP.NET Core 9 (2025년 12월 기준)

---

## 학습 로드맵

```
[기초] → [서버/파이프라인] → [DI/MVC] → [데이터/캐싱] → [실시간/성능]
                                ↓
[비동기/백그라운드] → [로깅] → [보안] → [최적화] → [배포/게임서버]
```

---

## 목차

### Phase 1: 핵심 기초

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 01 | [Fundamentals](./01-fundamentals/) | ASP.NET Core 탄생 배경, 프로젝트 구조 | ⭐ |
| 02 | [Server Infrastructure](./02-server-infrastructure/) | Kestrel, IIS, 리버스 프록시 | ⭐⭐ |
| 03 | [Request Pipeline](./03-request-pipeline/) | 미들웨어, 라우팅, 필터 | ⭐⭐ |
| 04 | [Dependency Injection](./04-dependency-injection/) | DI 컨테이너, 수명 주기 | ⭐⭐ |

### Phase 2: 실무 필수

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 05 | [MVC and APIs](./05-mvc-and-apis/) | MVC 패턴, Minimal APIs, 모델 바인딩 | ⭐⭐ |
| 06 | [Data Access](./06-data-access/) | EF Core, Dapper, 커넥션 풀링 | ⭐⭐ |
| 07 | [Caching](./07-caching/) | 메모리 캐시, 분산 캐시, HybridCache | ⭐⭐ |
| 10 | [Async Programming](./10-async-programming/) | TAP, async/await 내부 동작 | ⭐⭐⭐ |

### Phase 3: 고급 주제

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 08 | [Real-time Communication](./08-real-time/) | WebSocket, SignalR, gRPC | ⭐⭐⭐ |
| 09 | [Performance](./09-performance/) | Rate Limiting, 응답 압축 | ⭐⭐ |
| 11 | [Background Services](./11-background-services/) | IHostedService, Graceful Shutdown | ⭐⭐ |
| 12 | [Logging](./12-logging/) | 구조화된 로깅, Serilog | ⭐⭐ |

### Phase 4: 보안

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 15 | [Security](./15-security/) | 인증, 권한, JWT, OWASP Top 10 | ⭐⭐⭐ |

### Phase 5: 인프라/배포

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 18 | [Containerization](./18-containerization/) | Docker, Kubernetes, Health Checks | ⭐⭐ |
| 19 | [Cloud - AWS](./19-cloud-aws/) | AWS SDK, ECS/Fargate, Lambda | ⭐⭐⭐ |

### Phase 6: 특화 주제

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 13 | [Source Generators](./13-source-generators/) | 컴파일 타임 코드 생성 | ⭐⭐⭐ |
| 14 | [.NET Aspire](./14-aspire/) | 클라우드 네이티브 오케스트레이션 | ⭐⭐ |
| 16 | [Extreme Optimization](./16-extreme-optimization/) | Span, Memory, unsafe, SIMD | ⭐⭐⭐⭐ |
| 17 | [Reflection Alternatives](./17-reflection-alternatives/) | Expression Trees, IL Emit | ⭐⭐⭐⭐ |

### Phase 7: 게임 서버

| # | 섹션 | 설명 | 난이도 |
|---|------|------|--------|
| 20 | [Game Server Development](./20-game-server-development/) | 아키텍처, Orleans, 네트워킹 | ⭐⭐⭐ |
| 21 | [Game Engine Integration](./21-game-engine-integration/) | Unity, Unreal Engine 연동 | ⭐⭐⭐ |
| 22 | [Game Server References](./22-game-server-references/) | 오픈소스 레포지토리 분석 | ⭐⭐ |

---

## 빠른 참조

### 핵심 면접 질문 Top 10

1. **Kestrel과 IIS의 차이점은?** → [02-server-infrastructure](./02-server-infrastructure/)
2. **미들웨어 파이프라인은 어떻게 동작하나요?** → [03-request-pipeline](./03-request-pipeline/)
3. **DI의 Scoped, Transient, Singleton 차이는?** → [04-dependency-injection](./04-dependency-injection/)
4. **async/await 내부적으로 어떻게 동작하나요?** → [10-async-programming](./10-async-programming/)
5. **JWT 토큰 인증은 어떻게 구현하나요?** → [15-security](./15-security/)
6. **EF Core vs Dapper 성능 차이는?** → [06-data-access](./06-data-access/)
7. **SignalR vs gRPC 언제 사용하나요?** → [08-real-time](./08-real-time/)
8. **HybridCache란 무엇인가요?** → [07-caching](./07-caching/)
9. **Graceful Shutdown은 어떻게 구현하나요?** → [11-background-services](./11-background-services/)
10. **Span<T>과 Memory<T>의 차이는?** → [16-extreme-optimization](./16-extreme-optimization/)

### 예시 코드

모든 개념에는 실행 가능한 예시 코드가 포함되어 있습니다:

```
examples/
├── 01-basic-api/              # 기본 Web API
├── 02-middleware-demo/        # 커스텀 미들웨어
├── 03-di-demo/                # DI 패턴들
├── 04-caching-demo/           # 캐싱 전략
├── 05-signalr-demo/           # 실시간 통신
├── 06-grpc-demo/              # gRPC 서비스
├── 07-background-service-demo/ # 백그라운드 작업
├── 08-source-generator-demo/  # 소스 제너레이터
├── 09-security-demo/          # JWT 인증
├── 10-span-memory-demo/       # 고성능 메모리
├── 11-expression-tree-demo/   # 표현식 트리
├── 12-docker-k8s-demo/        # 컨테이너 배포
├── 13-aws-integration-demo/   # AWS 통합
└── 14-game-server-demo/       # 게임 서버
```

---

## 문서 구조

각 문서는 일관된 구조를 따릅니다:

```markdown
# 제목

## 개요
간결한 1-2문단 설명

## 핵심 개념
- 핵심 포인트들

## 예시 코드
// 실행 가능한 코드

## 깊이 있는 설명
→ [상세 문서 링크](./internals.md)

## 면접 예상 질문
- Q1: ...
- Q2: ...

## 참고 자료
- 공식 문서, 블로그 링크
```

### 깊이 단계

- **표면**: 면접 기초 질문 대비
- **중간**: 실무 적용 수준
- **깊음**: 프레임워크 내부 이해
- **매우 깊음**: 소스 코드 수준 이해

---

## 참고 자료

### 공식 문서
- [Microsoft Learn - ASP.NET Core](https://learn.microsoft.com/aspnet/core)
- [.NET Blog](https://devblogs.microsoft.com/dotnet/)
- [GitHub - dotnet/aspnetcore](https://github.com/dotnet/aspnetcore)

### 추천 블로그
- [Steve Gordon's Blog](https://www.stevejgordon.co.uk/)
- [Andrew Lock's Blog](https://andrewlock.net/)
- [Milan Jovanović's Blog](https://www.milanjovanovic.tech/)

### 면접 준비
- [GitHub - net-core-interview-questions](https://github.com/Devinterview-io/net-core-interview-questions)

---

## 기여

오류 수정이나 내용 추가는 Issue 또는 PR로 제출해 주세요.

---

*이 레포지토리는 Claude Code와 함께 작성되었습니다.*
