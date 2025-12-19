# ASP.NET Core 면접 준비 레포지토리 계획서

## 📋 개요

이 레포지토리는 ASP.NET Core 면접을 위한 종합적인 학습 자료를 제공합니다.
표면적인 개념부터 프레임워크 내부 동작까지, 단계별로 깊이 있는 설명과 예시 코드를 포함합니다.

---

## 🏗️ 레포지토리 구조

```
learn-aspnetcore-with-claude/
├── PLAN.md                          # 이 문서
├── README.md                        # 메인 진입점 (개요 + 목차)
│
├── 01-fundamentals/                 # 기초 개념
│   ├── README.md                    # 섹션 개요
│   ├── why-aspnetcore.md           # ASP.NET Core를 왜 이렇게 만들었는가
│   ├── project-structure.md        # 프로젝트 구조 (csproj, slnx)
│   └── hosting-models.md           # 호스팅 모델
│
├── 02-server-infrastructure/        # 서버 인프라
│   ├── README.md
│   ├── kestrel/
│   │   ├── overview.md             # Kestrel 개요
│   │   └── internals.md            # Kestrel 내부 동작
│   ├── iis-integration.md          # IIS 연동
│   └── reverse-proxy.md            # 리버스 프록시 패턴
│
├── 03-request-pipeline/             # 요청 파이프라인
│   ├── README.md
│   ├── middleware/
│   │   ├── overview.md             # 미들웨어 개요
│   │   └── internals.md            # 파이프라인 내부 구조
│   ├── routing.md                  # 라우팅
│   └── filters.md                  # MVC 필터
│
├── 04-dependency-injection/         # 의존성 주입
│   ├── README.md
│   ├── overview.md                 # DI 개요
│   ├── lifetimes.md                # 서비스 수명 주기
│   ├── internals.md                # DI 컨테이너 내부
│   └── advanced-patterns.md        # 고급 패턴
│
├── 05-mvc-and-apis/                 # MVC와 API
│   ├── README.md
│   ├── mvc-pattern.md              # MVC 패턴
│   ├── minimal-apis.md             # Minimal APIs
│   ├── model-binding.md            # 모델 바인딩
│   ├── input-validation.md         # 입력 검증
│   └── attributes.md               # ASP.NET Core 속성들
│
├── 06-data-access/                  # 데이터 접근
│   ├── README.md
│   ├── ef-core/
│   │   ├── overview.md             # EF Core 개요
│   │   └── advanced.md             # 고급 기능
│   ├── dapper.md                   # Dapper
│   ├── sqlkata.md                  # SqlKata
│   └── connection-pooling.md       # 커넥션 풀링
│
├── 07-caching/                      # 캐싱
│   ├── README.md
│   ├── in-memory-cache.md          # 메모리 캐시
│   ├── distributed-cache.md        # 분산 캐시
│   └── hybrid-cache.md             # HybridCache (.NET 9)
│
├── 08-real-time/                    # 실시간 통신
│   ├── README.md
│   ├── http-fundamentals.md        # HTTP 기초
│   ├── websocket.md                # WebSocket
│   ├── signalr/
│   │   ├── overview.md             # SignalR 개요
│   │   └── internals.md            # SignalR 내부
│   └── grpc/
│       ├── overview.md             # gRPC 개요
│       └── streaming.md            # gRPC 스트리밍
│
├── 09-performance/                  # 성능
│   ├── README.md
│   ├── rate-limiting.md            # Rate Limiting
│   ├── response-compression.md     # 응답 압축
│   └── benchmarking.md             # 벤치마킹
│
├── 10-async-programming/            # 비동기 프로그래밍
│   ├── README.md
│   ├── tap-overview.md             # TAP 개요
│   ├── tap-internals.md            # TAP 내부 (상태 머신)
│   ├── thread-pool.md              # 스레드 풀
│   └── best-practices.md           # 모범 사례
│
├── 11-background-services/          # 백그라운드 서비스
│   ├── README.md
│   ├── hosted-services.md          # IHostedService
│   ├── background-service.md       # BackgroundService
│   └── graceful-shutdown.md        # Graceful Shutdown
│
├── 12-logging/                      # 로깅
│   ├── README.md
│   ├── logging-fundamentals.md     # 로깅 기초
│   ├── structured-logging.md       # 구조화된 로깅
│   └── serilog-integration.md      # Serilog 통합
│
├── 13-source-generators/            # 소스 제너레이터
│   ├── README.md
│   ├── overview.md                 # 개요
│   ├── implementation.md           # 구현 방법
│   └── aspnetcore-generators.md    # ASP.NET Core의 소스 제너레이터들
│
├── 14-aspire/                       # .NET Aspire
│   ├── README.md
│   ├── overview.md                 # 개요
│   ├── orchestration.md            # 오케스트레이션
│   └── integrations.md             # 통합
│
├── 15-security/                     # 보안
│   ├── README.md
│   ├── authentication/
│   │   ├── overview.md             # 인증 개요
│   │   ├── jwt-bearer.md           # JWT Bearer 인증
│   │   ├── oauth-oidc.md           # OAuth 2.0 / OpenID Connect
│   │   └── identity.md             # ASP.NET Core Identity
│   ├── authorization/
│   │   ├── overview.md             # 권한 부여 개요
│   │   ├── policy-based.md         # 정책 기반 권한 부여
│   │   └── resource-based.md       # 리소스 기반 권한 부여
│   ├── data-protection.md          # Data Protection API
│   ├── https-ssl.md                # HTTPS/SSL 설정
│   ├── cors.md                     # CORS 정책
│   ├── csrf-protection.md          # CSRF 방어
│   ├── security-headers.md         # 보안 헤더
│   └── owasp-top10.md              # OWASP Top 10 대응
│
├── 16-extreme-optimization/         # 극한 최적화
│   ├── README.md
│   ├── span-memory/
│   │   ├── overview.md             # Span<T>/Memory<T> 개요
│   │   ├── internals.md            # 내부 동작 원리
│   │   └── patterns.md             # 활용 패턴
│   ├── low-allocation/
│   │   ├── stackalloc.md           # stackalloc 사용법
│   │   ├── array-pooling.md        # ArrayPool<T>
│   │   └── object-pooling.md       # ObjectPool<T>
│   ├── unsafe-code/
│   │   ├── pointers.md             # 포인터와 unsafe 코드
│   │   └── unsafe-class.md         # System.Runtime.CompilerServices.Unsafe
│   ├── simd-vectorization.md       # SIMD와 Vector<T>
│   └── native-aot.md               # Native AOT 컴파일
│
├── 17-reflection-alternatives/      # 리플렉션과 대안
│   ├── README.md
│   ├── reflection/
│   │   ├── overview.md             # 리플렉션 기초
│   │   ├── performance.md          # 리플렉션 성능 문제
│   │   └── caching.md              # 리플렉션 캐싱 기법
│   ├── expression-trees/
│   │   ├── overview.md             # Expression Trees 개요
│   │   ├── compilation.md          # 컴파일된 표현식
│   │   └── fast-member.md          # FastMember 등 라이브러리
│   ├── source-generators-alt.md    # 소스 제너레이터로 대체
│   └── emit-dynamic.md             # IL Emit과 동적 코드 생성
│
├── 18-containerization/             # 컨테이너화
│   ├── README.md
│   ├── docker/
│   │   ├── dockerfile-basics.md    # Dockerfile 기초
│   │   ├── multi-stage-build.md    # 멀티스테이지 빌드
│   │   └── optimization.md         # 이미지 최적화
│   ├── kubernetes/
│   │   ├── overview.md             # Kubernetes 개요
│   │   ├── deployment.md           # Deployment, Service, Ingress
│   │   └── health-probes.md        # Liveness/Readiness/Startup Probes
│   └── health-checks.md            # ASP.NET Core Health Checks
│
├── 19-cloud-aws/                    # AWS 배포
│   ├── README.md
│   ├── sdk-setup.md                # AWS SDK for .NET 설정
│   ├── services/
│   │   ├── s3.md                   # Amazon S3
│   │   ├── sqs-sns.md              # SQS, SNS 메시징
│   │   ├── dynamodb.md             # DynamoDB
│   │   └── secrets-manager.md      # Secrets Manager
│   ├── deployment/
│   │   ├── ecs-fargate.md          # ECS + Fargate
│   │   ├── lambda.md               # AWS Lambda
│   │   └── elastic-beanstalk.md    # Elastic Beanstalk
│   └── aspire-aws.md               # .NET Aspire + AWS 통합
│
├── 20-game-server-development/      # 게임 서버 개발
│   ├── README.md
│   ├── architecture/
│   │   ├── patterns.md             # 게임 서버 아키텍처 패턴
│   │   ├── state-management.md     # 상태 관리
│   │   └── scalability.md          # 확장성 설계
│   ├── networking/
│   │   ├── protocols.md            # TCP vs UDP vs WebSocket
│   │   ├── reliable-udp.md         # Reliable UDP 구현
│   │   └── serialization.md        # 직렬화 (MessagePack, Protobuf)
│   ├── orleans.md                  # Microsoft Orleans
│   ├── magiconion.md               # MagicOnion 프레임워크
│   └── networking-libs.md          # NetCoreServer, LiteNetLib, DotNetty
│
├── 21-game-engine-integration/      # 게임 엔진 연동
│   ├── README.md
│   ├── unity/
│   │   ├── rest-api.md             # Unity + REST API
│   │   ├── signalr.md              # Unity + SignalR
│   │   └── grpc.md                 # Unity + gRPC (MagicOnion)
│   ├── unreal/
│   │   ├── http-requests.md        # Unreal + HTTP
│   │   ├── grpc.md                 # Unreal + gRPC (TurboLink)
│   │   └── varest.md               # VaRest 플러그인
│   └── common/
│       ├── authentication.md       # 게임 클라이언트 인증
│       ├── matchmaking.md          # 매치메이킹 서버
│       └── leaderboard.md          # 리더보드/랭킹 시스템
│
├── 22-game-server-references/       # 게임 서버 레퍼런스
│   ├── README.md                   # 개요 및 추천 레포지토리
│   ├── open-source-servers.md      # 오픈소스 게임 서버들
│   └── case-studies.md             # 사례 연구
│
└── examples/                        # 예시 코드
    ├── 01-basic-api/
    ├── 02-middleware-demo/
    ├── 03-di-demo/
    ├── 04-caching-demo/
    ├── 05-signalr-demo/
    ├── 06-grpc-demo/
    ├── 07-background-service-demo/
    ├── 08-source-generator-demo/
    ├── 09-security-demo/
    ├── 10-span-memory-demo/
    ├── 11-expression-tree-demo/
    ├── 12-docker-k8s-demo/
    ├── 13-aws-integration-demo/
    └── 14-game-server-demo/
```

