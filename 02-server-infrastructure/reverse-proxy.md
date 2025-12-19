# 리버스 프록시 패턴

## 개요

리버스 프록시는 클라이언트와 백엔드 서버 사이에서 요청을 중계하는 서버입니다. ASP.NET Core 앱을 프로덕션에 배포할 때 Nginx, Apache, IIS 등의 리버스 프록시를 앞에 두는 것이 일반적입니다.

---

## 왜 리버스 프록시를 사용하나요?

```
┌─────────────────────────────────────────────────────────────────┐
│                    리버스 프록시의 이점                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🛡️ 보안           │  DDoS 방어, WAF, SSL 종료                  │
│  ⚖️ 로드 밸런싱    │  여러 백엔드 서버로 분산                   │
│  📦 캐싱           │  정적 콘텐츠 캐싱                          │
│  🗜️ 압축           │  gzip/Brotli 응답 압축                     │
│  🔄 URL 재작성     │  경로/호스트 변환                          │
│  📊 요청 제한      │  Rate Limiting                             │
│  🔍 로깅/모니터링  │  접근 로그, 메트릭 수집                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기능 비교

| 기능 | Kestrel 단독 | + 리버스 프록시 |
|------|-------------|-----------------|
| SSL 종료 | 가능 | ✅ 더 효율적 |
| 로드 밸런싱 | ❌ | ✅ |
| 정적 파일 캐싱 | 가능 | ✅ 더 효율적 |
| DDoS 방어 | 기본만 | ✅ 강력 |
| gzip 압축 | 미들웨어 | ✅ |
| HTTP/2 → HTTP/1.1 변환 | ❌ | ✅ |
| 헬스 체크 | 직접 구현 | ✅ 내장 |

---

## Nginx 구성

### 기본 설정

```nginx
# /etc/nginx/sites-available/myapp
upstream backend {
    server localhost:5000;

    # 여러 서버로 로드 밸런싱
    # server localhost:5001;
    # server localhost:5002;

    # 연결 유지 (성능 향상)
    keepalive 32;
}

server {
    listen 80;
    server_name example.com www.example.com;

    # HTTPS로 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL 인증서
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL 설정 최적화
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;

    # 보안 헤더
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # 압축
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;

    # 정적 파일
    location /static/ {
        alias /var/www/myapp/wwwroot/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # API 요청
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;

        # 연결 유지
        proxy_set_header Connection "";

        # 원본 클라이언트 정보 전달
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # 타임아웃
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # 버퍼링
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # WebSocket 지원 (SignalR)
    location /hubs/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 타임아웃
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    # 헬스 체크 엔드포인트
    location /health {
        proxy_pass http://backend/health;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # 캐시하지 않음
        add_header Cache-Control "no-cache, no-store";
    }
}
```

### 로드 밸런싱 설정

```nginx
upstream backend {
    # 기본: 라운드 로빈
    server localhost:5000 weight=5;
    server localhost:5001 weight=3;
    server localhost:5002 weight=2;

    # IP 해시 (세션 고정)
    # ip_hash;

    # 최소 연결
    # least_conn;

    # 헬스 체크 (Nginx Plus)
    # health_check interval=5s fails=3 passes=2;
}
```

---

## Apache 구성

### 기본 설정

```apache
# /etc/apache2/sites-available/myapp.conf
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName example.com

    # SSL
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    # 프록시 설정
    ProxyPreserveHost On
    ProxyPass / http://localhost:5000/
    ProxyPassReverse / http://localhost:5000/

    # 헤더 전달
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"

    # WebSocket 지원
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) ws://localhost:5000/$1 [P,L]

    # 압축
    <IfModule mod_deflate.c>
        AddOutputFilterByType DEFLATE text/html text/plain text/css application/json application/javascript
    </IfModule>

    # 정적 파일 캐싱
    <Location /static>
        ExpiresActive On
        ExpiresDefault "access plus 30 days"
    </Location>
</VirtualHost>
```

### 필요한 모듈 활성화

```bash
# 모듈 활성화
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_wstunnel
sudo a2enmod ssl
sudo a2enmod headers
sudo a2enmod rewrite
sudo a2enmod deflate
sudo a2enmod expires

# Apache 재시작
sudo systemctl restart apache2
```

---

## ASP.NET Core 설정

### Forwarded Headers 미들웨어

```csharp
var builder = WebApplication.CreateBuilder(args);

