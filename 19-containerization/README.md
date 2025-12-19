# 19. Containerization - 컨테이너화

Docker와 Kubernetes를 이용한 ASP.NET Core 컨테이너화를 이해합니다.

---

## Dockerfile 예시

```dockerfile
# 멀티 스테이지 빌드
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# 의존성 복원 (캐시 최적화)
COPY *.csproj .
RUN dotnet restore

# 빌드
COPY . .
RUN dotnet publish -c Release -o /app

# 런타임 이미지
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## Kubernetes 배포

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
```

---

## 다음 단계

→ [20. Game Server](../20-game-server/) - 게임 서버