---

## 📚 섹션별 상세 계획

### 1. Fundamentals (기초)

| 문서 | 내용 | 깊이 |
|------|------|------|
| why-aspnetcore.md | ASP.NET Core 탄생 배경, 설계 철학 | 표면 → 역사적 맥락 |
| project-structure.md | csproj, slnx, SDK 스타일 프로젝트 | 표면 → MSBuild 내부 |
| hosting-models.md | In-process, Out-of-process | 표면 → 프로세스 모델 |

### 2. Server Infrastructure (서버 인프라)

| 문서 | 내용 | 깊이 |
|------|------|------|
| kestrel/overview.md | Kestrel 소개, 설정 | 표면 |
| kestrel/internals.md | I/O 모델, 소켓 처리, HTTP 파싱 | 깊은 내부 |
| iis-integration.md | ASP.NET Core Module | 중간 |
| reverse-proxy.md | Nginx, YARP | 실용 |

### 3. Request Pipeline (요청 파이프라인)

| 문서 | 내용 | 깊이 |
|------|------|------|
| middleware/overview.md | 미들웨어 개념, Use/Run/Map | 표면 |
| middleware/internals.md | RequestDelegate 체인, 빌드 과정 | 깊은 내부 |
| routing.md | Endpoint Routing | 중간 → 깊음 |
| filters.md | Action/Result/Exception 필터 | 중간 |

