# IIS 연동

## 개요

IIS(Internet Information Services)는 Windows Server의 기본 웹 서버입니다. ASP.NET Core 앱을 IIS에서 호스팅할 때는 **ASP.NET Core Module (ANCM)**을 통해 연동됩니다.

---

## 호스팅 모델

### In-Process vs Out-of-Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    In-Process 호스팅                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐     │
│    │                   IIS (w3wp.exe)                    │     │
│    │  ┌───────────────────────────────────────────────┐  │     │
│    │  │         ASP.NET Core Module (ANCM)            │  │     │
│    │  │  ┌─────────────────────────────────────────┐  │  │     │
│    │  │  │          ASP.NET Core 앱               │  │  │     │
│    │  │  │         (IIS 프로세스 내부)             │  │  │     │
│    │  │  └─────────────────────────────────────────┘  │  │     │
│    │  └───────────────────────────────────────────────┘  │     │
│    └─────────────────────────────────────────────────────┘     │
│                                                                 │
│    장점: 프록시 없음 → 더 빠름                                   │
│    단점: IIS 앱 풀 재활용 시 앱도 재시작                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Out-of-Process 호스팅                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────────────────────────────────────────────┐     │
│    │                   IIS (w3wp.exe)                    │     │
│    │  ┌───────────────────────────────────────────────┐  │     │
│    │  │         ASP.NET Core Module (ANCM)            │  │     │
│    │  │              (리버스 프록시)                   │  │     │
│    │  └─────────────────────┬─────────────────────────┘  │     │
│    └────────────────────────┼────────────────────────────┘     │
│                             │                                   │
│                             ▼ localhost:xxxxx                   │
│    ┌─────────────────────────────────────────────────────┐     │
│    │              별도 dotnet.exe 프로세스               │     │
│    │  ┌───────────────────────────────────────────────┐  │     │
│    │  │        Kestrel + ASP.NET Core 앱             │  │     │
│    │  └───────────────────────────────────────────────┘  │     │
│    └─────────────────────────────────────────────────────┘     │
│                                                                 │
│    장점: 프로세스 격리, 독립적 재시작                            │
│    단점: 프록시 오버헤드                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 성능 비교

| 항목 | In-Process | Out-of-Process |
|------|------------|----------------|
| 처리량 | ~30% 더 빠름 | 기준 |
| 메모리 | 공유 | 추가 프로세스 |
| 시작 시간 | 빠름 | 프로세스 시작 필요 |
| 앱 풀 재활용 | 영향 받음 | 독립적 |
| 디버깅 | IIS 디버깅 | dotnet.exe 디버깅 |

---

## ASP.NET Core Module (ANCM)

### 역할

- In-Process: IIS 워커 프로세스 내에서 .NET 런타임 호스팅
- Out-of-Process: 요청을 Kestrel로 프록시

### 설치 확인

```powershell
# IIS에 ANCM이 설치되어 있는지 확인
Get-WebGlobalModule | Where-Object { $_.Name -like "*AspNetCore*" }

# 또는 IIS 관리자에서 확인:
# 서버 → 모듈 → AspNetCoreModuleV2
```

### ANCM 버전

| 버전 | 지원 .NET | 호스팅 모델 |
|------|----------|------------|
| ANCM v1 | .NET Core 2.0 | Out-of-Process만 |
| ANCM v2 | .NET Core 2.2+ | In-Process + Out-of-Process |

---

## web.config 설정

### In-Process (기본값, 권장)

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*"
             modules="AspNetCoreModuleV2"
             resourceType="Unspecified" />
      </handlers>

      <aspNetCore processPath="dotnet"
                  arguments=".\MyApp.dll"
                  stdoutLogEnabled="false"
                  stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess">

        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT"
                               value="Production" />
        </environmentVariables>
      </aspNetCore>
    </system.webServer>
  </location>
