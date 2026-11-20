# 이어온 하루 — Backend 요구사항 명세서

| 항목 | 내용 |
|---|---|
| 프로젝트 | 취소된 하루를 양도합니다 (AWS Skill Training 1조) |
| 서비스 | 이어온 하루 (모바일 웹) |
| 대상 저장소 | `knu-aws-2026-001/backend` |
| 기준 문서 | [`MVP.md`](../MVP.md) (API 계약의 Source of Truth), [frontend `MVP.md`](https://github.com/knu-aws-2026-001/frontend/blob/main/MVP.md) |
| 범위 | MVP P0 |

이 문서는 `MVP.md`의 내용을 구현·검증 가능한 요구사항 단위(ID 부여)로 정리한 것이다. 세부 스키마와 트랜잭션 식은 `MVP.md`의 해당 절을 따른다. 두 문서가 다르면 `MVP.md`가 우선하며, 차이를 발견하면 이 문서를 고친다.

---

## 1. 목적과 범위

계획자가 남긴 산책 계획(공개 장소 1–3곳)을 **한 명의 실행자만** 이어받고, 실행자의 방문 기록(글 + 선택 사진 1장)이 **정확히 한 번** 계획자의 수신함에 도착하도록 처리하는 서버리스 API를 만든다.

### 1.1 포함 (P0)
- Cognito 인증, 계획 생성·조회·목록, AI 하루 카드 초안
- 이어받기·포기·기한(지연 해제), 방문 기록·사진 업로드 URL·사진 비동기 처리·완료 트랜잭션
- 수신함 목록·상세·읽음·한 줄 답장
- 측정 데이터 저장, SAM 단일 스택 배포, CloudWatch 지표·알람
- 검증 시나리오 T1–T17

### 1.2 이후 (P1, 이 명세의 필수 요건 아님)
- `GET /me/claims` 응답에 `reply` 포함, `GET /me/plans`
- 계획 대표 사진, AI 생성 사용자별 일일 호출 제한

### 1.3 제외
- 지도·위치 검색, 여러 동네(`areaId = "default"` 고정), 푸시/이메일 알림, 운영자 기능, 신고·모더레이션, 다중 사진

## 2. 용어

| 용어 | 뜻 |
|---|---|
| 계획자 (owner) | 하루(Plan)를 남긴 사용자 |
| 실행자 (claimer) | 하루를 이어받은 사용자 |
| Plan | 하루 계획. `OPEN` → `CLAIMED` → `COMPLETED` |
| Claim | 한 번의 이어받기 시도. `ACTIVE` → `COMPLETED` / `ABANDONED` / `EXPIRED` |
| Record | Claim당 1개인 방문 기록(글 + 사진 상태) |
| InboxItem | 계획자 수신함 항목. Claim당 최대 1개, `itemId = claimId` |
| 지연 해제 | 스케줄러 없이, 기한 지난 Claim을 새 이어받기 요청 시점에 `EXPIRED`로 바꾸는 방식 |

## 3. 시스템 구성

- API Gateway HTTP API + JWT authorizer(Cognito User Pool, audience = App Client ID)
- Lambda (Python 3.12): `MeFunction`, `PlansFunction`, `AiCardFunction`, `ClaimsFunction`, `InboxFunction`, `PhotoProcessorFunction`
- DynamoDB 단일 테이블 `HaruTable` (GSI1: 동네·상태별 목록, GSI2: 사용자별 계획·Claim)
- S3: 웹 버킷(비공개, CloudFront OAC), 사진 버킷(퍼블릭 차단, presigned URL 전용)
- SQS `PhotoQueue` + `PhotoDLQ`, Amazon Bedrock, CloudWatch, AWS SAM
- 리전 `ap-northeast-2` (Bedrock만 별도 리전 허용)

## 4. 기능 요구사항

표기: **[필수]** P0 완료 조건, 근거는 `MVP.md` 절 번호.

### 4.1 인증·사용자 (AUTH)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-AUTH-01 | Cognito User Pool은 이메일을 사용자 이름으로 쓰고, `nickname`을 필수 속성으로 받는다. App Client는 클라이언트 시크릿 없이 생성한다(브라우저 SDK 사용). | §3 |
| BE-AUTH-02 | PreSignUp Lambda가 `autoConfirmUser = true`, `autoVerifyEmail = true`를 설정해 인증 코드 단계를 없앤다. | §3 |
| BE-AUTH-03 | 모든 API 경로는 JWT authorizer로 보호한다. 토큰이 없거나 만료되면 401 `UNAUTHORIZED`. | §7, T1 |
| BE-AUTH-04 | 사용자 ID는 JWT `sub`, 닉네임은 ID 토큰의 `nickname` 클레임에서 읽는다. 요청 본문의 사용자 ID는 쓰지 않는다. | §7, §9 |
| BE-AUTH-05 | `GET /me`는 `userId`, `nickname`, `unreadInboxCount`를 반환한다. | §7.1 |

### 4.2 AI 하루 카드 (AI)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-AI-01 | `POST /ai/day-card`는 `story`(1–1,000자)를 받아 `draftId`, `card`, `flags`를 반환한다. `card` 필드 이름은 `POST /plans` 요청과 같다. | §7.2 |
| BE-AI-02 | Bedrock 모델 ID는 환경 변수 `BEDROCK_MODEL_ID`로 받는다. tool use 또는 JSON 출력 지시로 `card` 스키마를 강제한다. | §7.2 |
| BE-AI-03 | 프롬프트는 원문에 있는 장소·행동만 쓰고, 없는 값은 `null`, 공개 장소만, 개인 정보 제거, 방문 질문 정확히 1개를 지시한다. 프롬프트는 `src/prompts/day_card.md`에 둔다. | §7.2 |
| BE-AI-04 | 응답은 pydantic으로 검증하고, 실패하면 1회 재시도한다. 그래도 실패하면 502 `AI_INVALID_OUTPUT`. 남은 실행 시간이 12초 미만이면 재시도 없이 `AI_INVALID_OUTPUT`. | §7.2 |
| BE-AI-05 | Bedrock 오류·타임아웃은 503 `AI_UNAVAILABLE`. Bedrock 클라이언트는 `connect_timeout=3`, `read_timeout=10`, `max_attempts=1`, Lambda 타임아웃은 25초. 어떤 실패도 코드 없는 500으로 끝나지 않는다. | §7.2, T12 |
| BE-AI-06 | `flags`: `places[].name`/`activity`의 핵심 명사가 정규화한 원문에 없으면 `NOT_IN_SOURCE`, `null` 필드는 `MISSING`. 판정 함수 `find_not_in_source(story, places)`는 `POST /plans` 지표 계산과 공유한다. | §7.2 |
| BE-AI-07 | 원문·원본 AI 응답·플래그를 `AiDraft`(TTL 7일)로 저장한다. 이 시점에 Plan은 만들지 않는다. | §7.2, G6 |

### 4.3 계획 (PLAN)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-PLAN-01 | `POST /plans`는 `Idempotency-Key` 헤더가 필수다. 같은 키·같은 본문이면 처음 만든 Plan을 반환하고, 본문이 다르면 409 `IDEMPOTENCY_CONFLICT`. 키는 `IDEM#{userId}#{key}`(TTL 24시간)로 저장한다. | §7.3, T13 |
| BE-PLAN-02 | 입력 검증: `title` 1–40자, `summary` 1–120자, `reason` 1–300자, `estimatedMinutes` 10–240, `places` 1–3개(`name` 1–30, `activity` 1–60, `whyChosen` 0–120), `question` 1–80자, `authoring.story` 1–1,000자(두 모드 모두 필수). 실패 시 400 `VALIDATION_ERROR` + `details`(요청 본문 경로별 메시지). | §5, §7.3 |
| BE-PLAN-03 | `metrics.authoringMode`, `durationMs`, `addedNotInSource`(최종 places 기준 `find_not_in_source` 개수)를 저장한다. `AI_EDITED`이면 `aiDraftId` 초안의 `NOT_IN_SOURCE` 수를 `aiFlaggedInDraft`로 저장한다. | §7.3 |
| BE-PLAN-04 | `GET /plans?status=OPEN&limit&cursor`는 `PlanSummary` 목록을 `createdAt` 역순으로 반환한다. 기한 지난 `CLAIMED`도 `status: "OPEN"`으로 포함한다. | §7.3, T16 |
| BE-PLAN-05 | `GET /plans/{planId}`는 `plan`, `isOwner`, `myClaim`, `claimable`을 반환한다. `claimable = !isOwner && myClaim == null && (OPEN 또는 기한 지난 CLAIMED)`. | §7.3 |
| BE-PLAN-06 | 상세의 상태 표시: 기한 지난 `CLAIMED`는 현재 실행자가 아니면 `OPEN`, 현재 실행자 본인에게는 `CLAIMED` + `myClaim`. 다른 사람이 기한 내 이어받았으면 `CLAIMED`이고 실행자 정보는 응답에서 뺀다. | §7.3 |
| BE-PLAN-07 | `COMPLETED` 계획은 소유자와 해당 실행자만 조회한다. 그 외는 404 `NOT_FOUND`. | §7.3, §9 |
| BE-PLAN-08 | `metrics`는 소유자 응답에만 포함한다. 내부 속성(`claimerId`, `currentClaimId`, `dueAt`, `GSI*`, `version`)은 응답 직렬화 단계에서 허용 목록 방식으로 걸러낸다. | §5, §7.3 |
| BE-PLAN-09 | Plan 갱신 시 `version`을 1 올리고, `status`가 바뀌면 `GSI1PK`를 함께 바꾼다. | §6 |

### 4.4 이어받기·포기 (CLAIM)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-CLAIM-01 | `POST /plans/{planId}/claims`는 TransactWriteItems로 Plan 조건부 갱신(소유자 아님, `OPEN` 또는 기한 지난 `CLAIMED`)과 Claim·ClaimLookup 생성을 한 번에 처리한다. 성공 201. | §6, G1 |
| BE-CLAIM-02 | 동시 요청 중 정확히 하나만 성공하고, 나머지는 409 `ALREADY_CLAIMED`. | G1, T3 |
| BE-CLAIM-03 | 같은 사용자의 중복 요청은 기존 ACTIVE Claim을 200으로 반환한다(Claim 1건). | §6, T4 |
| BE-CLAIM-04 | 소유자가 요청하면 409 `CANNOT_CLAIM_OWN_PLAN`. | §6 |
| BE-CLAIM-05 | `dueAt = claimedAt + CLAIM_DURATION_DAYS`(환경 변수, 기본 3일). 기한 지난 Claim을 대체할 때 같은 트랜잭션에서 이전 Claim을 `EXPIRED`로 바꾼다. | §5, T14 |
| BE-CLAIM-06 | `POST /claims/{claimId}/abandon`: Plan을 `OPEN`으로 되돌리고 `currentClaimId`/`claimerId`/`dueAt`을 제거, Claim은 `ABANDONED`, Record가 있으면 `photoStatus = DISCARDED`. 실행자가 아니면 403, 이미 종료된 Claim이면 409 `CLAIM_NOT_ACTIVE`. | §6, T6, T16 |
| BE-CLAIM-07 | `GET /me/claims?status=ACTIVE|COMPLETED`는 `{ claim, plan, record|null }` 목록을 반환한다. | §7.1 |

### 4.5 방문 기록·완료 (REC)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-REC-01 | `PUT /claims/{claimId}/record`: 요청자 = 실행자, Claim `ACTIVE`, Plan `currentClaimId = claimId`일 때만 글(1–1,000자)을 저장한다. 완료 전까지 덮어쓸 수 있다. 실행자가 아니면 403, 그 외 조건 실패는 409 `CLAIM_NOT_ACTIVE`. | §7.4, G3 |
| BE-REC-02 | `photo`가 있으면 `contentType`(jpeg/png)·`size`(≤ 10MB)를 검증하고 `photoKey = photos/original/{claimId}/{uuid}.{ext}`, `photoStatus = UPLOAD_PENDING`으로 저장한 뒤 presigned PUT URL(10분, `ContentType`·`ContentLength` 서명 포함)을 `upload`로 반환한다. | §7.4 |
| BE-REC-03 | `photo` 없이 요청하면 진행 중인 사진 상태는 건드리지 않고 글만 갱신한다. | §7.4 |
| BE-REC-04 | `POST /claims/{claimId}/record/photo`: 새 `photoKey`와 업로드 URL을 발급하고 `UPLOAD_PENDING`으로 바꾼다. 이전 키의 처리 결과는 무시된다. | §7.4, T15 |
| BE-REC-05 | `GET /claims/{claimId}/record`는 `record`, `claimStatus`, `delivered`, `photoStatus`를 반환한다. 기록이 없으면 `record: null`, `photoStatus: "NONE"`. 포기·만료된 Claim도 실행자 본인은 조회할 수 있다. | §7.4 |
| BE-REC-06 | `POST /claims/{claimId}/complete`(`mode: TEXT_ONLY`)는 트랜잭션 전에 요청자 = 실행자인지 확인한다(아니면 403). 완료는 공통 함수 `complete_claim(claimId, mode, ...)`로 Plan·Claim·Record 갱신과 InboxItem 생성을 하나의 트랜잭션에서 처리한다. | §6, G4 |
| BE-REC-07 | 완료 반복 호출은 멱등이다. 이미 완료된 Claim이면 기존 결과를 200으로 반환하고 `CompleteDuplicate` 메트릭을 올린다. 수신함 항목과 Record는 1건만 남는다. | G2, T5 |
| BE-REC-08 | `TEXT_ONLY` 완료 시 사진이 없었으면 `photoStatus = NONE`, 처리 중·실패였으면 `DISCARDED`. 이미 사진과 함께 완료된 뒤의 `TEXT_ONLY` 요청은 200, `hasPhoto: true`. | §6, §7.4, T10 |
| BE-REC-09 | 완료 조건 실패 분기: Claim `COMPLETED` → 200 멱등, `ABANDONED`/`EXPIRED` 또는 `currentClaimId` 불일치 → 409 `CLAIM_NOT_ACTIVE`, 글 없음 → 400 `RECORD_REQUIRED`. | §6, T6, T7 |
| BE-REC-10 | 모든 트랜잭션은 공통 실패 처리를 따른다: `TransactionConflict`가 있으면 50·100·200ms 백오프로 최대 3회 재시도, 모두 `ConditionalCheckFailed`일 때만 `ConsistentRead`로 재조회해 분기, 판정 불가 시 503 `TRY_AGAIN`. | §6 |

### 4.6 사진 처리 (PHOTO)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-PHOTO-01 | S3 `ObjectCreated`(prefix `photos/original/`) → SQS `PhotoQueue` → `PhotoProcessorFunction`(batch 1, timeout 30s, 1024MB). visibility timeout = 6 × Lambda timeout, `maxReceiveCount` 3 → `PhotoDLQ`. | §8 |
| BE-PHOTO-02 | `s3:TestEvent`나 `Records` 없는 메시지는 정상 종료한다. S3 키는 `unquote_plus`로 디코딩한다. | §8 |
| BE-PHOTO-03 | Record를 `ConsistentRead`로 읽어 이미 전달됨·`DISCARDED`·`photoKey` 불일치면 무시하고(처리본 있으면 삭제) 종료한다. | §8, T10, T15 |
| BE-PHOTO-04 | 조건부로 `PROCESSING` 전환 → 매직 바이트(JPEG/PNG)·실제 크기 ≤ 10MB·최대 해상도 검사. 실패하면 `FAILED` + `photoError`, 재시도하지 않는다. | §8, T8 |
| BE-PHOTO-05 | EXIF 방향 반영 → 긴 변 1600px 축소 → EXIF/GPS 포함 모든 메타데이터 제거 → JPEG 재인코딩 → `photos/processed/{claimId}/{uuid}.jpg`. | §8, G7, T11 |
| BE-PHOTO-06 | 처리 후 `complete_claim(claimId, WITH_PHOTO, key, processedKey)`를 호출한다. `READY`는 별도로 쓰지 않고 완료 트랜잭션이 설정한다. 경합으로 조건이 실패하면 처리본을 지운다. | §6, §8 |
| BE-PHOTO-07 | 일시 오류는 예외로 SQS 재시도한다. `ApproximateReceiveCount >= 3`이면 예외 전에 조건부로 `FAILED`를 기록한다. 환경 변수 `FAULT_INJECT=photo`로 강제 예외를 낼 수 있다. | §8, T9 |
| BE-PHOTO-08 | 사진 버킷: 퍼블릭 차단, 원본 7일 수명 주기, CORS(API와 같은 Origin, `PUT`/`GET`, `Content-Type`). | §8 |
| BE-PHOTO-09 | 버킷 알림 순환 의존성을 피하기 위해 버킷 이름을 `${AWS::StackName}-photos-${AWS::AccountId}`로 고정하고 버킷에 `DependsOn: PhotoQueuePolicy`를 둔다. | §8 |

### 4.7 수신함·답장 (INBOX)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-INBOX-01 | `GET /inbox`는 요청자의 InboxItem을 `createdAt` 역순으로 반환한다. | §7.5 |
| BE-INBOX-02 | `GET /inbox/{itemId}`는 `item`, `plan`, `record { letter, deliveredAt, photoUrl, photoCaption }`을 반환한다. `photoUrl`은 15분 presigned GET, `photoCaption`은 사진이 있을 때 마지막 장소 이름 + "에서". 소유자가 아니면 404. | §7.5, T2 |
| BE-INBOX-03 | `POST /inbox/{itemId}/read` → 204, `read = true`. | §7.5 |
| BE-INBOX-04 | `POST /inbox/{itemId}/reply`(`text` 1–100자)는 `attribute_not_exists(reply)` 조건으로 1회만 저장한다. 성공 201, 중복 409 `REPLY_ALREADY_SENT`. InboxItem 생성 시 `reply` 속성은 쓰지 않는다(응답에서만 `null`). | §5, §7.5, T17 |

### 4.8 배포·인프라 (INFRA)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-INFRA-01 | `sam build && sam deploy --config-env demo`로 빈 계정에 스택 전체가 한 번에 생성된다. `samconfig.toml`에 `dev`, `demo` 환경을 둔다. | §13 |
| BE-INFRA-02 | 스택은 웹 버킷(비공개) + CloudFront(OAC)를 만든다. SPA 라우팅을 위해 CloudFront에서 403/404를 `/index.html`(200)으로 돌려준다. | §4, §11 |
| BE-INFRA-03 | API CORS: Origin = CloudFront 도메인, `http://localhost:5173` / 헤더 `Authorization`, `Content-Type`, `Idempotency-Key` / 메서드 `GET`, `POST`, `PUT`, `OPTIONS`. | §7 |
| BE-INFRA-04 | 출력값 `ApiUrl`, `UserPoolId`, `UserPoolClientId`, `WebBucketName`, `DistributionId`, `DistributionDomain`을 내보낸다(프론트 배포용). | §11 |
| BE-INFRA-05 | `BEDROCK_MODEL_ID`, `CLAIM_DURATION_DAYS`, Bedrock 리전 등 설정값은 파라미터/환경 변수로 분리한다. | §13 |
| BE-INFRA-06 | Lambda별 IAM 권한은 필요한 테이블·버킷 prefix·큐·Bedrock 모델로 한정한다. | 보안 기본값 |

### 4.9 관측·측정 (OBS)

| ID | 요구사항 | 근거 |
|---|---|---|
| BE-OBS-01 | EMF 로그로 `ClaimConflict`, `CompleteDuplicate`, `StaleRequestRejected`, `PhotoProcessingMs`, `PhotoFailed`, `AiLatencyMs`, `AiFailure`, `AiInvalidOutput`을 기록한다. | §10 |
| BE-OBS-02 | 알람: `PhotoDLQ` 메시지 수 > 0, API Gateway 5xx, Lambda Errors/Duration. | §10 |
| BE-OBS-03 | CloudWatch 대시보드에 API 오류, 사진 처리 시간, AI 지연, DLQ 깊이를 표시한다. | §13 |
| BE-OBS-04 | `scripts/report.py`가 기한 내 방문 기록, 답장률, 모드별 작성 시간 중앙값, 모드별 `addedNotInSource`를 집계한다. | §10 |
| BE-OBS-05 | `scripts/seed.py`가 데모 계정 3개(지연, 수현, 민준)와 예시 계획을 다시 만든다. | §11, §13 |

## 5. 비기능 요구사항

| ID | 분류 | 요구사항 |
|---|---|---|
| BE-NFR-01 | 정합성 | 이어받기 1명 확정(G1), 완료·수신함 1건(G2), 종료된 Claim 쓰기 거부(G3), 완료 원자성(G4)을 T3–T7로 증명한다. |
| BE-NFR-02 | 보안 | 권한 판단은 JWT `sub`와 저장된 `ownerId`/`claimerId` 비교로만 한다(§9 표). 남의 리소스는 403 또는 404로 응답하고 데이터를 노출하지 않는다. |
| BE-NFR-03 | 개인정보 | 처리된 사진에 EXIF/GPS가 남지 않는다. 사진 접근은 presigned URL로만 한다. 로그에 토큰·사연 원문·사진 URL을 남기지 않는다. |
| BE-NFR-04 | 응답 시간 | AI 카드 요청은 30초(HTTP API 통합 상한) 안에 성공 또는 정해진 에러 코드로 응답한다. |
| BE-NFR-05 | 오류 형식 | 모든 4xx/5xx는 `{ "error": { "code", "message", "details"? } }` 형식이다. 시간 값은 ISO 8601 UTC. |
| BE-NFR-06 | 재현성 | 새 스택 배포 + seed로 데모 환경을 언제든 다시 만들 수 있다. |
| BE-NFR-07 | 계약 일치 | 응답 형태가 프론트 MSW 목업과 일치한다. 계약 변경은 `MVP.md` §7을 먼저 고치고 프론트에 알린 뒤 반영한다. |

## 6. 인수 기준

- T1–T17(`MVP.md` §12)이 배포된 스택에서 통과한다. T3·T5는 `concurrent.futures`로 실제 API를 동시에 호출하고 결과 로그를 보관한다.
- `MVP.md` §13 완료 기준의 모든 항목을 만족한다.
- 90초 시연 시나리오가 배포 환경에서 동작한다: 지연 사연 입력 → AI 카드 → 공개 → 수현·민준 동시 이어받기(한 명만 확정) → 사진+기록 전송 → 지연 수신함·답장.

## 7. 확인이 필요한 사항

`MVP.md`에 명시되지 않았거나 프론트 문서와 맞춰야 하는 부분이다. 구현 전에 팀이 결정하고 결과를 `MVP.md`에 반영한다.

| # | 내용 | 제안 |
|---|---|---|
| Q1 | `GET /claims/{claimId}/record`를 실행자가 아닌 사람이 호출할 때의 코드가 없다. | 403 `FORBIDDEN`. 프론트는 403/404/409를 모두 "종료된 이어받기"로 처리하므로 영향 없음. |
| Q2 | 처리 Lambda 1단계의 "`READY`이고 `deliveredAt` 없음" 상태는 `READY`를 완료 트랜잭션에서만 쓰는 현재 설계에서는 생기지 않는다. | 방어 코드로 두되 테스트 대상에서는 뺀다. |
| Q3 | `GET /plans`의 `cursor` 형식이 정해지지 않았다. OPEN·CLAIMED 두 쿼리를 합치므로 단일 `LastEvaluatedKey`로 표현하기 어렵다. | MVP는 `limit` 최대 50, `nextCursor: null` 고정. 프론트는 더 보기 없이 첫 페이지만 표시. |
| Q4 | 계획 상세의 "이어받음" 상태 노출 시 숨겨야 할 필드 목록. | BE-PLAN-08의 허용 목록으로 확정. |
| Q5 | 시범 운영 지표의 "처음 5개 Claim"에 `ABANDONED` 포함 여부. | 팀 결정 후 `report.py` 옵션으로 둘 다 계산. |
| Q6 | Bedrock 모델 ID와 리전. | 서울 리전 가용 경량 모델 확인 후 `samconfig.toml` 파라미터로 지정. |
