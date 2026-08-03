# Webhook 채널 도입 가이드 (Polaris)

작성일 2026-07-30 · 대상: AE 운영 모듈의 Webhook 터치 채널 (설정센터의 컨피그 채널 아님)
전제: Polaris 서버는 C#(.NET), K8s 클러스터 운영 중, Analytics SDK(TDAnalytics)는 이미 연동됨

전체 순서: **0 설계 → 1 서버 구축 → 2 K8s 배포 → 3 AE 채널 생성 → 4 SDK(이벤트) 작업 → 5 테스트 → 6 첫 발송**

---

## 주소 구분 (혼동 방지)

| 주소 | 정체 | 웹훅 URL로 사용 |
|---|---|---|
| `https://underdox-kr-web.thinkingai.io` | AE 콘솔 (관리 화면) | ❌ |
| `https://underdox-kr-receiver.thinkingai.io/` | SDK 데이터 수신기 (SERVER_URL) | ❌ |
| `https://<Polaris API 도메인>/ae/engage/webhook` | **언더독스가 구축할 웹훅 수신 엔드포인트** | ✅ |

위 두 개는 언더독스 → ThinkingAI 방향(데이터 수집)이고, 웹훅은 반대로 ThinkingAI → 언더독스 방향이라 받는 쪽 주소가 언더독스 인프라에 있어야 한다. Polaris 게임 서버의 외부 API 도메인(K8s Ingress 호스트)은 서버팀 확인 필요.

## STEP 0. 설계 결정 (30분)

코드를 만지기 전에 3가지를 확정한다.

| 결정 항목 | 권장값 (Polaris) | 이유 |
|---|---|---|
| 전달 수단 | 인게임 우편함(메일) 지급 | 오프라인 유저에게도 확실히 전달, 보상 첨부 가능 |
| 수신자 식별 키 (전송 ID) | `#account_id` | 서버가 계정을 바로 찾을 수 있는 키 |
| 메시지 스키마 (컨텐츠 템플릿) | `title`, `body`, `reward_id`(선택), `expire_days`(선택) | 운영자가 입력할 필드 = 서버가 받을 필드 |

## STEP 1. 수신 서버 구현 [서버 개발]

### 1-1. AE가 보내는 요청 (수신 스펙)

AE는 발송 시점에 채널에 등록된 URL로 아래 형태의 POST를 보낸다:

```json
POST /ae/engage/webhook
Content-Type: application/json

{
  "message_id": "msg_xxx",          // 발송 건 고유 ID (멱등성 키)
  "user_id": "<전송 ID 속성값>",     // = #account_id 값
  "channel": "webhook",
  "content": {
    "title": "복귀 보상이 도착했어요",
    "body": "7일 출석 보상을 우편함에서 확인하세요",
    "extra": { "reward_id": "comeback_pack_01", "expire_days": "7" }
  },
  "timestamp": 1699999999
}
```

### 1-2. 엔드포인트 구현 (ASP.NET Core 예시)

```csharp
// Program.cs (minimal API 예시 — 기존 게임서버 프로젝트에 추가)
app.MapPost("/ae/engage/webhook", async (HttpRequest req, MailService mail) =>
{
    // 1) 인증 검증 (아래 1-3 중 택1 이상)
    if (!WebhookAuth.Verify(req)) return Results.Unauthorized();

    var payload = await req.ReadFromJsonAsync<AeWebhookPayload>();

    // 2) 멱등성: 같은 message_id 재수신 시 무시 (AE 재시도 대비)
    if (await mail.AlreadyProcessed(payload.MessageId))
        return Results.Ok(new { code = 0 });

    // 3) 비즈니스 처리: 인게임 메일 발송
    await mail.SendToMailbox(
        accountId: payload.UserId,
        title: payload.Content.Title,
        body: payload.Content.Body,
        rewardId: payload.Content.Extra.GetValueOrDefault("reward_id"),
        expireDays: int.Parse(payload.Content.Extra.GetValueOrDefault("expire_days", "30")));

    // 4) 즉시 200 응답 (처리 결과 receipt는 별도 리포트 — 1-4)
    return Results.Ok(new { code = 0 });
});
```

