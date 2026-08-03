# 부록: 웹훅 수신 서버 구현 예시 (C# / ASP.NET Core)

Polaris 서버 스택 기준. 컨텐츠 템플릿 필드 ↔ 서버 코드가 어떻게 이어지는지 보여주는 참조 구현.

## 전제: 채널에 등록한 컨텐츠 템플릿

| 필드 (payload 키) | 표시 이름 | 필수 |
|---|---|---|
| `title` | 메일 제목 | ✓ |
| `body` | 메일 본문 | ✓ |
| `reward_id` | 보상 ID | — |
| `expire_days` | 만료일(일) | — |

**이 표가 곧 서버 개발 명세다.** 왼쪽 열 이름이 아래 DTO의 필드명과 1:1로 일치해야 한다.

---

## 1. DTO — AE가 보내는 JSON의 형태

```csharp
// Models/AeWebhookPayload.cs
using System.Text.Json.Serialization;

public sealed class AeWebhookPayload
{
    [JsonPropertyName("message_id")] public string MessageId { get; init; } = "";
    [JsonPropertyName("user_id")]    public string UserId    { get; init; } = "";  // = #account_id
    [JsonPropertyName("channel")]    public string Channel   { get; init; } = "";
    [JsonPropertyName("timestamp")]  public long   Timestamp { get; init; }
    [JsonPropertyName("content")]    public AeContent Content { get; init; } = new();
}

public sealed class AeContent
{
    // ↓ 컨텐츠 템플릿의 title / body 필드가 여기로 들어온다
    [JsonPropertyName("title")] public string Title { get; init; } = "";
    [JsonPropertyName("body")]  public string Body  { get; init; } = "";

    // ↓ 나머지 커스텀 필드(reward_id, expire_days)와 커스텀 파라미터는 extra로 들어온다
    [JsonPropertyName("extra")] public Dictionary<string, string> Extra { get; init; } = new();
}
```

> 주의: `content` 안에서 어떤 필드가 최상위로 오고 어떤 것이 `extra`로 묶이는지는 배포판에 따라 다를 수 있다.
> **개발 착수 전에 `webhook.site`로 시험 발송해 실제 JSON 형태를 눈으로 확인하고 DTO를 확정할 것.**
> (안전하게 가려면 `JsonExtensionData`로 미확인 필드를 모두 받아두는 방법도 있다.)

## 2. 엔드포인트 — 요청 수신부

```csharp
// Program.cs
app.MapPost("/ae/engage/webhook", async (
    HttpContext http,
    AeWebhookService service,
    ILogger<Program> log) =>
{
    // (1) 원문 보관 — HMAC 검증은 원문 기준이므로 역직렬화 전에 읽는다
    http.Request.EnableBuffering();
    using var reader = new StreamReader(http.Request.Body, leaveOpen: true);
    var raw = await reader.ReadToEndAsync();
    http.Request.Body.Position = 0;

    // (2) 인증
    var signature = http.Request.Headers["X-AE-Signature"].FirstOrDefault();
    if (!WebhookAuth.VerifyHmac(raw, signature))
    {
        log.LogWarning("AE webhook 서명 불일치");
        return Results.Unauthorized();
    }

    // (3) 파싱
    var payload = JsonSerializer.Deserialize<AeWebhookPayload>(raw);
    if (payload is null || string.IsNullOrEmpty(payload.MessageId))
        return Results.BadRequest(new { code = 400, msg = "invalid payload" });

    // (4) 큐에 적재하고 즉시 응답 — 5초 안에 200을 돌려주는 것이 핵심
    await service.EnqueueAsync(payload);
    return Results.Ok(new { code = 0 });
});
```

**왜 큐를 쓰는가**: 태스크 1건이 수천~수만 콜을 만들 수 있다. DB 쓰기와 보상 지급을 요청 스레드에서 처리하면 타임아웃이 나고, AE는 이를 발송 실패로 기록한다.

## 3. 인증 — HMAC-SHA256 검증

```csharp
// Services/WebhookAuth.cs
public static class WebhookAuth
{
    private static readonly string Secret =
        Environment.GetEnvironmentVariable("AE_WEBHOOK_SECRET")!;  // K8s Secret 주입

    public static bool VerifyHmac(string rawBody, string? signatureHeader)
    {
        if (string.IsNullOrEmpty(signatureHeader)) return false;

        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(Secret));
        var computed = Convert.ToHexString(
            hmac.ComputeHash(Encoding.UTF8.GetBytes(rawBody))).ToLowerInvariant();

        // 타이밍 공격 방지를 위해 고정시간 비교
        return CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(computed),
            Encoding.UTF8.GetBytes(signatureHeader.ToLowerInvariant()));
    }
}
```

> 서명 헤더 이름(`X-AE-Signature`)과 서명 대상(원문 전체 vs 특정 필드 조합)은 배포판 매뉴얼에서 확인해 맞출 것.
> HMAC을 쓰지 않는다면 커스텀 파라미터로 고정 토큰을 받아 대조하는 방식으로 대체 가능.

## 4. 처리 로직 — 템플릿 필드를 비즈니스로 연결