// Forwarded Headers 옵션 설정
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders =
        ForwardedHeaders.XForwardedFor |
        ForwardedHeaders.XForwardedProto |
        ForwardedHeaders.XForwardedHost;

    // 기본적으로 localhost만 신뢰
    // 프로덕션에서는 프록시 IP를 명시적으로 지정
    options.KnownProxies.Clear();
    options.KnownNetworks.Clear();

    // 프록시 IP 추가 (예: 내부 네트워크)
    options.KnownNetworks.Add(new IPNetwork(
        IPAddress.Parse("10.0.0.0"), 8));
    options.KnownNetworks.Add(new IPNetwork(
        IPAddress.Parse("172.16.0.0"), 12));
    options.KnownNetworks.Add(new IPNetwork(
        IPAddress.Parse("192.168.0.0"), 16));

    // 또는 특정 프록시 IP
    // options.KnownProxies.Add(IPAddress.Parse("10.0.0.1"));

    // 헤더 이름 커스터마이징 (필요시)
    // options.ForwardedForHeaderName = "X-Forwarded-For";
    // options.ForwardedProtoHeaderName = "X-Forwarded-Proto";
});

var app = builder.Build();

// 미들웨어 파이프라인 최상단에 배치!
app.UseForwardedHeaders();

// HTTPS 리다이렉션 (프록시에서 처리하면 제거)
// app.UseHttpsRedirection();

app.MapGet("/", (HttpContext ctx) =>
{
    var info = new
    {
        RemoteIP = ctx.Connection.RemoteIpAddress?.ToString(),
        Scheme = ctx.Request.Scheme,
        Host = ctx.Request.Host.ToString(),
        Path = ctx.Request.Path.ToString()
    };
    return Results.Json(info);
});

app.Run();
```

### 환경별 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

// 프로덕션 환경에서만 Forwarded Headers 활성화
if (!builder.Environment.IsDevelopment())
{
    builder.Services.Configure<ForwardedHeadersOptions>(options =>
    {
        options.ForwardedHeaders =
            ForwardedHeaders.XForwardedFor |
            ForwardedHeaders.XForwardedProto;

        // Docker/Kubernetes 환경
        options.KnownNetworks.Clear();
        options.KnownProxies.Clear();
    });
}

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseForwardedHeaders();
    app.UseHsts();
}

app.Run();
```

### appsettings.json 설정

```json
{
  "ForwardedHeaders": {
    "ForwardedHeaders": "XForwardedFor,XForwardedProto",
    "KnownProxies": ["10.0.0.1", "10.0.0.2"],
    "KnownNetworks": ["10.0.0.0/8", "172.16.0.0/12"]
  }
}
```

```csharp
// 설정 바인딩
builder.Services.Configure<ForwardedHeadersOptions>(
    builder.Configuration.GetSection("ForwardedHeaders"));
```

---

## YARP (Yet Another Reverse Proxy)

ASP.NET Core로 직접 리버스 프록시를 구현할 때 사용합니다.

### 설치

```bash
dotnet add package Yarp.ReverseProxy
```

### 기본 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

// YARP 서비스 등록
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// YARP 미들웨어
app.MapReverseProxy();

app.Run();
```

### appsettings.json

```json
{
  "ReverseProxy": {
    "Routes": {
      "api-route": {
        "ClusterId": "api-cluster",
        "Match": {
          "Path": "/api/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api" }
        ]
      },
      "web-route": {
        "ClusterId": "web-cluster",
        "Match": {
          "Path": "{**catch-all}"
        }
      }
    },
    "Clusters": {
      "api-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:05",
            "Path": "/health"
          }
        },
        "Destinations": {
          "api1": {
            "Address": "http://localhost:5001"
          },
          "api2": {
            "Address": "http://localhost:5002"
          }
        }
      },
      "web-cluster": {
        "Destinations": {
          "web1": {
            "Address": "http://localhost:5000"
          }
        }
      }
    }
  }
}
```

### 코드 기반 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromMemory(
        routes: new[]
        {
            new RouteConfig
            {
                RouteId = "api",
                ClusterId = "api-cluster",
                Match = new RouteMatch { Path = "/api/{**catch-all}" }
            }
        },
        clusters: new[]
        {
            new ClusterConfig
            {
                ClusterId = "api-cluster",
                LoadBalancingPolicy = LoadBalancingPolicies.RoundRobin,
                Destinations = new Dictionary<string, DestinationConfig>
                {
                    ["api1"] = new() { Address = "http://localhost:5001" },
                    ["api2"] = new() { Address = "http://localhost:5002" }
                }
            }
        });

var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

### 커스텀 변환

```csharp
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(context =>
    {
        // 요청 헤더 추가
        context.AddRequestHeader("X-Custom-Header", "value");

        // 응답 헤더 추가
        context.AddResponseHeader("X-Proxy-By", "YARP");

        // 요청 경로 변환
        context.AddPathPrefix("/v1");
    });