### 4. Dependency Injection (의존성 주입)

| 문서 | 내용 | 깊이 |
|------|------|------|
| overview.md | DI 기본 개념, 등록 방법 | 표면 |
| lifetimes.md | Singleton/Scoped/Transient | 중간 |
| internals.md | ServiceProvider 내부 구조 | 깊은 내부 |
| advanced-patterns.md | Factory, Decorator, 다중 구현 | 고급 |

### 5. MVC and APIs

| 문서 | 내용 | 깊이 |
|------|------|------|
| mvc-pattern.md | MVC 아키텍처 | 표면 |
| minimal-apis.md | Minimal APIs | 표면 → 중간 |
| model-binding.md | 모델 바인딩 과정 | 중간 → 깊음 |
| input-validation.md | DataAnnotations, FluentValidation | 중간 |
| attributes.md | [ApiController], [FromBody] 등 | 표면 → 내부 |

### 6. Data Access (데이터 접근)

| 문서 | 내용 | 깊이 |
|------|------|------|
| ef-core/overview.md | EF Core 기본 | 표면 |
| ef-core/advanced.md | Change Tracking, 성능 최적화 | 깊음 |
| dapper.md | Dapper 사용법, 성능 | 중간 |
| sqlkata.md | SqlKata 쿼리 빌더 | 중간 |
| connection-pooling.md | ADO.NET 커넥션 풀링 | 중간 → 깊음 |