</configuration>
```

### Out-of-Process

```xml
<aspNetCore processPath="dotnet"
            arguments=".\MyApp.dll"
            stdoutLogEnabled="true"
            stdoutLogFile=".\logs\stdout"
            hostingModel="outofprocess">

  <!-- Kestrel이 사용할 포트 지정 (선택) -->
  <environmentVariables>
    <environmentVariable name="ASPNETCORE_URLS"
                         value="http://localhost:5000" />
  </environmentVariables>
</aspNetCore>
```

### 주요 속성

| 속성 | 설명 | 기본값 |
|------|------|--------|
| `processPath` | 실행할 프로세스 | `dotnet` |
| `arguments` | 프로세스 인수 | `.\MyApp.dll` |
| `hostingModel` | `inprocess` 또는 `outofprocess` | `inprocess` |
| `stdoutLogEnabled` | stdout 로깅 활성화 | `false` |
| `stdoutLogFile` | stdout 로그 경로 | `.\logs\stdout` |
| `startupTimeLimit` | 시작 대기 시간 (초) | `120` |
| `shutdownTimeLimit` | 종료 대기 시간 (초) | `10` |
| `requestTimeout` | 요청 타임아웃 | `00:02:00` |

---

## .csproj 설정

```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>

  <!-- 호스팅 모델 지정 -->
  <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>
  <!-- 또는 OutOfProcess -->

  <!-- web.config 변환 비활성화 (필요한 경우) -->
  <IsTransformWebConfigDisabled>true</IsTransformWebConfigDisabled>
</PropertyGroup>
```

---

## 앱 풀 구성

### 권장 설정

```powershell
# PowerShell로 앱 풀 생성
Import-Module WebAdministration

# 앱 풀 생성
New-WebAppPool -Name "MyAppPool"

# 설정 적용
Set-ItemProperty "IIS:\AppPools\MyAppPool" -Name "managedRuntimeVersion" -Value ""  # CLR 없음
Set-ItemProperty "IIS:\AppPools\MyAppPool" -Name "enable32BitAppOnWin64" -Value $false
Set-ItemProperty "IIS:\AppPools\MyAppPool" -Name "processModel.identityType" -Value "ApplicationPoolIdentity"

# 재활용 설정
Set-ItemProperty "IIS:\AppPools\MyAppPool" -Name "recycling.periodicRestart.time" -Value "00:00:00"  # 비활성화
Set-ItemProperty "IIS:\AppPools\MyAppPool" -Name "processModel.idleTimeout" -Value "00:00:00"  # 비활성화
```

### 앱 풀 설정 표

| 설정 | 값 | 이유 |
|------|-----|------|
| .NET CLR 버전 | 관리 코드 없음 | ANCM이 관리 |
| 관리 파이프라인 | 통합 | 기본값 |
| 32비트 앱 활성화 | False | 64비트 권장 |
| 재활용 시간 간격 | 0 (비활성화) | 자체 관리 |
| 유휴 시간 제한 | 0 (비활성화) | 항상 실행 |

---

## IIS 사이트 배포

### 방법 1: Web Deploy

```powershell
# Visual Studio에서 게시 프로필 사용
# 또는 명령줄
dotnet publish -c Release
msdeploy.exe -source:contentPath=".\publish" -dest:contentPath="MySite",computerName="server"
```

### 방법 2: xcopy/Robocopy

```powershell
# 빌드
dotnet publish -c Release -o .\publish

# 복사 (앱 중지 필요)
Stop-WebAppPool -Name "MyAppPool"
robocopy .\publish \\server\c$\inetpub\wwwroot\myapp /MIR
Start-WebAppPool -Name "MyAppPool"
```

### 방법 3: app_offline.htm

```powershell
# 유지보수 모드로 전환
"<html><body>Updating...</body></html>" | Out-File "C:\inetpub\wwwroot\myapp\app_offline.htm"

# 파일 배포...

# 유지보수 모드 해제
Remove-Item "C:\inetpub\wwwroot\myapp\app_offline.htm"
```

---

## 문제 해결

### stdout 로깅 활성화

```xml
<aspNetCore processPath="dotnet"
            arguments=".\MyApp.dll"
            stdoutLogEnabled="true"
            stdoutLogFile=".\logs\stdout"
            hostingModel="inprocess">