```

---

## 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                    프로덕션 배포 아키텍처                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     인터넷                                                      │
│        │                                                        │
│        ▼                                                        │
│  ┌─────────────┐                                                │
│  │    CDN      │  정적 파일, 이미지 캐싱                        │
│  │ (CloudFlare)│                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │   L4 LB     │  TCP/UDP 로드 밸런싱                           │
│  │  (AWS NLB)  │  (고가용성)                                    │
│  └──────┬──────┘                                                │
│         │                                                        │
│    ┌────┴────┐                                                  │
│    ▼         ▼                                                  │
│ ┌──────┐  ┌──────┐                                              │
│ │Nginx │  │Nginx │  L7 리버스 프록시                            │
│ │  #1  │  │  #2  │  (SSL 종료, 라우팅)                          │
│ └──┬───┘  └──┬───┘                                              │
│    │         │                                                  │
│    └────┬────┘                                                  │
│         │                                                        │
│    ┌────┼────┬────┐                                             │
│    ▼    ▼    ▼    ▼                                             │
│ ┌────┐┌────┐┌────┐┌────┐                                        │
│ │App ││App ││App ││App │  Kestrel 인스턴스                      │
│ │ #1 ││ #2 ││ #3 ││ #4 │  (Docker 컨테이너)                     │
│ └────┘└────┘└────┘└────┘                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 문제 해결

### 일반적인 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| 원본 IP가 프록시 IP | UseForwardedHeaders 누락 | 미들웨어 추가 |
| HTTPS가 HTTP로 감지 | X-Forwarded-Proto 미전달 | 프록시 설정 확인 |
| 리다이렉트 루프 | HTTPS 리다이렉션 중복 | 프록시/앱 중 하나만 처리 |
| WebSocket 연결 실패 | Upgrade 헤더 미전달 | 프록시 WebSocket 설정 |
| 502 Bad Gateway | 백엔드 연결 실패 | 앱 실행 상태 확인 |
| 504 Gateway Timeout | 백엔드 응답 지연 | 타임아웃 증가 |

### 디버깅 팁

```csharp
// 프록시 헤더 확인용 엔드포인트
app.MapGet("/debug/headers", (HttpContext ctx) =>
{
    var headers = ctx.Request.Headers
        .Where(h => h.Key.StartsWith("X-", StringComparison.OrdinalIgnoreCase))
        .ToDictionary(h => h.Key, h => h.Value.ToString());

    return Results.Json(new
    {
        RemoteIP = ctx.Connection.RemoteIpAddress?.ToString(),
        RemotePort = ctx.Connection.RemotePort,
        Scheme = ctx.Request.Scheme,
        Host = ctx.Request.Host.ToString(),
        ForwardedHeaders = headers
    });
});
```

---

## 면접 예상 질문

### Q1: 리버스 프록시를 사용하는 이유는?

**A:**
1. **보안**: DDoS 방어, WAF, SSL 종료를 프록시에서 처리
2. **로드 밸런싱**: 여러 백엔드 서버로 트래픽 분산
3. **캐싱**: 정적 콘텐츠 캐싱으로 백엔드 부하 감소
4. **압축**: gzip/Brotli 압축 처리
5. **유연성**: URL 재작성, 헤더 조작

### Q2: UseForwardedHeaders()를 파이프라인 최상단에 배치해야 하는 이유는?

**A:** 다른 미들웨어들이 올바른 클라이언트 IP, 프로토콜(HTTP/HTTPS), 호스트 정보를 사용할 수 있어야 합니다. 예를 들어 HTTPS 리다이렉션 미들웨어가 X-Forwarded-Proto를 확인하려면 ForwardedHeaders가 먼저 실행되어야 합니다.

### Q3: KnownProxies/KnownNetworks를 설정해야 하는 이유는?

**A:** 보안상의 이유입니다. 아무나 X-Forwarded-For 헤더를 조작하여 IP 스푸핑을 할 수 있습니다. 신뢰할 수 있는 프록시 IP만 지정하면 해당 프록시에서 온 헤더만 신뢰합니다.

---

## 참고 자료

- [프록시 서버 및 로드 밸런서 구성](https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer)
- [YARP 공식 문서](https://microsoft.github.io/reverse-proxy/)
- [Nginx 리버스 프록시 가이드](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

---

## 다음 섹션

→ [03. Request Pipeline](../03-request-pipeline/) - 미들웨어 파이프라인 이해하기
