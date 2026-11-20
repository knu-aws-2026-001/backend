# 이어온 하루 — Backend MVP

> 프로젝트: **취소된 하루를 양도합니다** (AWS Skill Training 1조)
> 서비스명: **이어온 하루** — 취소된 동네 산책을 이어받고, 그날의 이야기를 돌려주는 모바일 웹

이 문서는 백엔드 저장소에서 구현할 MVP 범위와 **API 계약의 기준(Source of Truth)** 을 정리한다. 프론트엔드 화면 요구사항은 [frontend `MVP.md`](https://github.com/knu-aws-2026-001/frontend/blob/main/MVP.md)를 참고한다.

참고 자료 (팀 공유 PDF)
- `aws1조_초안.pdf` — 문제 정의, 목표, 아키텍처, 주요 기능, 데모 계획
- `이어온 하루-동네산책.pdf` — 최신 화면 시안 (동네 산책 버전, MVP 기준)
- `이어온 하루.pdf` — 이전 화면 시안 (캠퍼스 산책 버전)

---

## 1. 한 줄 요약

계획자가 남긴 산책 계획(공개 장소 1~3곳)을 **한 명의 실행자만** 이어받고, 실행자의 방문 기록(글 + 선택 사진 1장)이 **정확히 한 번** 계획자의 수신함에 도착하도록 처리하는 서버리스 API.

## 2. 백엔드가 보장해야 하는 것 (초안 목표에서 도출)

| # | 요구 | 구현 수단 |
|---|---|---|
| G1 | 여러 사용자가 동시에 같은 계획을 요청해도 **한 명만 확정** | DynamoDB 조건부 갱신 (TransactWriteItems) |
| G2 | 같은 완료 요청을 재시도해도 **방문 기록·수신함 알림은 한 건** | 결정적 키(`claimId`) + `attribute_not_exists` 조건 + 멱등 응답 |
| G3 | 실행자가 포기한 뒤 들어온 **이전 요청은 거부** | 모든 쓰기에 `plan.currentClaimId = :claimId` 조건 |
| G4 | 완료 상태·방문 기록·수신함 항목은 **하나의 트랜잭션**으로 저장 | TransactWriteItems |
| G5 | 계획별 작성자/실행자 **권한 검사** | Cognito JWT → Lambda에서 `sub` 비교 |
| G6 | AI 초안은 검증 후 편집 화면에 반환, **사용자가 확인한 카드만 저장** | Bedrock 응답 스키마 검증, 저장은 `POST /plans`에서만 |
| G7 | 사진은 비동기 처리, **위치 메타데이터 제거**, 실패 시 글로 완료 가능 | S3 → SQS → Lambda, DLQ |
| G8 | 오류·처리 시간 추적, SAM으로 새 스택 배포 후 검증 | CloudWatch, AWS SAM |

## 3. MVP 범위

### 포함 (P0)
- Cognito 인증 (이메일 + 비밀번호, 닉네임 속성). 시범 운영 편의를 위해 PreSignUp Lambda로 `autoConfirmUser`/`autoVerifyEmail = true` (인증 코드 단계 없음)
- 계획 생성/조회/목록, AI 하루 카드 초안 생성
- 이어받기(인수) 확정, 포기, 기한 관리
- 방문 기록 저장, 사진 업로드 URL 발급, 사진 비동기 처리, 완료(트랜잭션)
- 수신함 목록/상세/읽음, 한 줄 답장
- 측정용 데이터 저장 (작성 시간, AI 충실도, 기한 내 완료, 답장 여부)
- SAM 템플릿 (API, Lambda, DynamoDB, S3, SQS, Cognito, CloudFront, 알람)
- 검증 테스트: 비인가 접근, 동시 인수, 중복 완료, 사진 실패 복구

### 이후 (P1)
- 실행자가 계획자의 답장 확인 (`GET /me/claims` 응답에 `reply` 포함)
- 계획 대표 사진 (같은 사진 파이프라인 재사용)
- AI 생성 사용자별 일일 호출 제한

### 제외
- 지도/위치 기반 검색, 여러 동네 (MVP는 동네 한 곳 고정, `areaId` 필드만 예약)
- 푸시/이메일 알림, 문화공간 운영자 기능, 신고·모더레이션, 다중 사진

## 4. 아키텍처

```
브라우저(React) ──▶ CloudFront ──▶ S3 (비공개, 정적 파일, OAC)
     │
     ├─ Cognito User Pool (로그인 → ID 토큰)
     │
     └─▶ API Gateway (HTTP API, JWT authorizer)
            └─▶ API Lambda들 ──▶ DynamoDB (단일 테이블)
                    ├─▶ Amazon Bedrock (AI 하루 카드)
                    └─▶ S3 사진 버킷 presigned PUT/GET URL 발급

사진: 브라우저 ──PUT──▶ S3 photos/original/ ──이벤트──▶ SQS ──▶ 사진 처리 Lambda
                                                        │        ├ 검사·축소·메타데이터 제거 → photos/processed/
                                                        │        └ DynamoDB 사진 상태 갱신 + 완료 트랜잭션
                                                        └─(maxReceiveCount 3 초과)─▶ SQS DLQ ──▶ CloudWatch 알람

CloudWatch Logs/Metrics/Alarms, 배포: AWS SAM
```

### 권장 기술 스택
- 런타임: **Python 3.12** (boto3, Pillow 레이어, pydantic으로 입력/AI 응답 검증) — 팀 합의 시 Node.js 20(sharp)로 변경 가능
- IaC: AWS SAM (`template.yaml`, `samconfig.toml`에 `dev`/`demo` 환경)
- 리전: `ap-northeast-2` (Bedrock 모델 가용 리전이 다르면 Bedrock 클라이언트만 별도 리전 지정)

### Lambda 구성 (제안)

| 함수 | 트리거 | 역할 |
|---|---|---|
| `MeFunction` | `GET /me`, `GET /me/claims`, `GET /me/plans` | 내 정보, 내 하루 |
| `PlansFunction` | `/plans*` | 계획 생성·조회·목록, 이어받기 |
| `AiCardFunction` | `POST /ai/day-card` | Bedrock 호출, 응답 검증 (타임아웃 25s) |
| `ClaimsFunction` | `/claims/*` | 포기, 방문 기록 저장, 업로드 URL, 완료 |
| `InboxFunction` | `/inbox*` | 수신함, 읽음, 답장 |
| `PhotoProcessorFunction` | SQS | 사진 검사·축소·메타데이터 제거, 완료 트랜잭션 |

## 5. 도메인 모델

### 상태

| 엔티티 | 상태 | 의미 |
|---|---|---|
| Plan | `OPEN` | 이어받기 가능 |
| | `CLAIMED` | 누군가 이어받음 (기한 내) |
| | `COMPLETED` | 방문 기록이 수신함에 전달됨 |
| Claim | `ACTIVE` → `COMPLETED` / `ABANDONED` / `EXPIRED` | 한 번의 이어받기 시도 |
| Record.photoStatus | `NONE` / `UPLOAD_PENDING` / `PROCESSING` / `READY` / `FAILED` / `DISCARDED` | 사진 처리 상태 |

```
Plan:  OPEN ──claim──▶ CLAIMED ──complete──▶ COMPLETED
         ▲               │
         └──abandon──────┘
         └──(dueAt 경과 후 다른 사용자가 claim 시 이전 Claim은 EXPIRED)
```

기한: `dueAt = claimedAt + CLAIM_DURATION_DAYS(기본 3일)`. 별도 스케줄러 없이 **지연 해제** 방식으로 처리한다. 기한이 지난 `CLAIMED` 계획은 새 이어받기 요청이 들어오면 인수 가능하며, 그 전까지는 기존 실행자가 늦게라도 완료할 수 있다(`late = true`로 기록). 둘러보기 목록에는 `OPEN`과 기한 지난 `CLAIMED`를 함께 "이어받기 가능"으로 노출한다.

### Plan (하루 계획)
```json
{
  "planId": "pln_01H...",
  "areaId": "default",
  "owner": { "userId": "cognito-sub", "nickname": "지연" },
  "title": "이사 전, 마지막 동네 산책",
  "summary": "매일 지나치던 빵집에 들르고, 하천 벤치에 잠시 앉고 싶었어요.",
  "reason": "이삿짐 정리가 길어져 5년간 살던 동네에서 마지막 산책을 하지 못했어요.",
  "estimatedMinutes": 45,
  "places": [
    { "order": 1, "name": "골목 빵집", "activity": "처음 들러 소금빵 사기", "whyChosen": "출근길에 매일 지나쳤지만 한 번도 못 들어갔어요." },
    { "order": 2, "name": "하천 산책로 벤치", "activity": "벤치에 잠시 앉아 물소리 듣기", "whyChosen": "저녁마다 걸었던 길에서 마지막으로 쉬고 싶었어요." }
  ],
  "question": "벤치에 앉았을 때 어떤 소리가 들렸나요?",
  "status": "OPEN",
  "currentClaimId": null,
  "createdAt": "2026-11-17T09:00:00Z",
  "completedAt": null,
  "metrics": { "authoringMode": "AI_EDITED", "durationMs": 183000, "addedNotInSource": 0, "aiFlaggedInDraft": 1 }
}
```

검증 규칙: `title` 1–40자, `summary` 1–120자, `reason` 1–300자, `estimatedMinutes` 10–240, `places` 1–3개(`name` 1–30자, `activity` 1–60자, `whyChosen` 0–120자), `question` 1–80자(하루 카드의 방문 질문은 정확히 1개, 필수).

필드 쓰임 (시안 기준)
- `summary` — 하고 싶었던 일 한두 문장. 둘러보기 카드 본문, 돌아온 하루의 "내가 남긴 계획" 칸.
- `reason` — 취소한 이유. 하루 상세의 "{계획자} · 취소한 이유" 박스.
- `question` — 방문 기록 작성 화면에서 다시 보여주는 질문.
- `metrics` — 측정용. API 응답에서는 소유자에게만 포함한다.

### Claim (이어받기)
`claimId`, `planId`, `claimer {userId, nickname}`, `status`, `claimedAt`, `dueAt`, `completedAt`, `late`.

### Record (방문 기록) — Claim당 1개, 키는 `claimId`
`claimId`, `planId`, `authorId`, `letter`(1–1000자), `photoStatus`, `photoKey`(현재 유효한 원본 키), `processedKey`, `photoError`, `updatedAt`, `deliveredAt`.

### InboxItem (수신함) — Claim당 최대 1개, `itemId = claimId`
`ownerId`, `itemId`, `planId`, `planTitle`, `claimerNickname`, `hasPhoto`, `read`, `createdAt`, `reply {text, sentAt} | null`.

> 저장 시 `reply` 속성은 **쓰지 않는다**(DynamoDB `NULL` 타입으로 저장하면 `attribute_not_exists(reply)`가 항상 거짓이 된다). API 응답에서만 없으면 `null`로 직렬화한다.

## 6. DynamoDB 설계 (단일 테이블 `HaruTable`)

| 엔티티 | PK | SK | GSI1PK / GSI1SK | GSI2PK / GSI2SK |
|---|---|---|---|---|
| Plan | `PLAN#{planId}` | `META` | `AREA#{areaId}#STATUS#{status}` / `{createdAt}` | `USER#{ownerId}` / `PLAN#{createdAt}` |
| Claim | `PLAN#{planId}` | `CLAIM#{claimId}` | — | `USER#{claimerId}` / `CLAIM#{claimedAt}` |
| Record | `PLAN#{planId}` | `RECORD#{claimId}` | — | — |
| ClaimLookup | `CLAIM#{claimId}` | `META` | — | — |
| InboxItem | `USER#{ownerId}` | `INBOX#{claimId}` | — | — |
| AiDraft | `AIDRAFT#{draftId}` | `META` (TTL 7일) | — | — |
| IdempotencyKey | `IDEM#{userId}#{key}` | `META` (TTL 24시간) | — | — |

- `ClaimLookup`은 `claimId`만으로 `planId`를 찾기 위한 항목 (`/claims/{claimId}/*` 경로용).
- 계획 `status`가 바뀔 때 `GSI1PK`도 함께 갱신한다.
- 모든 Plan 갱신은 `version` 속성을 1씩 올린다(디버깅·낙관적 잠금용).
- 수신함은 사용자당 항목이 적으므로 `Query` 후 Lambda에서 `createdAt` 역순 정렬.

### 핵심 트랜잭션

**공통: 트랜잭션 실패 처리**
1. `TransactionCanceledException`이면 `CancellationReasons`를 먼저 본다.
2. `TransactionConflict`(동시 트랜잭션 충돌)가 하나라도 있으면 트랜잭션 전체를 지수 백오프(50·100·200ms, 최대 3회)로 재시도한다.
3. 모든 실패 사유가 `ConditionalCheckFailed`일 때만 아래 각 트랜잭션의 "조건 실패 시" 분기로 간다. 분기용 재조회는 반드시 `ConsistentRead=True`.
4. 재시도를 다 써도 판정할 수 없으면 503 `TRY_AGAIN`.

**이어받기 (G1)** — `POST /plans/{planId}/claims`
```
TransactWriteItems:
  Update PLAN#{planId}/META
    Condition: ownerId <> :me AND
               (status = 'OPEN' OR (status = 'CLAIMED' AND dueAt < :now))
    Set: status='CLAIMED', currentClaimId=:claimId, claimerId=:me, dueAt=:dueAt,
         GSI1PK=AREA#..#STATUS#CLAIMED, version = version + 1
  Put  PLAN#{planId}/CLAIM#{claimId}        Condition: attribute_not_exists(PK)
  Put  CLAIM#{claimId}/META                  Condition: attribute_not_exists(PK)
```
- 조건 실패 시 계획을 다시 읽어 분기: `claimerId = :me`이고 Claim이 ACTIVE면 **기존 Claim을 200으로 반환**(중복 클릭 멱등), 소유자면 `CANNOT_CLAIM_OWN_PLAN`, 그 외 `ALREADY_CLAIMED`(409).
- 기한 지난 Claim을 대체한 경우 이전 Claim을 `EXPIRED`로 표시(같은 트랜잭션에 `Update` 추가, 조건 `status = 'ACTIVE'`).

**포기 (G3)** — `POST /claims/{claimId}/abandon`
```
Update PLAN META   Condition: currentClaimId = :claimId AND status = 'CLAIMED' AND claimerId = :me
                   Set: status='OPEN', GSI1PK=AREA#..#STATUS#OPEN, version = version + 1
                   REMOVE currentClaimId, claimerId, dueAt
Update CLAIM       Condition: status = 'ACTIVE'   Set: status='ABANDONED'
Update RECORD(있으면) Set: photoStatus='DISCARDED'   (Record가 없으면 이 항목은 트랜잭션에서 뺀다)
```
실패 시 `CLAIM_NOT_ACTIVE`(409).

**완료 (G2, G3, G4)** — `POST /claims/{claimId}/complete` 와 사진 처리 Lambda가 공통 함수 `complete_claim(claimId, mode)`를 사용
```
TransactWriteItems:
  Update PLAN META   Condition: currentClaimId = :claimId AND status = 'CLAIMED'
                     Set: status='COMPLETED', completedAt, GSI1PK=..#STATUS#COMPLETED
  Update CLAIM       Condition: status = 'ACTIVE'
                     Set: status='COMPLETED', completedAt, late=(now > dueAt)
  Update RECORD      Condition: attribute_exists(letter)
                                (WITH_PHOTO일 때 추가: AND photoKey = :key AND photoStatus = 'PROCESSING')
                     Set: deliveredAt, photoStatus = 아래 규칙
                       - mode = WITH_PHOTO → 'READY' + processedKey (사진 처리 Lambda가 넘긴 값)
                       - mode = TEXT_ONLY, 사진이 없었음 → 'NONE'
                       - mode = TEXT_ONLY, 사진이 UPLOAD_PENDING/PROCESSING/FAILED → 'DISCARDED'
  Put    USER#{ownerId}/INBOX#{claimId}
                     Condition: attribute_not_exists(PK)
```
- 조건 실패 시 Claim을 다시 읽어 분기:
  - `COMPLETED` → **이미 완료된 결과를 200으로 반환** (중복 완료 멱등, `CompleteDuplicate` 메트릭 +1)
  - `ABANDONED` / `EXPIRED` 또는 currentClaimId 불일치 → `CLAIM_NOT_ACTIVE`(409)
  - 기록 없음 → `RECORD_REQUIRED`(400)
- 사진 처리 Lambda가 완료하려는 시점에 이미 글로 완료되어 있으면 아무것도 하지 않고 종료한다(사진은 버리고 처리본 삭제).

## 7. API 계약

- Base: `https://{api-id}.execute-api.{region}.amazonaws.com` (HTTP API, 스테이지 `$default`)
- 인증: 모든 경로에 `Authorization: Bearer {Cognito ID token}` 필수. HTTP API JWT authorizer의 audience는 User Pool App Client ID. 사용자 ID는 JWT `sub`, 닉네임은 ID 토큰의 `nickname` 클레임(액세스 토큰에는 없으므로 반드시 ID 토큰 사용).
- API CORS: 허용 Origin = CloudFront 도메인, `http://localhost:5173` / 허용 헤더 = `Authorization`, `Content-Type`, `Idempotency-Key` / 허용 메서드 = `GET`, `POST`, `PUT`, `OPTIONS`.
- 에러 형식: `{ "error": { "code": "ALREADY_CLAIMED", "message": "이미 다른 사람이 이어받았어요.", "details": { "places[0].name": "1–30자로 입력해 주세요." } } }` — `details`는 `VALIDATION_ERROR`일 때만, 키는 요청 본문 경로(`flags[].path`와 같은 표기).
- 시간은 모두 ISO 8601 UTC 문자열.

### 7.1 내 정보
**`GET /me`** → 200
```json
{ "userId": "...", "nickname": "수현", "unreadInboxCount": 1 }
```

**`GET /me/claims?status=ACTIVE|COMPLETED`** → 200 `{ "items": [ { "claim": Claim, "plan": Plan, "record": Record|null } ] }`

**`GET /me/plans`** (P1) → 200 `{ "items": [Plan] }`

### 7.2 AI 하루 카드
**`POST /ai/day-card`**
```json
// request
{ "story": "이사 전에 5년 살던 동네를 마지막으로 걷고 싶었어요. 출근길 빵집에 들러 빵을 사고, 저녁마다 걷던 하천 벤치에 앉아 쉬려고 했는데 이삿짐 정리가 길어져서 못 갔어요." }
// 200
{
  "draftId": "drf_01H...",
  "card": {
    "title": "이사 전, 마지막 동네 산책",
    "summary": "출근길 빵집에 들르고, 저녁마다 걷던 하천 벤치에 앉아 쉬고 싶었어요.",
    "reason": "이삿짐 정리가 길어져 5년간 살던 동네에서 마지막 산책을 하지 못했어요.",
    "estimatedMinutes": null,
    "places": [
      { "name": "골목 빵집", "activity": "들러서 빵 사기", "whyChosen": "출근길에 지나치던 곳이에요." },
      { "name": "하천 산책로 벤치", "activity": "벤치에 앉아 쉬기", "whyChosen": "저녁마다 걷던 길이에요." }
    ],
    "question": "하천 벤치에 앉았을 때 어떤 풍경이 보였나요?"
  },
  "flags": [
    { "path": "places[0].name", "type": "NOT_IN_SOURCE", "value": "골목 빵집" },
    { "path": "estimatedMinutes", "type": "MISSING" }
  ]
}
```
- `card`의 필드 이름은 `POST /plans` 요청과 **동일**하다(`places[].name/activity/whyChosen`). 프론트는 `card`를 그대로 폼 초기값으로 쓰고, `flags[].path`도 같은 경로로 폼 필드에 연결한다.
- 입력: `story` 1–1000자.
- Bedrock 호출: 모델 ID는 환경 변수 `BEDROCK_MODEL_ID`(팀에서 리전 가용 모델 확인 후 지정, 경량 모델 권장). 도구 사용(tool use) 또는 JSON 출력 지시로 위 `card` 스키마를 강제한다.
- 프롬프트 원칙: 원문에 있는 장소·행동만 사용, 없는 값은 `null`, 공개 장소만, 개인 정보 제거, 방문 질문은 정확히 1개.
- 검증: pydantic으로 스키마·길이 검증 → 실패 시 1회 재시도 → 그래도 실패면 `AI_INVALID_OUTPUT`(502).
- 시간 제한: Lambda 타임아웃 25s(HTTP API 통합 상한 30s). Bedrock 클라이언트는 `Config(connect_timeout=3, read_timeout=10, retries={'max_attempts': 1})`로 만든다. `context.get_remaining_time_in_millis()`가 12초 미만이면 검증 실패 재시도를 생략하고 `AI_INVALID_OUTPUT`을 반환한다. 이렇게 해서 모든 실패가 Lambda 강제 종료(→ 코드 없는 500) 전에 정해진 에러 코드로 응답되게 한다.
- 충실도 플래그: 정규화(공백·조사 제거)한 `places[].name`/`places[].activity`의 핵심 명사가 원문에 없으면 `NOT_IN_SOURCE`, `null` 필드는 `MISSING`. 같은 판정 함수 `find_not_in_source(story, places)`를 `POST /plans`의 지표 계산에도 쓴다.
- Bedrock 오류/타임아웃 → `AI_UNAVAILABLE`(503). 프론트는 사연을 유지한 채 직접 작성으로 전환한다.
- 원문, 원본 AI 응답, 플래그를 `AiDraft`로 저장(TTL 7일). 이 시점에는 계획을 저장하지 않는다.

### 7.3 계획
**`POST /plans`** — 헤더 `Idempotency-Key: {uuid}` 필수
```json
// request
{
  "title": "...", "summary": "...", "reason": "...", "estimatedMinutes": 45,
  "places": [ { "name": "...", "activity": "...", "whyChosen": "..." } ],
  "question": "...",
  "authoring": { "mode": "AI_EDITED", "durationMs": 183000, "aiDraftId": "drf_01H...", "story": "사연 원문" }
}
// 201
{ "plan": Plan }
```
- 같은 `Idempotency-Key`와 같은 본문으로 재요청하면 처음 생성한 Plan을 그대로 반환. 본문이 다르면 409 `IDEMPOTENCY_CONFLICT`.
- `authoring.story`는 **두 모드 모두 필수**(1–1000자). 직접 작성(`MANUAL`)도 사연 입력 화면을 거치므로 원문이 있다.
- 지표 계산 (모드와 무관): `find_not_in_source(story, 최종 places)` 결과 개수를 `metrics.addedNotInSource`로 저장. `AI_EDITED`이면 `aiDraftId` 초안의 `NOT_IN_SOURCE` 플래그 수를 `metrics.aiFlaggedInDraft`로 함께 저장.

**`GET /plans?status=OPEN&limit=20&cursor=...`** → 200 `{ "items": [PlanSummary], "nextCursor": null }`
- `PlanSummary`: `planId, title, summary, estimatedMinutes, placeCount, status, ownerNickname, createdAt`
- 기한 지난 `CLAIMED`도 `status: "OPEN"`으로 내려준다. 구현: `GSI1PK = AREA#{areaId}#STATUS#OPEN` 조회 + `STATUS#CLAIMED` 조회 후 `dueAt < now` 필터를 합쳐 `createdAt` 역순 정렬 (MVP 데이터 규모에서는 충분).

**`GET /plans/{planId}`** → 200
```json
{ "plan": Plan, "isOwner": false, "myClaim": Claim | null, "claimable": true }
```
- `COMPLETED` 계획은 소유자와 해당 실행자만 조회 가능, 그 외 `NOT_FOUND`(404).
- 상태 표시 규칙 (목록과 동일):
  - 기한 지난 `CLAIMED`는 현재 실행자가 아닌 요청자에게 `status: "OPEN"`으로 내려준다. 현재 실행자 본인에게는 `CLAIMED` + `myClaim`.
  - 다른 사람이 기한 내 이어받은 상태면 `status: "CLAIMED"`, 실행자 정보는 숨긴다.
- `myClaim`은 요청자의 Claim 중 `ACTIVE` 또는 `COMPLETED`인 것만 담는다. 포기·만료된 Claim만 있으면 `null`(다시 이어받을 수 있음).
- `claimable = !isOwner && myClaim == null && (status == OPEN 또는 기한 지난 CLAIMED)`. 프론트는 이어받기 버튼 노출을 이 값으로 판단한다.

**`POST /plans/{planId}/claims`** → 201 `{ "claim": Claim }` / 200 `{ "claim": Claim }`(본인 중복 요청 — 기존 Claim 반환) / 409 `ALREADY_CLAIMED` | `CANNOT_CLAIM_OWN_PLAN`

### 7.4 이어받기·방문 기록
**`POST /claims/{claimId}/abandon`** → 200 `{ "claim": Claim }` / 409 `CLAIM_NOT_ACTIVE` / 403

**`PUT /claims/{claimId}/record`**
```json
// request
{ "letter": "말씀하신 빵집에서 소금빵을 사서 ...", "photo": { "contentType": "image/jpeg", "size": 2480000 } }   // photo는 선택
// 200
{
  "record": Record,
  "upload": { "url": "https://...presigned...", "method": "PUT", "headers": { "Content-Type": "image/jpeg" }, "expiresAt": "..." } | null
}
```
- 조건: 요청자 = claimer, Claim `ACTIVE`, Plan `currentClaimId = claimId`. 아니면 403 / 409 `CLAIM_NOT_ACTIVE`.
- 글은 여러 번 덮어쓸 수 있다(완료 전까지).
- `photo`가 있으면 `photoKey = photos/original/{claimId}/{uuid}.{ext}`, `photoStatus = UPLOAD_PENDING`. presigned PUT URL 유효 10분.
  - 요청의 `size`가 10MB를 넘거나 `contentType`이 `image/jpeg`/`image/png`가 아니면 400 `VALIDATION_ERROR`.
  - presigned PUT은 크기 범위를 강제할 수 없으므로 `ContentType`과 정확한 `ContentLength`(= 요청의 `size`)를 서명에 포함하고, 사진 처리 Lambda가 실제 객체 크기를 다시 검사한다.
- 사진 없이 요청하면(글로 완료 경로) 진행 중이던 사진은 그대로 두고 글만 갱신한다. 사진 폐기는 `complete(TEXT_ONLY)` 시점에 한다.

**`POST /claims/{claimId}/record/photo`** — 사진 교체. 요청 `{ "contentType", "size" }`, 응답은 위와 같은 `upload`. 새 `photoKey`로 교체하고 `photoStatus = UPLOAD_PENDING`. 이전 키에서 오는 처리 결과는 무시된다.

**`GET /claims/{claimId}/record`** → 200
```json
{ "record": Record, "claimStatus": "ACTIVE", "delivered": false, "photoStatus": "PROCESSING" }
```
- 요청자 = claimer만 조회 가능. 기록이 아직 없으면 200 `{ "record": null, "claimStatus": "ACTIVE", "delivered": false, "photoStatus": "NONE" }`.
- 포기·만료된 Claim도 조회는 가능하다. 프론트는 **`claimStatus`** 가 `ABANDONED`/`EXPIRED`이면 종료 안내를 띄운다(`photoStatus = DISCARDED`는 글로 완료 성공 시에도 나오므로 종료 판단에 쓰지 않는다).

**`POST /claims/{claimId}/complete`** — 요청자 = claimer가 아니면 403 `FORBIDDEN` (트랜잭션 전 `ClaimLookup`/Claim으로 확인. 사진 처리 Lambda 경로는 이 검사를 생략)
```json
// request
{ "mode": "TEXT_ONLY" }
// 200 (처음 완료 또는 이미 완료된 경우 모두)
{ "claim": Claim, "delivered": true, "hasPhoto": false }
```
- 사진 포함 완료는 사진 처리 Lambda가 서버 측에서 수행하므로, 클라이언트가 호출하는 완료는 `TEXT_ONLY`뿐이다.
- 사진이 처리 중이거나 처리 끝났지만 미전달(`READY` & `deliveredAt` 없음)일 때 `TEXT_ONLY`로 완료하면 `photoStatus = DISCARDED`.
- 이미 사진과 함께 완료된 뒤 들어온 `TEXT_ONLY` 요청은 멱등 응답(200, `hasPhoto: true`)을 돌려준다. 프론트는 완료 화면 문구를 응답의 `hasPhoto`로 정한다.
- Record가 없으면 400 `RECORD_REQUIRED`. 프론트는 항상 `PUT /record`로 글을 저장한 뒤 호출한다.

### 7.5 수신함
**`GET /inbox`** → 200 `{ "items": [InboxItem] }` (최신순)

**`GET /inbox/{itemId}`** → 200
```json
{
  "item": InboxItem,
  "plan": Plan,
  "record": { "letter": "...", "deliveredAt": "...", "photoUrl": "https://...presigned GET (15분)..." | null, "photoCaption": "하천 산책로 벤치에서" | null }
}
```
- `photoCaption`은 저장하지 않고 응답 시 만든다: 사진이 있으면 `plan.places`의 마지막 장소 이름 + "에서", 없으면 `null`.
- `ownerId = 요청자`만 조회 가능. 아니면 404.

**`POST /inbox/{itemId}/read`** → 204. `read = true`.

**`POST /inbox/{itemId}/reply`** → 201 `{ "reply": { "text": "...", "sentAt": "..." } }`
- `text` 1–100자. 조건 `ownerId = :me AND attribute_not_exists(reply)` → 이미 있으면 409 `REPLY_ALREADY_SENT` (5장 InboxItem 메모 참고: `reply`는 생성 시 쓰지 않는다).

### 7.6 에러 코드

| HTTP | code | 상황 |
|---|---|---|
| 400 | `VALIDATION_ERROR` | 입력 형식/길이 오류 (`details`에 필드별 사유) |
| 400 | `RECORD_REQUIRED` | 기록 없이 완료 요청 |
| 401 | `UNAUTHORIZED` | 토큰 없음/만료 (API Gateway authorizer) |
| 403 | `FORBIDDEN` | 남의 Claim/Record에 쓰기 시도 |
| 404 | `NOT_FOUND` | 없거나 볼 권한이 없는 리소스 |
| 409 | `ALREADY_CLAIMED` | 다른 사람이 먼저 이어받음 |
| 409 | `CANNOT_CLAIM_OWN_PLAN` | 자기 계획 이어받기 |
| 409 | `CLAIM_NOT_ACTIVE` | 포기/만료/완료된 Claim으로 쓰기 |
| 409 | `REPLY_ALREADY_SENT` | 답장 중복 |
| 409 | `IDEMPOTENCY_CONFLICT` | 같은 키로 다른 본문 요청 |
| 503 | `TRY_AGAIN` | 트랜잭션 충돌 재시도를 모두 소진 |
| 502 | `AI_INVALID_OUTPUT` | AI 응답 검증 실패 |
| 503 | `AI_UNAVAILABLE` | Bedrock 오류/타임아웃 |

## 8. 사진 처리 파이프라인

1. `PUT /record`(또는 `/record/photo`)가 presigned PUT URL 발급 → 브라우저가 `photos/original/...`에 업로드.
2. S3 `ObjectCreated` 이벤트(prefix `photos/original/`) → SQS `PhotoQueue` (visibility timeout 6×Lambda timeout, `maxReceiveCount` 3 → `PhotoDLQ`).
3. `PhotoProcessorFunction` (batch size 1, timeout 30s, 메모리 1024MB):
   0. 메시지 본문에 `Records`가 없거나 `Event == "s3:TestEvent"`(알림 설정 시 S3가 보내는 테스트 메시지)이면 정상 종료. S3 키는 `unquote_plus`로 디코딩해서 쓴다.
   1. 키에서 `claimId` 추출 → Record를 `ConsistentRead`로 조회.
      - `deliveredAt`이 있거나, `photoStatus = DISCARDED`이거나, `photoKey`가 이 키와 다르면 **무시하고 종료**(처리본이 있으면 삭제).
      - `photoStatus = READY`이고 `deliveredAt`이 없으면(이전 시도에서 완료 트랜잭션만 실패) 처리를 건너뛰고 6단계로.
   2. 조건부 갱신 `photoStatus = PROCESSING` (Condition: `photoKey = :key AND photoStatus IN ('UPLOAD_PENDING','PROCESSING') AND attribute_not_exists(deliveredAt)`). 조건 실패 → 무시하고 종료.
   3. 매직 바이트로 JPEG/PNG 확인, 실제 객체 크기 ≤ 10MB, 최대 해상도 검사. 실패 → 조건부로 `FAILED` + `photoError`(재시도하지 않음).
   4. EXIF 방향 반영 후 긴 변 1600px로 축소, **EXIF/GPS 포함 모든 메타데이터 제거**하여 JPEG로 재인코딩 → `photos/processed/{claimId}/{uuid}.jpg`.
   5. `complete_claim(claimId, WITH_PHOTO, key, processedKey)` 호출. 별도의 READY 쓰기 없이 완료 트랜잭션의 Record Update가 `photoStatus = READY`, `processedKey`를 함께 설정한다(6장 조건 참고). 사진 교체·글로 완료와 경합하면 조건 실패로 끝나고, 처리본은 삭제한다.
   6. (재시도 경로) `complete_claim`만 다시 호출. Claim이 이미 `ABANDONED`/`EXPIRED`면 종료.
   7. 일시 오류(S3/DynamoDB/트랜잭션 충돌)는 예외를 던져 SQS 재시도. `ApproximateReceiveCount >= 3`이면 예외를 던지기 전에 조건부로 `photoStatus = FAILED`, `photoError` 기록 → 메시지는 DLQ로 이동.
4. `PhotoDLQ` 메시지 수 > 0이면 CloudWatch 알람. 운영자는 DLQ에서 원인을 확인한다.
5. 사용자는 `FAILED` 상태에서 사진 교체 또는 `TEXT_ONLY` 완료 가능.
6. 원본은 S3 수명 주기로 7일 후 삭제. 버킷은 퍼블릭 차단, 접근은 presigned URL로만.
7. 사진 버킷 CORS: API와 같은 Origin에 `PUT`, `GET` 메서드와 `Content-Type` 헤더 허용 (브라우저가 presigned URL로 직접 업로드하므로 필수).

SAM 주의: 버킷 알림(QueueConfiguration)은 SQS 큐 정책이 먼저 있어야 생성된다. 큐 정책에서 `!GetAtt PhotoBucket.Arn`을 참조하면 순환 의존성이 생기므로, 버킷 이름을 `!Sub '${AWS::StackName}-photos-${AWS::AccountId}'`로 고정하고 큐 정책의 `aws:SourceArn`도 같은 문자열로 쓴 뒤 버킷에 `DependsOn: PhotoQueuePolicy`를 지정한다.

## 9. 권한 규칙

| 리소스 | 읽기 | 쓰기 |
|---|---|---|
| Plan (OPEN/CLAIMED) | 로그인 사용자 전체 | 생성: 로그인 사용자 / 수정·삭제: MVP 미지원 |
| Plan (COMPLETED) | 소유자, 해당 실행자 | — |
| Claim | 해당 실행자 | 생성: 소유자 외 사용자 / 포기: 해당 실행자 |
| Record | 해당 실행자, 전달 후 계획 소유자 | 해당 실행자 (Claim ACTIVE일 때만) |
| InboxItem / Reply | 계획 소유자 | 읽음·답장: 계획 소유자 |
| 사진 GET URL | 해당 실행자, 전달 후 계획 소유자 | — |

모든 판단은 Lambda에서 JWT `sub`와 저장된 `ownerId`/`claimerId` 비교로 한다. 클라이언트가 보낸 사용자 ID는 신뢰하지 않는다.

## 10. 관측과 측정

### 운영 지표 (CloudWatch, EMF 로그)
- `ClaimConflict` (409 ALREADY_CLAIMED 수), `CompleteDuplicate` (멱등 응답 수), `StaleRequestRejected` (CLAIM_NOT_ACTIVE 수)
- `PhotoProcessingMs`, `PhotoFailed`, `PhotoDLQDepth`(알람)
- `AiLatencyMs`, `AiFailure`, `AiInvalidOutput`
- API Gateway 5xx, Lambda Errors/Duration 알람

### 시범 운영 지표 (DynamoDB에 저장된 값으로 집계, `scripts/report.py`)
| 지표 | 목표 | 계산 |
|---|---|---|
| 기한 내 방문 기록 | 최초 확정 5건 중 3건 이상 | 처음 5개 Claim(ABANDONED 제외 여부는 팀 결정) 중 `COMPLETED && !late` |
| 답장률 | 완료 건 중 2건 이상 | `InboxItem.reply != null` |
| 작성 시간 비교 | — | `metrics.authoringMode`별 `durationMs` 중앙값 |
| 원문에 없던 장소·활동 추가 건수 | — | 모드별 `metrics.addedNotInSource` 합계/평균 비교 (+ `AI_EDITED`는 `aiFlaggedInDraft` 대비 잔존 수) |

## 11. 저장소 구조 (제안)

```
template.yaml               # SAM: HttpApi, Cognito, DynamoDB, S3(웹·사진), CloudFront(OAC), SQS+DLQ, Lambda, 알람
samconfig.toml
src/
  common/                   # auth(JWT claims), ddb(트랜잭션 헬퍼), errors, responses, validation(pydantic), metrics
  handlers/                 # me.py, plans.py, ai_card.py, claims.py, inbox.py
  domain/                   # claim_service.py(claim/abandon/complete_claim), plan_service.py, inbox_service.py
  workers/photo_processor.py
  prompts/day_card.md       # Bedrock 프롬프트
layers/pillow/
tests/
  unit/
  integration/              # 배포된 스택 대상: authz, concurrency, idempotency, photo_failure
scripts/
  seed.py                   # 데모 계정 3개(계획자 지연, 실행자 수현·민준)·예시 계획 생성
  report.py                 # 시범 운영 지표 집계
```

SAM 출력값(`ApiUrl`, `UserPoolId`, `UserPoolClientId`, `WebBucketName`, `DistributionId`, `DistributionDomain`)은 프론트엔드 배포에 사용한다.

## 12. 검증 시나리오 (새 스택에서 실행)

| # | 시나리오 | 기대 결과 |
|---|---|---|
| T1 | 토큰 없이 모든 경로 호출 | 401 |
| T2 | 사용자 B가 사용자 A의 수신함/Record/Claim 접근 | 404 또는 403, 데이터 노출 없음 |
| T3 | 서로 다른 사용자 10명이 같은 계획에 동시에 이어받기 | 성공 정확히 1건, 나머지 409 `ALREADY_CLAIMED`, Plan의 `currentClaimId`는 성공 Claim |
| T4 | 같은 사용자가 이어받기를 5회 동시 요청 | Claim 1건, 모두 같은 `claimId` 반환 |
| T5 | `complete`를 10회 동시/연속 호출 | 수신함 항목 1건, Record 1건, 응답은 모두 200 |
| T6 | 포기 후 이전 `claimId`로 기록 저장·완료 | 409 `CLAIM_NOT_ACTIVE`, 수신함 항목 없음 |
| T7 | 포기 → 다른 사용자가 이어받기 → 첫 사용자의 늦은 완료 요청 | 409, 두 번째 사용자의 완료는 정상 |
| T8 | 손상된 이미지 업로드 | `photoStatus = FAILED`, 글 유지, `TEXT_ONLY` 완료 가능 |
| T9 | 사진 처리 Lambda 강제 예외 (환경 변수 `FAULT_INJECT=photo`) | 3회 재시도 후 DLQ 이동, `FAILED`, 알람 발생 |
| T10 | 사진 처리 중 `TEXT_ONLY` 완료 후 사진 처리 성공 | 수신함 1건(사진 없음), 처리 Lambda는 무시 |
| T11 | GPS EXIF가 있는 사진 업로드 | 처리본에 EXIF/GPS 없음 (`exiftool`로 확인) |
| T12 | Bedrock 권한 제거 또는 타임아웃 | 503 `AI_UNAVAILABLE`, 계획 저장 없음 |
| T13 | 같은 `Idempotency-Key`로 `POST /plans` 3회 | Plan 1건 |
| T14 | 기한 지난 Claim이 있는 계획을 다른 사용자가 이어받기 | 성공, 이전 Claim `EXPIRED` |
| T15 | 사진 처리 중 사진 교체 | 교체 전 사진은 전달되지 않고 교체한 사진으로 완료 |
| T16 | 포기 후 둘러보기 목록 조회 | 해당 계획이 OPEN으로 다시 노출 |
| T17 | 첫 답장 / 두 번째 답장 | 201 / 409 `REPLY_ALREADY_SENT` |

T3·T5는 `tests/integration`에서 `concurrent.futures`로 실제 API를 동시에 호출해 검증하고, 결과를 실행 결과 문서(데모 자료)에 첨부한다.

## 13. 완료 기준 (Definition of Done)

- [ ] `sam build && sam deploy --config-env demo`로 빈 계정에 스택이 한 번에 생성된다
- [ ] 7장의 모든 엔드포인트가 계약대로 응답한다 (프론트 MSW 목업과 응답 형태 일치)
- [ ] T1–T17 통과, T3·T5 결과 로그 보관
- [ ] 90초 시연 시나리오(지연 사연 입력 → AI 카드 → 공개 → 수현·민준 동시 이어받기 중 한 명만 확정 → 사진+기록 전송 → 지연 수신함·답장)가 배포 환경에서 동작
- [ ] CloudWatch 대시보드에 API 오류, 사진 처리 시간, AI 지연, DLQ 깊이 표시
- [ ] `scripts/seed.py`로 데모 계정·데이터 재생성 가능
- [ ] Bedrock 모델 ID, 기한 일수 등 설정값은 환경 변수/파라미터로 분리

## 14. 작업 순서 (제안)

1. SAM 골격: Cognito, HttpApi + JWT authorizer, DynamoDB, `GET /me`
2. 계획 생성/목록/상세, 멱등 키
3. 이어받기·포기·완료 트랜잭션 (`complete_claim` 공통화) + T3–T7 통합 테스트
4. 방문 기록 저장(글), 수신함·읽음·답장
5. 사진 버킷·SQS·DLQ·처리 Lambda, 업로드 URL, T8–T11
6. Bedrock AI 하루 카드, 충실도 플래그, 작성 시간 저장, T12
7. CloudFront + 웹 버킷, 알람·대시보드, seed/report 스크립트, 시연 리허설