구현 원칙:
- **5초 안에 응답**한다. 무거운 처리는 큐에 넣고 즉시 200을 반환 (타임아웃 → AE가 실패 처리/재시도)
- **멱등성 필수**: `message_id` 기준 중복 처리 방지 (Redis SETNX 또는 DB unique key)
- 대량 발송 대비: 태스크 1건이 수천 콜을 만들 수 있음 → 큐 + 워커 구조 권장

### 1-3. 인증 (3가지 중 택1 이상)

| 방식 | 구현 |
|---|---|
| HMAC-SHA256 서명 | 채널에 설정한 시크릿으로 요청 바디 서명 검증 (권장) |
| Bearer 토큰 | 커스텀 파라미터로 고정 토큰을 실어 보내고 서버에서 대조 |
| IP 화이트리스트 | AE 서버 대역만 Ingress/방화벽에서 허용 (보조 수단) |

```csharp
// HMAC 검증 예시
static bool VerifyHmac(string body, string signatureHeader, string secret)
{
    using var h = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    var expected = Convert.ToHexString(h.ComputeHash(Encoding.UTF8.GetBytes(body))).ToLower();
    return CryptographicOperations.FixedTimeEquals(
        Encoding.UTF8.GetBytes(expected), Encoding.UTF8.GetBytes(signatureHeader ?? ""));
}
```

### 1-4. 발송 결과 receipt 리포트

처리 성공/실패를 AE에 회신해야 도달 퍼널이 정확해진다:

```json
{ "message_id": "msg_xxx", "user_id": "xxx", "status": "success",  // 또는 "failed"
  "error_msg": "", "timestamp": 1699999999 }
```

(회신 엔드포인트 주소·형식은 채널 저장 후 "매뉴얼 보기"의 receipt 규격을 따를 것)

### 1-5. 채널 검증 대응

콘솔의 "채널 검증" 토글을 켜면 저장 시 AE가 URL로 검증 요청을 보낸다. 서버가 정상 응답(매뉴얼 규격, 통상 200 + echo)을 돌려줘야 저장이 완료된다. 배포가 끝난 뒤에 채널을 저장하는 순서인 이유.

## STEP 2. K8s 배포 [서버 개발]

새 인프라는 필요 없다. 두 가지 중 택1:

**A안 (권장, 간단): 기존 게임서버에 라우트 추가** — 위 엔드포인트를 기존 서비스에 포함해 배포하고 Ingress에 경로만 추가:

```yaml
# ingress에 추가
- path: /ae/engage/webhook
  pathType: Prefix
  backend: { service: { name: game-api, port: { number: 80 } } }
```

**B안 (격리 원할 때): 전용 마이크로서비스** — `engage-webhook` Deployment + Service + Ingress 분리. 대량 발송 트래픽이 게임 API에 영향 주지 않도록 격리. HPA로 발송 피크 대응.

검증:
```bash
curl -X POST https://api.<도메인>/ae/engage/webhook \
  -H "Content-Type: application/json" \
  -d '{"message_id":"test_1","user_id":"<테스트계정ID>","channel":"webhook","content":{"title":"t","body":"b","extra":{}},"timestamp":0}'
# 기대: 200 + 테스트 계정 우편함에 메일 도착
```

## STEP 3. AE 콘솔에서 채널 생성 [플랫폼 관리자]

운영 모듈 → 채널 설정 → Webhook 채널 생성 (화면 기준):

| 필드 | 입력값 | 비고 |
|---|---|---|
| 채널 이름 | `polaris-ingame-mail` | 용도가 드러나게 |
| URL | `https://api.<도메인>/ae/engage/webhook` | STEP 2에서 배포한 주소 |
| 전송 ID | `#account_id` | 드롭다운에서 선택 — payload의 `user_id`로 실림 |
| 채널 검증 | ON | 서버 배포 완료 후 저장해야 통과 (1-5) |
| 도달 퍼널 설정 | ON | 발송→도달→클릭 집계. 도달/클릭 이벤트 이름 매핑 (STEP 4) |
| 컨텐츠 템플릿 | 아래 표 | 운영자 입력 폼 = payload `content` 구조 |
| 커스텀 파라미터 | `env=prod` (+Bearer 토큰 방식이면 `auth_token=...`) | 전 요청 공통 첨부 |