```csharp
// Services/AeWebhookService.cs
public sealed class AeWebhookService
{
    private readonly IMailboxService _mailbox;   // 기존 인게임 우편함 서비스
    private readonly IDistributedCache _cache;   // Redis
    private readonly IAeReceiptReporter _receipt;
    private readonly ILogger<AeWebhookService> _log;

    public async Task ProcessAsync(AeWebhookPayload p)
    {
        // (1) 멱등성 — AE 재시도로 인한 보상 중복 지급 방지
        var key = $"ae:webhook:{p.MessageId}";
        if (!await _cache.TrySetIfNotExistsAsync(key, "1", TimeSpan.FromDays(7)))
        {
            _log.LogInformation("중복 수신 무시: {MessageId}", p.MessageId);
            return;
        }

        try
        {
            // (2) 계정 조회 — user_id = 전송 ID로 지정한 #account_id
            var account = await _mailbox.FindAccountAsync(p.UserId);
            if (account is null)
            {
                await _receipt.ReportAsync(p, success: false, error: "account_not_found");
                return;
            }

            // (3) ★ 여기가 템플릿 필드 ↔ 서버 변수의 매핑 지점 ★
            var title      = p.Content.Title;                              // 템플릿 필드: title
            var body       = p.Content.Body;                               // 템플릿 필드: body
            var rewardId   = p.Content.Extra.GetValueOrDefault("reward_id");    // 템플릿 필드: reward_id
            var expireDays = int.TryParse(
                p.Content.Extra.GetValueOrDefault("expire_days"), out var d) ? d : 30;

            // (4) 우편함 지급 (기존 게임 로직 재사용)
            await _mailbox.SendAsync(new MailRequest
            {
                AccountId  = account.Id,
                Title      = title,
                Body       = body,                 // 개인화 변수는 AE가 이미 치환해서 보내줌
                RewardId   = rewardId,             // null이면 보상 없는 안내 메일
                ExpiresAt  = DateTime.UtcNow.AddDays(expireDays),
                SourceTag  = $"ae:{p.MessageId}",  // 추적용
            });

            // (5) 도달 이벤트 리포트 — 도달 퍼널의 '도달' 단계
            await _analytics.TrackAsync(account.Id, "mail_delivered", new {
                message_id = p.MessageId, reward_id = rewardId
            });

            // (6) receipt 회신 — 성공
            await _receipt.ReportAsync(p, success: true);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "AE 웹훅 처리 실패 {MessageId}", p.MessageId);
            await _cache.RemoveAsync(key);   // 재시도 허용을 위해 멱등키 해제
            await _receipt.ReportAsync(p, success: false, error: ex.Message);
        }
    }
}
```

## 5. receipt 회신

```csharp
// Services/AeReceiptReporter.cs
public async Task ReportAsync(AeWebhookPayload p, bool success, string? error = null)
{
    var payload = new
    {
        message_id = p.MessageId,
        user_id    = p.UserId,
        status     = success ? "success" : "failed",
        error_msg  = error ?? "",
        timestamp  = DateTimeOffset.UtcNow.ToUnixTimeSeconds(),
    };
    await _http.PostAsJsonAsync(_receiptUrl, payload);   // 회신 주소는 매뉴얼 규격 확인
}
```

## 6. K8s 배포 (기존 서비스에 라우트 추가하는 경우)

```yaml
# Secret — 웹훅 서명 키
apiVersion: v1
kind: Secret
metadata:
  name: ae-webhook
type: Opaque
stringData:
  secret: "<운영자가 AE 채널에 설정한 것과 동일한 값>"
---
# Deployment에 환경변수 주입
        env:
          - name: AE_WEBHOOK_SECRET
            valueFrom:
              secretKeyRef: { name: ae-webhook, key: secret }
---
# Ingress에 경로 추가
      - path: /ae/engage/webhook
        pathType: Prefix
        backend:
          service:
            name: game-api
            port: { number: 80 }
```

## 7. 로컬 검증

```bash
# 서명 생성 후 호출 (SECRET은 서버와 동일하게)
BODY='{"message_id":"test_1","user_id":"TEST_ACCOUNT","channel":"webhook","timestamp":0,"content":{"title":"테스트","body":"본문","extra":{"reward_id":"comeback_pack_01","expire_days":"7"}}}'
SIG=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$AE_WEBHOOK_SECRET" | awk '{print $2}')

curl -X POST https://<도메인>/ae/engage/webhook \
  -H "Content-Type: application/json" \
  -H "X-AE-Signature: $SIG" \
  -d "$BODY"
```

기대 결과: `{"code":0}` 응답 + 테스트 계정 우편함에 보상 메일 도착 + 동일 요청 재전송 시 중복 지급 없음.

---

## 필드를 추가할 때

예를 들어 「이동 링크(`deep_link`)」를 추가하고 싶다면 **양쪽을 같이** 바꿔야 한다:

1. AE 채널의 컨텐츠 템플릿에 `deep_link` 행 추가
2. 서버에서 `p.Content.Extra.GetValueOrDefault("deep_link")` 읽어 처리

한쪽만 바꾸면 값이 조용히 무시된다 — 자동 매핑이 아니라 이름 약속이기 때문.
