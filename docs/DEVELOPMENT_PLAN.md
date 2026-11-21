# 이어온 하루 — Backend 개발계획서

| 항목 | 내용 |
|---|---|
| 대상 저장소 | `knu-aws-2026-001/backend` |
| 관련 문서 | [`MVP.md`](../MVP.md), [`REQUIREMENTS.md`](./REQUIREMENTS.md), [frontend `DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/frontend/blob/main/docs/DEVELOPMENT_PLAN.md) |
| 스택 | Python 3.12, AWS SAM, API Gateway HTTP API, Lambda, DynamoDB, S3, SQS, Cognito, Bedrock(Converse tool use, Global CRIS, Guardrails 보조), CloudFront, CloudWatch |

요구사항 ID(`BE-*`)는 [`REQUIREMENTS.md`](./REQUIREMENTS.md)를 가리킨다. 일정(날짜)은 팀 인원과 기간이 정해지면 6장 표에 채운다.

---

## 1. 개발 원칙

1. **계약 우선.** API 계약은 `MVP.md` §7이 기준이다. 프론트는 이 계약으로 MSW 목업을 만들어 병행 개발하므로, 응답 형태를 바꿀 때는 `MVP.md`를 먼저 고치고 프론트 담당에게 알린 뒤 코드에 반영한다.
2. **정합성 로직은 한 곳에.** 이어받기·포기·완료 트랜잭션은 `src/domain/claim_service.py`에만 둔다. API 완료와 사진 처리 Lambda는 같은 `complete_claim()`을 호출한다.
3. **배포된 스택에서 검증.** 동시성·멱등성은 로컬 목으로 증명할 수 없으므로 T1–T22는 실제 `dev` 스택을 대상으로 돌린다.
4. **처음부터 배포 가능하게.** 1단계부터 `sam deploy`가 되는 상태를 유지하고, 각 단계 끝에 `dev` 스택에 배포한다.

## 2. 저장소 구조

```
template.yaml               # SAM: HttpApi, Cognito, DynamoDB, S3(웹·사진), CloudFront(OAC), SQS+DLQ, Lambda, 알람
samconfig.toml              # dev / demo
src/
  common/                   # auth.py, ddb.py(트랜잭션·재시도 헬퍼), errors.py, responses.py, validation.py, metrics.py
  handlers/                 # me.py, plans.py, ai_card.py(/ai/card-sessions*), claims.py, inbox.py, pre_signup.py
  agent/                    # loop.py(Converse tool use 루프·예산·12초 규칙·레벨 0), tools.py(check_draft, ask_planner, submit_card)
  domain/                   # claim_service.py, plan_service.py, inbox_service.py,
                            # fidelity.py(find_not_in_source(sources, targets)), privacy.py(scan_privacy)
  workers/photo_processor.py
  prompts/                  # day_card_agent.md(에이전트), day_card.md(레벨 0·단발 기준선)
layers/pillow/
tests/
  unit/                     # moto 또는 DynamoDB Local 기반, Bedrock 스텁
  integration/              # 배포 스택 대상: test_authz, test_concurrency, test_idempotency, test_photo, test_ai
  fixtures/stories.json     # 오프라인 평가 사연 15건
scripts/
  seed.py
  report.py                 # 시범 운영 지표, check_invariants(선택)
  eval_agent.py             # 오프라인 평가: 단발·레벨 0·에이전트 비교
docs/
  REQUIREMENTS.md
  DEVELOPMENT_PLAN.md
```

공통 모듈 역할
- `errors.py`: `ApiError(code, status, message, details)`와 `MVP.md` §7.6 코드 상수. 핸들러는 예외만 던지고 `responses.py`가 에러 형식으로 바꾼다.
- `ddb.py`: `transact_with_retry(items)` — `TransactionConflict` 백오프(50·100·200ms) 재시도, `ConditionalCheckFailed`만 남으면 `ConditionFailed` 예외로 올려 서비스 계층이 분기하게 한다(BE-REC-10).
- `validation.py`: pydantic 모델(PlanCreate, RecordPut, PhotoRequest, Reply, AiCard, AiSessionCreate, AiTurn, SessionView). 검증 오류를 `details`의 `places[0].name` 형식 경로로 바꾼다. `check_draft`의 스키마 검증도 같은 경로 표기를 쓴다.
- `agent/tools.py`: 도구 3개와 서버 게이트 판정은 DynamoDB를 직접 부르지 않는 순수 함수로 두고, 세션 쓰기·예산·결정적 폴백 적용은 `loop.py`가 맡는다. 다른 에이전트 프레임워크로 옮길 때 도구를 그대로 쓸 수 있다. 단위 테스트는 Bedrock 스텁으로 돌린다.
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
| `find_not_in_source(sources, targets)` 구현과 `metrics` 계산 (AI 단계 전에 먼저 만든다. 처음부터 근거 텍스트 목록을 받는 시그니처로 만든다) | BE-PLAN-03, BE-AI-06 |
| `privacy.py`(`scan_privacy`) 정규식·비공개 장소 사전과 단위 테스트 (`privacyInPublished` 계산에도 쓴다) | BE-AI-06, BE-PLAN-03 |
| `GET /plans`(OPEN + 기한 지난 CLAIMED 병합), `GET /plans/{planId}`(isOwner/myClaim/claimable, 상태 표시 규칙) | BE-PLAN-04–08 |
| `version`·`GSI1PK` 갱신 규칙 | BE-PLAN-09 |

- 검증: 단위 테스트(검증 경계값, `find_not_in_source`, `scan_privacy`: 전화·이메일·주민번호·차량번호·동/호수·번지, 정규화 사전 픽스처), T13.

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

### 6단계. AI 하루 카드 정리 에이전트

6단계는 6a(레벨 0과 계약)와 6b(에이전트 루프)로 나눈다. 6a로 계약과 화면을 먼저 고정하고, 6b는 계약을 바꾸지 않고 내부만 교체한다.

#### 6a. 레벨 0, card-sessions 계약, AiQuota (3단계와 병행)

| 작업 | 요구사항 |
|---|---|
| Global CRIS 모델 확정(Q6): `list-inference-profiles`·`GetInferenceProfile`로 라우팅 확인, SCP 목적지 리전 점검, 모델 접근 권한 신청, IAM(추론 프로필 ARN + 목적지 모델 ARN) | BE-AI-02, BE-INFRA-05, 06 |
| `/ai/card-sessions*` 3개 경로, `SessionView` 직렬화, `AiSession` 저장(TTL 30일), 소유자 아니면 404 | BE-AI-01, 07 |
| `turns` 경로의 분기: 세션 없음·소유자 아님 404, `turn ≠ expectedTurn` 200, `READY`·`FAILED` 409 `AI_SESSION_CLOSED`(레벨 0은 생성 즉시 `READY`이므로 같은 `turn`의 `turns`는 `AI_SESSION_CLOSED`). 턴 잠금·`AI_SESSION_BUSY`는 6b | BE-AI-09 |
| 레벨 0 파이프라인(`AGENT_MODE=pipeline`): `prompts/day_card.md` 단발 생성 → pydantic 검증 → `check_draft` → 결정적 폴백 → `READY` | BE-AI-11 |
| 타임아웃 규칙(Lambda 27s, connect 2 / read 8, 남은 12초 미만이면 호출 중단)과 에러 코드 | BE-AI-05 |
| `AiQuota`(하루 10세션, 429 `AI_QUOTA_EXCEEDED`), 경로별 스로틀 | BE-AI-10, BE-INFRA-06 |
| `POST /plans`의 `authoring.aiSessionId`로 AI 세션 지표 계산 | BE-PLAN-03 |
| 메트릭 `AiLatencyMs`, `AiFailure`, `AiInvalidOutput`, 통합 테스트 T12 | BE-OBS-01 |

- 산출물: 레벨 0으로 동작하는 card-sessions API. 프론트가 이 시점부터 실제 API로 3-1 화면을 붙인다.

#### 6b. 에이전트 루프, 게이트, 잠금

| 작업 | 요구사항 |
|---|---|
| `agent/loop.py`: Converse `toolChoice = any` 루프, 세션 쓰기, 세션 예산, `end_turn` 재촉 1회, 마지막 호출·`finishRequested`에서 `ask_planner` 제거, 게이트 거부 2회 후 결정적 폴백 | BE-AI-04, 08 |
| `agent/tools.py`: `check_draft`, `ask_planner`(거부 규칙), `submit_card`(서버 게이트 판정). 모두 순수 함수 | BE-AI-04 |
| `prompts/day_card_agent.md`: 데이터 블록(`<story>`, `<answer>`), 질문 우선순위, "빼기보다 원문 표현으로 바꾸기" | BE-AI-03 |
| 턴 잠금 조건부 갱신(`ownerId = :me` 포함, 잠금과 함께 `status = WORKING`), `expectedTurn` 멱등, `AI_SESSION_BUSY` | BE-AI-09 |
| (선택) Guardrails `ApplyGuardrail` 보조(Q10) | BE-AI-12 |
| 메트릭 `AgentModelCalls`, `AgentTokens`, `AgentTurnMs`, `AgentQuestionsAsked`, `AgentGateRejected`, `AgentDegraded`, `AgentSelfRepairs` | BE-OBS-01 |
| 통합 테스트 T18–T22, 오프라인 평가 15건(`scripts/eval_agent.py`) | BE-OBS-04 |

- 개발은 Bedrock 스텁으로 먼저 하고, 3단계(T3–T7)를 밀어내지 않는다.
- 오프라인 평가: `tests/fixtures/stories.json` 15건(데모 사연, 비공개 장소·연락처·인젝션 함정 사연 포함)을 같은 모델로 ① 기존 단발 프롬프트 ② 레벨 0 ③ 에이전트에 돌려 `NOT_IN_SOURCE`·`MISSING`·`PRIVACY` 잔존 수, 호출 수, 지연을 비교한다. 에이전트가 낫지 않은 유형도 그대로 적는다. 결과로 모델(Q6)을 고른다.
- **리허설 전에 T18–T22를 통과하지 못하면 `AGENT_MODE=pipeline`(레벨 0)으로 출시한다.** 계약과 화면은 그대로이고 질문 말풍선만 빠진다.
- **프론트 연동 S4**: 6a 배포 후 프론트가 AI 정리 화면(3-1)을 실제 API로 전환한다. 6b는 계약 변경 없이 서버 내부만 교체하고, 6b 배포 후 프론트가 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로를 실제 API로 다시 확인한다.

### 7단계. 웹 배포, 관측, 시연 준비

| 작업 | 요구사항 |
|---|---|
| 웹 버킷 + CloudFront(OAC, SPA 에러 응답 → `/index.html`), CORS Origin에 배포 도메인 추가 | BE-INFRA-02, 03 |
| 알람(API 5xx, Lambda Errors/Duration, `AgentTokens`), AWS Budgets 월 알림, 대시보드(AI 세션 상태 분포 포함) | BE-OBS-02, 03 |
| `scripts/seed.py`, `scripts/report.py` | BE-OBS-04, 05 |
| `demo` 환경으로 빈 계정 배포 → T1–T22 전체 실행 → 결과 로그 보관 | BE-INFRA-01 |
| 프론트와 90초 시연 리허설 | §6 인수 기준 |
| AI 시연 안정장치: 리허설 10회 이상으로 질문 1개 경로 재현 확인, 턴 p95와 `read_timeout` 8초 안 완료 여부 측정(Q12), 시연 직전 1회 호출로 콜드 스타트 제거, 지연이 길면 `AGENT_MODE=pipeline` 전환 | BE-AI-05, 11 |

- **프론트 연동 S5**: `WebBucketName`, `DistributionId`, `DistributionDomain` 전달 → 프론트가 빌드 산출물을 sync하고 캐시를 무효화한다.

## 4. 프론트엔드와의 연동 지점 요약

| 지점 | 백엔드가 넘기는 것 | 프론트가 시작하는 일 |
|---|---|---|
| S0 (시작 시) | `MVP.md` §7 계약 확정 (card-sessions 계약 포함) | 타입 정의, MSW 목업(AI 세션 상태 머신 포함)으로 전 화면 개발 |
| S1 (1단계 후) | `ApiUrl`, `UserPoolId`, `UserPoolClientId` | Cognito 로그인·가입, `GET /me` |
| S2 (4단계 후) | 계획·이어받기·글 기록·수신함 API | 글 경로 실제 API 전환 |
| S3 (5단계 후) | 사진 업로드 URL, 처리 파이프라인 | 사진 업로드·폴링·실패·교체 |
| S4 (6a 후) | `/ai/card-sessions*` (레벨 0) | AI 정리 화면 실제 연동. 6b는 계약 변경 없이 교체하고, 6b 배포 후 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로 재확인 |
| S5 (7단계) | 웹 버킷·CloudFront 출력값, seed 데이터 | 배포, 시연 리허설 |

계약을 바꿔야 할 때: (1) `MVP.md` §7 수정 PR → (2) 프론트 담당 리뷰 승인 → (3) 백엔드 구현 PR. 프론트 MSW 핸들러도 같은 시점에 고친다.

## 5. 테스트 전략

| 수준 | 도구 | 대상 |
|---|---|---|
| 단위 | pytest, moto(또는 DynamoDB Local), Bedrock 스텁 | 입력 검증 경계값, `find_not_in_source`, `scan_privacy`, 직렬화 허용 목록, 트랜잭션 분기 로직, 에이전트 루프(예산·게이트·폴백·12초 규칙), 사진 처리(샘플 이미지·GPS EXIF 이미지·손상 파일) |
| 통합 | pytest + requests, `concurrent.futures` | `dev` 스택 대상 T1–T22. 테스트용 Cognito 사용자를 `AdminCreateUser`로 만들고 ID 토큰을 발급해 쓴다. |
| 계약 | 수동 + 리뷰 | 응답 JSON을 프론트 MSW 핸들러 예시와 비교(S2·S3·S4 시점) |
| 오프라인 평가 | `scripts/eval_agent.py` | `tests/fixtures/stories.json` 15건, 단발·레벨 0·에이전트 비교 |
| 시연 | 리허설 | 90초 시나리오, `demo` 스택 |

통합 테스트 파일과 시나리오 대응

| 파일 | 시나리오 |
|---|---|
| `test_authz.py` | T1, T2 |
| `test_concurrency.py` | T3, T4, T5, T14 |
| `test_claim_lifecycle.py` | T6, T7, T16 |
| `test_photo.py` | T8, T9, T10, T11, T15 |
| `test_ai.py` | T12, T18, T19, T20, T21, T22 |
| `test_idempotency.py` | T13 |
| `test_inbox.py` | T17 |

T9(`FAULT_INJECT=photo`), T12(Bedrock 권한 제거), T21(`FAULT_INJECT=ai-slow`)은 환경 설정을 바꿔야 하므로 별도 마커(`@pytest.mark.fault`)로 분리해 마지막에 실행한다.

## 6. 일정 표 (팀이 채움)

| 단계 | 담당 | 시작 | 완료 | 연동 지점 |
|---|---|---|---|---|
| 1. SAM 골격·인증 | | | | S1 |
| 2. 계획·멱등 키 | | | | |
| 3. 이어받기·포기·완료 | | | | |
| 4. 기록(글)·수신함 | | | | S2 |
| 5. 사진 파이프라인 | | | | S3 |
| 6a. AI 레벨 0·계약·AiQuota | | | | S4 |
| 6b. AI 에이전트 루프·T18–T22·오프라인 평가 | | | | |
| 7. 웹 배포·관측·시연 | | | | S5 |

병렬 작업이 가능한 조합: 2단계 이후 "3단계(트랜잭션)"와 "6a단계(AI 레벨 0·계약)"는 서로 의존하지 않으므로 인원이 2명 이상이면 나눠서 진행한다. 6b는 Bedrock 스텁으로 개발하되 3단계 T3–T7을 우선한다. 5단계의 사진 처리 로직(Pillow 축소·메타데이터 제거)도 3단계와 병행해 단위 테스트까지 먼저 만들 수 있다.

추가 공수(기존 6단계 대비, 인일): 6a +1, 6b +2.5, 오프라인 평가 15건과 스크립트 +1. 계획 대표 사진·`GET /me/plans`를 이번 일정에서 빼서 약 1.5~2인일을 상쇄한다. 선택 작업: Guardrails 보조 +0.5, `check_invariants` +0.5~1, P1 질문 도우미 약 +3.

## 7. 작업 규칙 (제안)

- 브랜치: `main`은 배포 가능한 상태 유지. 작업은 `feat/<단계>-<요약>`(예: `feat/3-claim-transaction`) 브랜치에서 하고 PR로 병합한다.
- PR: 관련 요구사항 ID와 통과한 테스트 시나리오(T 번호)를 본문에 적는다. 리뷰어 1명 이상.
- 커밋 메시지: `feat:`, `fix:`, `test:`, `infra:`, `docs:` 접두어.
- 비밀값: 계정 ID·토큰·테스트 비밀번호는 저장소에 넣지 않는다. 테스트 계정 정보는 로컬 `.env`(gitignore)나 SSM Parameter Store에 둔다.

## 8. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| 트랜잭션 분기 오류로 이어받기 2명 확정 또는 수신함 중복 | 핵심 목표(G1, G2) 실패 | 3단계에서 T3–T7을 먼저 통과시키고, 이후 단계 PR마다 `test_concurrency.py` 재실행 |
| 경량 모델(Haiku 등)은 서울에서 Global CRIS로만 호출됨. 국외 리전 처리, SCP·권한 승인 지연 | AI 기능 지연, 동의 문구 누락 | 6a 시작 전 Q6·Q7 확정: `list-inference-profiles`·`GetInferenceProfile` 확인, SCP 목적지 리전 점검, 모델 접근 권한 신청, 동의 문구 반영 |
| AI 턴 지연으로 30초 초과 | 코드 없는 500 | Lambda 27s, read timeout 8s, 남은 시간 12초 미만이면 모델 호출 없이 `WORKING` 응답 → 프론트 자동 continue, 연속 2회면 `finish`(BE-AI-05). 리허설 p95 측정, 초과하면 레벨 0 |
| 경량 모델의 한국어 tool use 불안정(반복 호출, 제출 미루기) | 카드 품질 저하, 호출 증가 | `toolChoice = any`, 제출 재촉 1회, 마지막 호출에서 ask 제거, 세션 예산, 결정적 폴백, 오프라인 평가로 모델 선정 |
| 에이전트 작업이 정합성 핵심(G1–G4) 일정을 밀어냄 | 핵심 목표 지연 | 3단계 T3–T7 우선. 6a는 병행, 6b는 Bedrock 스텁으로 개발. 마감 시 불안정하면 `AGENT_MODE=pipeline`으로 출시 |
| 에이전트가 레벨 0보다 낫지 않음 | 에이전트 도입 근거 약화 | 오프라인 평가 표로 비교. 차이가 없으면 질문 기능만 남기고 수정 루프를 줄인다 |
| 명사 휴리스틱 오탐(빵집·빵 가게)으로 과잉 수정·불필요한 질문 | 작성 시간 증가 | "빼기보다 원문 표현으로 바꾸기" 프롬프트, 정규화 사전 픽스처, 메모 '다시 넣기' |
| 규칙 기반 개인정보 탐지 누락 | 공개 Plan에 개인정보 노출 | 탐지는 보조라고 명시. Guardrails Standard 보조, 사연 입력 안내와 계획자 확인 유지, 누락 사례를 픽스처에 추가 |
| 신규 계정이라 예약 동시성을 설정할 수 없음 | AI 호출 폭주 제어 불가 | HTTP API 경로별 스로틀과 `AiQuota`로 대신(Q8) |
| SAM 순환 의존성(버킷 알림 ↔ 큐 정책) | 배포 실패 | 버킷 이름 고정 + `DependsOn`(BE-PHOTO-09) |
| Pillow 레이어 빌드 환경 차이 | 사진 처리 Lambda import 실패 | `sam build --use-container` |
| 프론트·백엔드 계약 불일치 | 연동 시 화면 오류 | 4장 계약 변경 절차, S2–S4 시점 응답 비교 |
| 시연 중 데이터 꼬임 | 시연 실패 | `seed.py`로 재생성, 시연 직전 `demo` 데이터 초기화 |

## 9. 완료 체크리스트

`MVP.md` §13을 그대로 따른다.

- [ ] `sam build && sam deploy --config-env demo`로 빈 계정에 스택 생성
- [ ] §7 모든 엔드포인트가 계약대로 응답 (프론트 MSW와 형태 일치)
- [ ] T1–T22 통과, T3·T5 결과 로그 보관
- [ ] 90초 시연 시나리오가 배포 환경에서 동작
- [ ] CloudWatch 대시보드(API 오류, 사진 처리 시간, AI 지연, AI 세션 상태 분포, DLQ 깊이)
- [ ] `scripts/seed.py`로 데모 계정·데이터 재생성
- [ ] 설정값(Bedrock 모델 ID, `AGENT_MODE`, 기한 일수 등) 파라미터 분리
- [ ] 오프라인 평가(`scripts/eval_agent.py`, 15건) 결과표 보관