### 7. Caching (캐싱)

| 문서 | 내용 | 깊이 |
|------|------|------|
| in-memory-cache.md | IMemoryCache | 표면 |
| distributed-cache.md | IDistributedCache, Redis | 중간 |
| hybrid-cache.md | HybridCache (L1+L2) | 중간 → 깊음 |

### 8. Real-time Communication (실시간 통신)

| 문서 | 내용 | 깊이 |
|------|------|------|
| http-fundamentals.md | HTTP/1.1, HTTP/2, HTTP/3 | 표면 → 중간 |
| websocket.md | WebSocket 프로토콜 | 중간 |
| signalr/overview.md | SignalR 개요 | 표면 |
| signalr/internals.md | Hub 프로토콜, 전송 협상 | 깊음 |
| grpc/overview.md | gRPC 개요, Protobuf | 표면 |
| grpc/streaming.md | 스트리밍 패턴 | 중간 |

### 9. Performance (성능)

| 문서 | 내용 | 깊이 |
|------|------|------|
| rate-limiting.md | 내장 Rate Limiting | 표면 → 중간 |
| response-compression.md | 응답 압축 | 표면 |
| benchmarking.md | BenchmarkDotNet, 성능 측정 | 중간 |

### 10. Async Programming (비동기 프로그래밍)

| 문서 | 내용 | 깊이 |
|------|------|------|
| tap-overview.md | async/await 기본 | 표면 |
| tap-internals.md | 상태 머신, Awaiter | 매우 깊음 |
| thread-pool.md | ThreadPool 동작 원리 | 깊음 |
| best-practices.md | ConfigureAwait, ValueTask | 실용 |

### 11. Background Services (백그라운드 서비스)

| 문서 | 내용 | 깊이 |
|------|------|------|
| hosted-services.md | IHostedService | 표면 |
| background-service.md | BackgroundService | 중간 |
| graceful-shutdown.md | Graceful Shutdown 구현 | 중간 → 깊음 |

### 12. Logging (로깅)

| 문서 | 내용 | 깊이 |
|------|------|------|
| logging-fundamentals.md | ILogger, LogLevel | 표면 |
| structured-logging.md | 구조화된 로깅 개념 | 중간 |
| serilog-integration.md | Serilog 설정 및 싱크 | 실용 |

