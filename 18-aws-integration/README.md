# 18. AWS Integration

AWS SDK 및 서비스와 ASP.NET Core 통합을 이해합니다.

---

## 주요 AWS 서비스

| 서비스 | 용도 | NuGet 패키지 |
|--------|------|-------------|
| S3 | 객체 스토리지 | AWSSDK.S3 |
| SQS | 메시지 큐 | AWSSDK.SQS |
| DynamoDB | NoSQL DB | AWSSDK.DynamoDBv2 |
| Lambda | 서버리스 | Amazon.Lambda.AspNetCoreServer |
| ECS/Fargate | 컨테이너 | 배포 타겟 |
| Secrets Manager | 비밀 관리 | AWSSDK.SecretsManager |

---

## 빠른 시작

```csharp
// AWS SDK 설정
builder.Services.AddDefaultAWSOptions(
    builder.Configuration.GetAWSOptions());
builder.Services.AddAWSService<IAmazonS3>();
builder.Services.AddAWSService<IAmazonSQS>();
builder.Services.AddAWSService<IAmazonDynamoDB>();

// S3 사용
public class FileService(IAmazonS3 s3)
{
    public async Task UploadAsync(Stream stream, string key)
    {
        await s3.PutObjectAsync(new PutObjectRequest
        {
            BucketName = "my-bucket",
            Key = key,
            InputStream = stream
        });
    }
}
```

---

## 다음 단계

→ [19. Containerization](../19-containerization/) - 컨테이너화