```

```powershell
# logs 폴더 생성 및 권한 부여
New-Item -ItemType Directory -Path "C:\inetpub\wwwroot\myapp\logs"

# 앱 풀 계정에 쓰기 권한
$acl = Get-Acl "C:\inetpub\wwwroot\myapp\logs"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "IIS AppPool\MyAppPool", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)
Set-Acl "C:\inetpub\wwwroot\myapp\logs" $acl
```

### 일반적인 오류

| 오류 코드 | 원인 | 해결 |
|----------|------|------|
| 500.19 | web.config 문법 오류 | XML 검증 |
| 500.21 | ANCM 미설치 | Hosting Bundle 설치 |
| 500.30 | 앱 시작 실패 | stdout 로그 확인 |
| 500.31 | 네이티브 의존성 누락 | VC++ 런타임 설치 |
| 500.32 | ANCM Out-of-Process 시작 실패 | dotnet 경로 확인 |
| 500.33 | ANCM In-Process 시작 실패 | 앱 코드 오류 확인 |
| 500.34 | ANCM In-Process 요청 처리 실패 | 앱 예외 확인 |
| 502.5 | 프로세스 실패 | stdout 로그 확인 |

### 진단 명령

```powershell
# .NET 런타임 확인
dotnet --list-runtimes

# 앱 직접 실행 테스트
cd C:\inetpub\wwwroot\myapp
dotnet MyApp.dll

# ANCM 로그 위치
# C:\inetpub\logs\LogFiles\W3SVC{사이트ID}\
# 또는 이벤트 뷰어 → 응용 프로그램 로그
```

---

## Windows 인증

### IIS 설정

```powershell
# Windows 인증 활성화
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/windowsAuthentication" `
    -Name "enabled" -Value "True" -PSPath "IIS:\Sites\MySite"

# 익명 인증 비활성화
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/anonymousAuthentication" `
    -Name "enabled" -Value "False" -PSPath "IIS:\Sites\MySite"
```

### ASP.NET Core 설정

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(IISDefaults.AuthenticationScheme);
builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/", (ClaimsPrincipal user) =>
{
    return $"Hello, {user.Identity?.Name}";
}).RequireAuthorization();

app.Run();
```

```xml
<!-- web.config - forwardWindowsAuthToken 활성화 (Out-of-Process) -->
<aspNetCore processPath="dotnet"
            arguments=".\MyApp.dll"
            stdoutLogEnabled="false"
            hostingModel="outofprocess"
            forwardWindowsAuthToken="true" />
```

---

## 면접 예상 질문

### Q1: In-Process와 Out-of-Process의 차이점은?

**A:**
- **In-Process**: IIS 워커 프로세스(w3wp.exe) 내부에서 앱 실행. 프록시 없이 직접 처리하므로 약 30% 더 빠름.
- **Out-of-Process**: 별도 dotnet.exe 프로세스에서 Kestrel 실행. ANCM이 프록시 역할. 프로세스가 완전히 격리됨.

### Q2: web.config의 hostingModel 기본값은?

**A:** .NET Core 3.0부터 기본값은 `inprocess`입니다.

### Q3: 500.30 오류가 발생하면 어떻게 디버깅하나요?

**A:**
1. `stdoutLogEnabled="true"` 설정
2. stdout 로그 확인 (logs 폴더)
3. 앱을 명령줄에서 직접 실행하여 오류 확인
4. 이벤트 뷰어 확인

---

## 참고 자료

- [IIS에서 ASP.NET Core 호스팅](https://learn.microsoft.com/aspnet/core/host-and-deploy/iis/)
- [ASP.NET Core Module](https://learn.microsoft.com/aspnet/core/host-and-deploy/aspnet-core-module)
- [IIS 문제 해결](https://learn.microsoft.com/aspnet/core/test/troubleshoot-azure-iis)

---

## 다음 문서

→ [reverse-proxy.md](./reverse-proxy.md) - 리버스 프록시 패턴