### 13. Source Generators (소스 제너레이터)

| 문서 | 내용 | 깊이 |
|------|------|------|
| overview.md | 소스 제너레이터 개념 | 표면 |
| implementation.md | IIncrementalGenerator 구현 | 중간 → 깊음 |
| aspnetcore-generators.md | ASP.NET Core가 사용하는 제너레이터들 | 중간 |

### 14. .NET Aspire

| 문서 | 내용 | 깊이 |
|------|------|------|
| overview.md | Aspire 개요 | 표면 |
| orchestration.md | AppHost, 서비스 오케스트레이션 | 중간 |
| integrations.md | Redis, PostgreSQL 등 통합 | 실용 |

### 15. Security (보안)

| 문서 | 내용 | 깊이 |
|------|------|------|
| authentication/overview.md | 인증 시스템 개요 | 표면 |
| authentication/jwt-bearer.md | JWT 토큰 인증, 검증 | 중간 → 깊음 |
| authentication/oauth-oidc.md | OAuth 2.0, OpenID Connect | 중간 |
| authentication/identity.md | ASP.NET Core Identity | 중간 |
| authorization/overview.md | 권한 부여 개요 | 표면 |
| authorization/policy-based.md | 정책 기반 권한 부여 | 중간 |
| authorization/resource-based.md | 리소스 기반 권한 부여 | 중간 → 깊음 |
| data-protection.md | Data Protection API, 키 관리 | 중간 → 깊음 |
| https-ssl.md | HTTPS 설정, 인증서 | 표면 → 중간 |
| cors.md | CORS 정책 설정 | 표면 → 중간 |
| csrf-protection.md | CSRF 방어, Anti-Forgery Token | 중간 |
| security-headers.md | CSP, HSTS 등 보안 헤더 | 중간 |
| owasp-top10.md | OWASP Top 10 취약점 대응 | 실용 |

### 16. Extreme Optimization (극한 최적화)

| 문서 | 내용 | 깊이 |
|------|------|------|
| span-memory/overview.md | Span<T>, Memory<T> 개요 | 표면 → 중간 |
| span-memory/internals.md | ref struct, 스택 할당 | 깊음 |
| span-memory/patterns.md | 문자열 파싱, 버퍼 처리 패턴 | 실용 |
| low-allocation/stackalloc.md | stackalloc 사용법 | 중간 |
| low-allocation/array-pooling.md | ArrayPool<T> 활용 | 중간 |
| low-allocation/object-pooling.md | ObjectPool<T> 활용 | 중간 |
| unsafe-code/pointers.md | 포인터 기초, fixed 문 | 중간 → 깊음 |
| unsafe-code/unsafe-class.md | Unsafe 클래스 활용 | 깊음 |
| simd-vectorization.md | Vector<T>, SIMD 최적화 | 깊음 |
| native-aot.md | Native AOT, 트리밍 | 중간 → 깊음 |

### 17. Reflection Alternatives (리플렉션과 대안)

| 문서 | 내용 | 깊이 |
|------|------|------|
| reflection/overview.md | 리플렉션 기초, Type/MethodInfo | 표면 |
| reflection/performance.md | 리플렉션 성능 벤치마크 | 중간 |
| reflection/caching.md | Delegate.CreateDelegate 캐싱 | 중간 → 깊음 |
| expression-trees/overview.md | Expression Trees 기초 | 중간 |
| expression-trees/compilation.md | Compile(), 성능 비교 | 깊음 |
| expression-trees/fast-member.md | FastMember, 고성능 라이브러리 | 실용 |
| source-generators-alt.md | 리플렉션 → 소스 제너레이터 마이그레이션 | 중간 |
| emit-dynamic.md | IL Emit, DynamicMethod | 매우 깊음 |

### 18. Containerization (컨테이너화)

