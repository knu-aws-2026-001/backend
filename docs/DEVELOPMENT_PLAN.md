# 이어온 하루 — Backend 개발계획서

| 항목 | 내용 |
|---|---|
| 대상 저장소 | `knu-aws-2026-001/backend` |
| 관련 문서 | [`MVP.md`](../MVP.md), [`REQUIREMENTS.md`](./REQUIREMENTS.md), [frontend `DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/frontend/blob/main/docs/DEVELOPMENT_PLAN.md) |
| 스택 | Python 3.12, AWS SAM, API Gateway HTTP API, Lambda, DynamoDB, S3, SQS, Cognito, Bedrock, CloudFront, CloudWatch |

요구사항 ID(`BE-*`)는 [`REQUIREMENTS.md`](./REQUIREMENTS.md)를 가리킨다. 일정(날짜)은 팀 인원과 기간이 정해지면 6장 표에 채운다.

---

## 1. 개발 원칙

1. **계약 우선.** API 계약은 `MVP.md` §7이 기준이다. 프론트는 이 계약으로 MSW 목업을 만들어 병행 개발하므로, 응답 형태를 바꿀 때는 `MVP.md`를 먼저 고치고 프론트 담당에게 알린 뒤 코드에 반영한다.
2. **정합성 로직은 한 곳에.** 이어받기·포기·완료 트랜잭션은 `src/domain/claim_service.py`에만 둔다. API 완료와 사진 처리 Lambda는 같은 `complete_claim()`을 호출한다.
3. **배포된 스택에서 검증.** 동시성·멱등성은 로컬 목으로 증명할 수 없으므로 T1–T17은 실제 `dev` 스택을 대상으로 돌린다.
4. **처음부터 배포 가능하게.** 1단계부터 `sam deploy`가 되는 상태를 유지하고, 각 단계 끝에 `dev` 스택에 배포한다.

## 2. 저장소 구조

```
template.yaml               # SAM: HttpApi, Cognito, DynamoDB, S3(웹·사진), CloudFront(OAC), SQS+DLQ, Lambda, 알람
samconfig.toml              # dev / demo
src/
  common/                   # auth.py, ddb.py(트랜잭션·재시도 헬퍼), errors.py, responses.py, validation.py, metrics.py
  handlers/                 # me.py, plans.py, ai_card.py, claims.py, inbox.py, pre_signup.py
  domain/                   # claim_service.py, plan_service.py, inbox_service.py, fidelity.py(find_not_in_source)
  workers/photo_processor.py
  prompts/day_card.md
layers/pillow/
tests/
  unit/                     # moto 또는 DynamoDB Local 기반
  integration/              # 배포 스택 대상: test_authz, test_concurrency, test_idempotency, test_photo, test_ai
scripts/
  seed.py
  report.py
docs/
  REQUIREMENTS.md
  DEVELOPMENT_PLAN.md
```

공통 모듈 역할
- `errors.py`: `ApiError(code, status, message, details)`와 `MVP.md` §7.6 코드 상수. 핸들러는 예외만 던지고 `responses.py`가 에러 형식으로 바꾼다.
- `ddb.py`: `transact_with_retry(items)` — `TransactionConflict` 백오프(50·100·200ms) 재시도, `ConditionalCheckFailed`만 남으면 `ConditionFailed` 예외로 올려 서비스 계층이 분기하게 한다(BE-REC-10).
- `validation.py`: pydantic 모델(PlanCreate, RecordPut, PhotoRequest, Reply, AiCard). 검증 오류를 `details`의 `places[0].name` 형식 경로로 바꾼다.
- `responses.py`: Plan·Claim·Record·InboxItem 직렬화. 허용 목록 방식으로 내부 속성을 걸러낸다(BE-PLAN-08).

## 3. 단계별 계획

각 단계는 "작업 → 산출물 → 검증 → 프론트 연동 지점" 순서로 적는다. 단계 순서는 `MVP.md` §14를 따른다.

### 1단계. SAM 골격과 인증

| 작업 | 요구사항 |
|---|---|
| `template.yaml` 골격: Globals(Python 3.12, 리전, 환경 변수), HttpApi + JWT authorizer, CORS | BE-AUTH-03, BE-INFRA-03 |
| Cognito User Pool(이메일 로그인, `nickname` 필수) + App Client(시크릿 없음) + PreSignUp Lambda | BE-AUTH-01, 02 |
| DynamoDB `HaruTable`(PK/SK, GSI1, GSI2, TTL 속성 `ttl`) | §6 |
| `common/` 모듈, `GET /me` | BE-AUTH-04, 05 |
| `samconfig.toml`(dev/demo), 출력값 정의 | BE-INFRA-01, 04 |

- 산출물: `dev` 스택, `ApiUrl`/`UserPoolId`/`UserPoolClientId` 출력.
- 검증: T1(토큰 없이 401), 가입 직후 로그인 가능, `GET /me` 닉네임 확인.
- **프론트 연동 S1**: 출력값을 프론트에 전달 → 프론트가 Cognito 로그인 연동을 시작할 수 있다.

### 2단계. 계획 생성·목록·상세, 멱등 키

| 작업 | 요구사항 |
|---|---|
| `POST /plans` 검증·저장, `Idempotency-Key` 처리(본문 해시 비교) | BE-PLAN-01, 02 |
| `find_not_in_source()` 구현과 `metrics` 계산 (AI 단계 전에 먼저 만든다) | BE-PLAN-03, BE-AI-06 |
| `GET /plans`(OPEN + 기한 지난 CLAIMED 병합), `GET /plans/{planId}`(isOwner/myClaim/claimable, 상태 표시 규칙) | BE-PLAN-04–08 |
| `version`·`GSI1PK` 갱신 규칙 | BE-PLAN-09 |

- 검증: 단위 테스트(검증 경계값, `find_not_in_source`), T13.

### 3단계. 이어받기·포기·완료 트랜잭션

| 작업 | 요구사항 |
|---|---|
| `claim_service.claim()` — 기한 지난 Claim의 `EXPIRED` 처리 포함 | BE-CLAIM-01–05 |
| `claim_service.abandon()` | BE-CLAIM-06 |
| `claim_service.complete_claim(claimId, mode, key=None, processedKey=None)` — 조건 실패 분기, 멱등 응답 | BE-REC-06–10 |
| `POST /plans/{planId}/claims`, `POST /claims/{claimId}/abandon`, `POST /claims/{claimId}/complete`, `GET /me/claims` | BE-CLAIM-07 |
| 메트릭 `ClaimConflict`, `CompleteDuplicate`, `StaleRequestRejected` | BE-OBS-01 |
| 통합 테스트 T3–T7, T14, T16 | — |

- 이 단계가 프로젝트의 핵심 검증 대상(G1–G4)이다. 3단계 통합 테스트가 통과하기 전에는 4단계로 넘어가지 않는다.
- 완료 트랜잭션은 Record가 있어야 하므로, 3단계 테스트에서는 4단계의 `PUT /record`를 먼저 최소 구현하거나 테스트 픽스처로 Record를 직접 넣는다.

### 4단계. 방문 기록(글)과 수신함

| 작업 | 요구사항 |
|---|---|
| `PUT /claims/{claimId}/record`(글만), `GET /claims/{claimId}/record` | BE-REC-01, 03, 05 |
| `GET /inbox`, `GET /inbox/{itemId}`, `POST /inbox/{itemId}/read`, `POST /inbox/{itemId}/reply` | BE-INBOX-01–04 |
| `unreadInboxCount` 계산(`GET /me`) | BE-AUTH-05 |
| 통합 테스트 T2, T17 | — |

- **프론트 연동 S2**: 2–4단계 API가 `dev`에 배포되면 프론트가 글 경로 전체(둘러보기 → 이어받기 → 글 기록 → 수신함 → 답장)를 실제 API로 전환한다.

### 5단계. 사진 파이프라인

| 작업 | 요구사항 |
|---|---|
| 사진 버킷(퍼블릭 차단, CORS, 수명 주기), `PhotoQueue` + `PhotoDLQ` + 큐 정책, 버킷 알림(`DependsOn`) | BE-PHOTO-01, 08, 09 |
| `PUT /record`의 `photo` 처리와 presigned PUT, `POST /record/photo` | BE-REC-02, 04 |
| Pillow 레이어, `photo_processor.py`(무시 조건 → PROCESSING → 검사 → 축소·메타데이터 제거 → `complete_claim(WITH_PHOTO)`) | BE-PHOTO-02–07 |
| `GET /inbox/{itemId}`의 `photoUrl`(15분), `photoCaption` | BE-INBOX-02 |
| DLQ 알람, `PhotoProcessingMs`/`PhotoFailed` 메트릭 | BE-OBS-01, 02 |
| 통합 테스트 T8–T11, T15 | — |

- 위험: 버킷 알림 순환 의존성(BE-PHOTO-09), Pillow 레이어 아키텍처 불일치. 레이어는 `sam build --use-container`로 Lambda와 같은 아키텍처에서 빌드한다.
- **프론트 연동 S3**: 사진 버킷 CORS가 배포되면 프론트가 업로드·폴링·실패·교체 화면을 실제 API로 전환한다.

### 6단계. AI 하루 카드

| 작업 | 요구사항 |
|---|---|
| Bedrock 모델·리전 확정(Q6), IAM 권한 | BE-AI-02, BE-INFRA-05 |
| `prompts/day_card.md`, tool use 스키마, pydantic 검증·1회 재시도·남은 시간 검사 | BE-AI-03, 04 |
| 타임아웃·에러 코드 처리, `AiDraft` 저장(TTL 7일), `flags` 생성 | BE-AI-01, 05–07 |
| `POST /plans`에서 `aiDraftId`로 `aiFlaggedInDraft` 계산 | BE-PLAN-03 |
| 메트릭 `AiLatencyMs`, `AiFailure`, `AiInvalidOutput`, 통합 테스트 T12 | BE-OBS-01 |

- 프롬프트 품질 확인용으로 데모 사연을 포함한 예시 사연 5–10개를 `tests/fixtures/stories.json`에 두고 결과를 눈으로 확인한다.
- **프론트 연동 S4**: 배포 후 프론트가 AI 카드 화면을 실제 API로 전환한다.

### 7단계. 웹 배포, 관측, 시연 준비

| 작업 | 요구사항 |
|---|---|
| 웹 버킷 + CloudFront(OAC, SPA 에러 응답 → `/index.html`), CORS Origin에 배포 도메인 추가 | BE-INFRA-02, 03 |
| 알람(API 5xx, Lambda Errors/Duration), 대시보드 | BE-OBS-02, 03 |
| `scripts/seed.py`, `scripts/report.py` | BE-OBS-04, 05 |
| `demo` 환경으로 빈 계정 배포 → T1–T17 전체 실행 → 결과 로그 보관 | BE-INFRA-01 |
| 프론트와 90초 시연 리허설 | §6 인수 기준 |

- **프론트 연동 S5**: `WebBucketName`, `DistributionId`, `DistributionDomain` 전달 → 프론트가 빌드 산출물을 sync하고 캐시를 무효화한다.

## 4. 프론트엔드와의 연동 지점 요약

| 지점 | 백엔드가 넘기는 것 | 프론트가 시작하는 일 |
|---|---|---|
| S0 (시작 시) | `MVP.md` §7 계약 확정 | 타입 정의, MSW 목업으로 전 화면 개발 |
| S1 (1단계 후) | `ApiUrl`, `UserPoolId`, `UserPoolClientId` | Cognito 로그인·가입, `GET /me` |
| S2 (4단계 후) | 계획·이어받기·글 기록·수신함 API | 글 경로 실제 API 전환 |
| S3 (5단계 후) | 사진 업로드 URL, 처리 파이프라인 | 사진 업로드·폴링·실패·교체 |
| S4 (6단계 후) | `POST /ai/day-card` | AI 카드 화면 실제 연동 |
| S5 (7단계) | 웹 버킷·CloudFront 출력값, seed 데이터 | 배포, 시연 리허설 |

계약을 바꿔야 할 때: (1) `MVP.md` §7 수정 PR → (2) 프론트 담당 리뷰 승인 → (3) 백엔드 구현 PR. 프론트 MSW 핸들러도 같은 시점에 고친다.

## 5. 테스트 전략

| 수준 | 도구 | 대상 |
|---|---|---|
| 단위 | pytest, moto(또는 DynamoDB Local) | 입력 검증 경계값, `find_not_in_source`, 직렬화 허용 목록, 트랜잭션 분기 로직, 사진 처리(샘플 이미지·GPS EXIF 이미지·손상 파일) |
| 통합 | pytest + requests, `concurrent.futures` | `dev` 스택 대상 T1–T17. 테스트용 Cognito 사용자를 `AdminCreateUser`로 만들고 ID 토큰을 발급해 쓴다. |
| 계약 | 수동 + 리뷰 | 응답 JSON을 프론트 MSW 핸들러 예시와 비교(S2·S3·S4 시점) |
| 시연 | 리허설 | 90초 시나리오, `demo` 스택 |

통합 테스트 파일과 시나리오 대응

| 파일 | 시나리오 |
|---|---|
| `test_authz.py` | T1, T2 |
| `test_concurrency.py` | T3, T4, T5, T14 |
| `test_claim_lifecycle.py` | T6, T7, T16 |
| `test_photo.py` | T8, T9, T10, T11, T15 |
| `test_ai.py` | T12 |
| `test_idempotency.py` | T13 |
| `test_inbox.py` | T17 |

T9(`FAULT_INJECT=photo`)와 T12(Bedrock 권한 제거)는 환경 설정을 바꿔야 하므로 별도 마커(`@pytest.mark.fault`)로 분리해 마지막에 실행한다.

## 6. 일정 표 (팀이 채움)

| 단계 | 담당 | 시작 | 완료 | 연동 지점 |
|---|---|---|---|---|
| 1. SAM 골격·인증 | | | | S1 |
| 2. 계획·멱등 키 | | | | |
| 3. 이어받기·포기·완료 | | | | |
| 4. 기록(글)·수신함 | | | | S2 |
| 5. 사진 파이프라인 | | | | S3 |
| 6. AI 하루 카드 | | | | S4 |
| 7. 웹 배포·관측·시연 | | | | S5 |

병렬 작업이 가능한 조합: 2단계 이후 "3단계(트랜잭션)"와 "6단계(AI 프롬프트·검증 로직)"는 서로 의존하지 않으므로 인원이 2명 이상이면 나눠서 진행한다. 5단계의 사진 처리 로직(Pillow 축소·메타데이터 제거)도 3단계와 병행해 단위 테스트까지 먼저 만들 수 있다.

## 7. 작업 규칙 (제안)

- 브랜치: `main`은 배포 가능한 상태 유지. 작업은 `feat/<단계>-<요약>`(예: `feat/3-claim-transaction`) 브랜치에서 하고 PR로 병합한다.
- PR: 관련 요구사항 ID와 통과한 테스트 시나리오(T 번호)를 본문에 적는다. 리뷰어 1명 이상.
- 커밋 메시지: `feat:`, `fix:`, `test:`, `infra:`, `docs:` 접두어.
- 비밀값: 계정 ID·토큰·테스트 비밀번호는 저장소에 넣지 않는다. 테스트 계정 정보는 로컬 `.env`(gitignore)나 SSM Parameter Store에 둔다.

## 8. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| 트랜잭션 분기 오류로 이어받기 2명 확정 또는 수신함 중복 | 핵심 목표(G1, G2) 실패 | 3단계에서 T3–T7을 먼저 통과시키고, 이후 단계 PR마다 `test_concurrency.py` 재실행 |
| Bedrock 모델이 서울 리전에 없음 | AI 기능 지연 | Bedrock 클라이언트 리전을 별도 환경 변수로 분리(BE-INFRA-05). 6단계 시작 전에 모델 접근 권한 신청 |
| AI 응답 지연으로 30초 초과 | 코드 없는 500 | read timeout 10s, 남은 시간 12초 미만이면 재시도 생략(BE-AI-04, 05) |
| SAM 순환 의존성(버킷 알림 ↔ 큐 정책) | 배포 실패 | 버킷 이름 고정 + `DependsOn`(BE-PHOTO-09) |
| Pillow 레이어 빌드 환경 차이 | 사진 처리 Lambda import 실패 | `sam build --use-container` |
| 프론트·백엔드 계약 불일치 | 연동 시 화면 오류 | 4장 계약 변경 절차, S2–S4 시점 응답 비교 |
| 시연 중 데이터 꼬임 | 시연 실패 | `seed.py`로 재생성, 시연 직전 `demo` 데이터 초기화 |

## 9. 완료 체크리스트

`MVP.md` §13을 그대로 따른다.

- [ ] `sam build && sam deploy --config-env demo`로 빈 계정에 스택 생성
- [ ] §7 모든 엔드포인트가 계약대로 응답 (프론트 MSW와 형태 일치)
- [ ] T1–T17 통과, T3·T5 결과 로그 보관
- [ ] 90초 시연 시나리오가 배포 환경에서 동작
- [ ] CloudWatch 대시보드(API 오류, 사진 처리 시간, AI 지연, DLQ 깊이)
- [ ] `scripts/seed.py`로 데모 계정·데이터 재생성
- [ ] 설정값(Bedrock 모델 ID, 기한 일수 등) 파라미터 분리