컨텐츠 템플릿:

| 필드 | 표시 이름 | 입력 방식 | 필수 |
|---|---|---|---|
| `title` | 메일 제목 | 텍스트 | ✓ |
| `body` | 메일 본문 | 텍스트 | ✓ |
| `reward_id` | 보상 ID | 텍스트 | — |
| `expire_days` | 만료일(일) | 텍스트 (기본값 30) | — |

CLI 확인: `ae-cli engage-setting channel list -p 3` / 상세 `channel get`

## STEP 4. SDK·이벤트 작업 [클라이언트 개발]

**중요: Webhook 채널 발송 자체에는 SDK가 필요 없다.** SDK가 관여하는 지점은 두 곳뿐:

1. **전송 ID 속성** — `#account_id`가 유저 속성으로 리포트되고 있어야 대상 선정·발송이 된다. Polaris는 기존 Analytics 연동으로 이미 충족 (추가 작업 없음)
2. **도달 퍼널 이벤트** — 퍼널의 "도달·클릭"을 채우려면 클라이언트가 이벤트를 쏴야 한다:
   - 도달: 우편함에 메일 생성 시 서버가 이벤트 리포트 (서버 SDK, 예: `mail_delivered`)
   - 클릭: 유저가 메일을 열람/수령할 때 클라이언트가 리포트 (예: `mail_opened`, 기존 TDAnalytics `track` 사용)
   - 채널의 도달 퍼널 설정에서 이 이벤트 이름을 매핑 (CLI 대응: `--event-delivery-name`, `--event-click-name`)

즉 신규 SDK 설치는 없고, **이벤트 2종 추가 + 트래킹 플랜 반영**이 클라이언트/서버 작업의 전부다.
(FCM 푸시를 나중에 도입할 때 비로소 `SetPushToken` 등 SDK 확장이 필요해진다)

## STEP 5. 테스트 [공동]

1. 테스트 허용목록 등록: `ae-cli engage-setting whitelist add` (entity + 테스트 계정 ID)
2. 채널 테스트 발송: 콘솔 또는 `ae-cli engage-setting channel test-send` — 수신자(push_id)에 테스트 계정, 컨텐츠 템플릿 값 입력
3. 확인 체크리스트:
   - [ ] 서버 로그에 요청 수신 + 인증 통과
   - [ ] 테스트 계정 우편함에 메일 도착 (보상 첨부 포함)
   - [ ] 같은 message_id 재전송 시 중복 지급 안 됨
   - [ ] receipt가 AE에 기록되고, `mail_opened` 클릭 이벤트가 유입됨

## STEP 6. 첫 운영 태스크 [운영자]

1. 세그먼트: "설치 후 24h 미접속" 등 (D1 복귀 시나리오)
2. 수동 태스크 생성 → 채널 `polaris-ingame-mail` 선택 → 템플릿 필드 입력 → 허용목록 대상 시험 발송 → 본 발송
3. 검증: 발송 건수 = 대상 수, `engage-task push-record`로 발송 기록, 도달 퍼널로 도달·클릭률 확인

## 트러블슈팅

| 증상 | 확인 |
|---|---|
| 채널 저장 실패 (검증 실패) | 서버 미배포/URL 오타/검증 응답 규격 — 매뉴얼의 검증 스펙 확인 |
| 발송했는데 서버에 요청 없음 | 채널 상태 OFF, 전송 ID 속성값 비어 있는 유저, Ingress 경로 |
| 메일 중복 지급 | message_id 멱등 처리 누락 |
| 도달 퍼널이 비어 있음 | receipt 미회신, 도달/클릭 이벤트 이름 매핑 불일치 |