| 문서 | 내용 | 깊이 |
|------|------|------|
| docker/dockerfile-basics.md | Dockerfile 기초, 명령어 | 표면 |
| docker/multi-stage-build.md | 멀티스테이지 빌드 최적화 | 중간 |
| docker/optimization.md | 이미지 크기 최적화, 보안 | 중간 → 깊음 |
| kubernetes/overview.md | Kubernetes 기본 개념 | 표면 |
| kubernetes/deployment.md | Deployment, Service, Ingress | 중간 |
| kubernetes/health-probes.md | Liveness/Readiness/Startup Probes | 중간 |
| health-checks.md | ASP.NET Core Health Checks API | 중간 |

### 19. Cloud - AWS (AWS 클라우드)

| 문서 | 내용 | 깊이 |
|------|------|------|
| sdk-setup.md | AWS SDK for .NET 설정, DI 통합 | 표면 → 중간 |
| services/s3.md | S3 파일 업로드/다운로드 | 중간 |
| services/sqs-sns.md | 메시지 큐, 푸시 알림 | 중간 |
| services/dynamodb.md | DynamoDB CRUD, 쿼리 | 중간 |
| services/secrets-manager.md | 시크릿 관리 | 중간 |
| deployment/ecs-fargate.md | ECS + Fargate 배포 | 중간 → 깊음 |
| deployment/lambda.md | AWS Lambda + API Gateway | 중간 |
| deployment/elastic-beanstalk.md | Elastic Beanstalk | 표면 → 중간 |
| aspire-aws.md | .NET Aspire + AWS 통합 | 중간 |

### 20. Game Server Development (게임 서버 개발)

| 문서 | 내용 | 깊이 |
|------|------|------|
| architecture/patterns.md | 게임 서버 아키텍처 패턴 | 중간 → 깊음 |
| architecture/state-management.md | 게임 상태 관리 | 중간 |
| architecture/scalability.md | 수평/수직 확장 전략 | 중간 → 깊음 |
| networking/protocols.md | TCP vs UDP vs WebSocket | 중간 |
| networking/reliable-udp.md | Reliable UDP 구현 원리 | 깊음 |
| networking/serialization.md | MessagePack, Protobuf 직렬화 | 중간 |
| orleans.md | Microsoft Orleans Virtual Actor | 깊음 |
| magiconion.md | MagicOnion 실시간 통신 | 중간 → 깊음 |
| networking-libs.md | NetCoreServer, LiteNetLib, DotNetty | 중간 |

### 21. Game Engine Integration (게임 엔진 연동)

| 문서 | 내용 | 깊이 |
|------|------|------|
| unity/rest-api.md | Unity + UnityWebRequest | 표면 → 중간 |
| unity/signalr.md | Unity + SignalR Client | 중간 |
| unity/grpc.md | Unity + gRPC/MagicOnion | 중간 → 깊음 |
| unreal/http-requests.md | Unreal + HTTP Module | 표면 → 중간 |
| unreal/grpc.md | Unreal + gRPC (TurboLink) | 중간 |
| unreal/varest.md | VaRest REST API 플러그인 | 중간 |
| common/authentication.md | 게임 클라이언트 인증 패턴 | 중간 |
| common/matchmaking.md | 매치메이킹 서버 구현 | 중간 → 깊음 |
| common/leaderboard.md | 리더보드/랭킹 시스템 | 중간 |

### 22. Game Server References (게임 서버 레퍼런스)

| 문서 | 내용 | 깊이 |
|------|------|------|
| open-source-servers.md | 추천 오픈소스 게임 서버 | 참고 |
| case-studies.md | 실제 게임 서버 사례 연구 | 참고 |

---

## 🎮 추천 오픈소스 게임 서버 레포지토리

### 아키텍처 학습용

| 레포지토리 | 설명 | 특징 |
|------------|------|------|
| [dotnet/orleans](https://github.com/dotnet/orleans) | Microsoft Virtual Actor Framework | Halo, Gears of War 백엔드에 사용 |
| [heroiclabs/nakama](https://github.com/heroiclabs/nakama) | 오픈소스 게임 서버 | 2M CCU 지원, 완성도 높음 |
| [OpenCoreMMO](https://github.com/OpenCoreMMO/OpenCoreMMO) | C# MMORPG 서버 에뮬레이터 | .NET 10, 현대적 구조 |
| [NoCode-NoLife/Melia](https://github.com/NoCode-NoLife/Melia) | MMORPG 서버 에뮬레이터 | .NET 8+, MySQL |

### 네트워킹 컴포넌트 학습용

| 레포지토리 | 설명 | 특징 |
|------------|------|------|
| [chronoxor/NetCoreServer](https://github.com/chronoxor/NetCoreServer) | 고성능 소켓 서버 | TCP/UDP/WebSocket, 10K 문제 해결 |
| [RevenantX/LiteNetLib](https://github.com/RevenantX/LiteNetLib) | 경량 UDP 라이브러리 | 게임용 Reliable UDP, Unity 지원 |
| [Azure/DotNetty](https://github.com/Azure/DotNetty) | Netty의 .NET 포팅 | 이벤트 기반 네트워크 프레임워크 |
| [microsoft/reverse-proxy (YARP)](https://github.com/microsoft/reverse-proxy) | ASP.NET Core 리버스 프록시 | Kestrel 기반 프록시 구현 학습 |

### 콘텐츠 및 게임 로직 학습용

| 레포지토리 | 설명 | 특징 |
|------------|------|------|
| [TrinityCore](https://github.com/TrinityCore/TrinityCore) | WoW 서버 에뮬레이터 | C++이지만 아키텍처 참고용 |
| [egametang/ET](https://github.com/egametang/ET) | C# 게임 프레임워크 | .NET 8 + Unity, 행동 트리, 버프 시스템 |

---

## 📝 문서 작성 원칙

### 1. 구조 일관성
각 문서는 다음 구조를 따릅니다:
```markdown
# 제목

## 개요
간결한 1-2문단 설명

## 핵심 개념
- 핵심1
- 핵심2

## 예시 코드
```csharp
// 실행 가능한 예시
```

## 깊이 있는 설명
→ [상세 문서 링크](./internals.md)

## 면접 예상 질문
- Q1: ...
- Q2: ...

## 참고 자료
- [공식 문서](...)
- [관련 아티클](...)
```

### 2. 코드 예시 원칙
- 모든 코드는 복사해서 바로 실행 가능해야 함
- 주석으로 핵심 포인트 설명
- 잘못된 예시와 올바른 예시 대비

### 3. 깊이 단계
- **표면**: 면접 기초 질문 대비
- **중간**: 실무 적용 수준
- **깊음**: 프레임워크 내부 이해
- **매우 깊음**: 소스 코드 수준 이해

---

## 🚀 작업 순서

### Phase 1: 핵심 기초 (우선순위 높음)
1. README.md (메인)
2. 01-fundamentals
3. 02-server-infrastructure
4. 03-request-pipeline
5. 04-dependency-injection

### Phase 2: 실무 필수
6. 05-mvc-and-apis
7. 06-data-access
8. 07-caching
9. 10-async-programming

### Phase 3: 고급 주제
10. 08-real-time
11. 09-performance
12. 11-background-services
13. 12-logging

### Phase 4: 보안 (중요)
14. 15-security (인증, 권한, OWASP)

### Phase 5: 인프라/배포
15. 18-containerization (Docker, Kubernetes)
16. 19-cloud-aws (AWS 서비스 및 배포)

### Phase 6: 특화 주제
17. 13-source-generators
18. 14-aspire
19. 16-extreme-optimization
20. 17-reflection-alternatives

### Phase 7: 게임 서버 (실시간성 낮은 게임)
21. 20-game-server-development
22. 21-game-engine-integration
23. 22-game-server-references

### Phase 8: 예시 코드
24. examples/ 디렉토리 구현

---

## 📖 참고 자료 출처

### 공식 문서
- [Microsoft Learn - ASP.NET Core](https://learn.microsoft.com/aspnet/core)
- [.NET Blog](https://devblogs.microsoft.com/dotnet/)
- [GitHub - dotnet/aspnetcore](https://github.com/dotnet/aspnetcore)

### 커뮤니티 자료
- [Steve Gordon's Blog](https://www.stevejgordon.co.uk/)
- [Andrew Lock's Blog](https://andrewlock.net/)
- [Milan Jovanović's Blog](https://www.milanjovanovic.tech/)
- [Code with Mukesh](https://codewithmukesh.com/)

### 보안 자료
- [OWASP .NET Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DotNet_Security_Cheat_Sheet.html)
- [Microsoft Learn - ASP.NET Core Security](https://learn.microsoft.com/aspnet/core/security)
- [JWT Validation and Authorization in ASP.NET Core](https://devblogs.microsoft.com/dotnet/jwt-validation-and-authorization-in-asp-net-core/)

### 고성능/최적화 자료
- [Adam Sitnik - Span](https://adamsitnik.com/Span/)
- [Writing High-Performance Code Using Span<T> and Memory<T>](https://www.codemag.com/Article/2207031/Writing-High-Performance-Code-Using-SpanT-and-MemoryT-in-C)
- [Source Generators Cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md)
- [FastExpressionCompiler](https://github.com/dadhi/FastExpressionCompiler)

### 컨테이너/클라우드 자료
- [Docker - 9 Tips for Containerizing Your .NET Application](https://www.docker.com/blog/9-tips-for-containerizing-your-net-application/)
- [Andrew Lock - Deploying to Kubernetes](https://andrewlock.net/deploying-asp-net-core-applications-to-kubernetes-part-6-adding-health-checks-with-liveness-readiness-and-startup-probes/)
- [AWS SDK for .NET Developer Guide](https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/)
- [AWS - Hosting ASP.NET Core in Amazon ECS](https://aws.amazon.com/blogs/compute/hosting-asp-net-core-applications-in-amazon-ecs-using-aws-fargate/)

### 게임 서버 자료
- [Cysharp/MagicOnion](https://github.com/Cysharp/MagicOnion) - Unity/gRPC 통합
- [thejinchao/TurboLink](https://github.com/thejinchao/turbolink) - Unreal Engine gRPC
- [ufna/VaRest](https://github.com/ufna/VaRest) - Unreal Engine REST API
- [Microsoft Orleans Documentation](https://learn.microsoft.com/dotnet/orleans/)
- [.NET Game Services](https://dotnet.microsoft.com/apps/games/services)

### 면접 준비 자료
- [GitHub - Devinterview-io/net-core-interview-questions](https://github.com/Devinterview-io/net-core-interview-questions)
- [ScholarHat - ASP.NET Core Interview Questions](https://www.scholarhat.com/tutorial/aspnet/asp-net-core-interview-questions)

---

## ✅ 체크리스트

- [ ] README.md 작성
- [ ] 01-fundamentals 완료
- [ ] 02-server-infrastructure 완료
- [ ] 03-request-pipeline 완료
- [ ] 04-dependency-injection 완료
- [ ] 05-mvc-and-apis 완료
- [ ] 06-data-access 완료
- [ ] 07-caching 완료
- [ ] 08-real-time 완료
- [ ] 09-performance 완료
- [ ] 10-async-programming 완료
- [ ] 11-background-services 완료
- [ ] 12-logging 완료
- [ ] 13-source-generators 완료
- [ ] 14-aspire 완료
- [ ] 15-security 완료
- [ ] 16-extreme-optimization 완료
- [ ] 17-reflection-alternatives 완료
- [ ] 18-containerization 완료
- [ ] 19-cloud-aws 완료
- [ ] 20-game-server-development 완료
- [ ] 21-game-engine-integration 완료
- [ ] 22-game-server-references 완료
- [ ] examples/ 완료

---

*이 계획서는 2025년 12월 기준 최신 .NET 9 / ASP.NET Core 9 를 기반으로 작성되었습니다.*
