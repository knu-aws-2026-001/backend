# 이어온 하루 — Backend 개발계획서

| 항목 | 내용 |
|---|---|
| 대상 저장소 | `knu-aws-2026-001/backend` |
| 관련 문서 | [`MVP.md`](../MVP.md), [`REQUIREMENTS.md`](./REQUIREMENTS.md), [frontend `DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/frontend/blob/main/docs/DEVELOPMENT_PLAN.md) |
| 스택 | Python 3.12, AWS SAM, API Gateway HTTP API, Lambda, DynamoDB, S3, SQS, Cognito, SSM Parameter Store, CloudFront, CloudWatch / AI: LangGraph(`StateGraph`, `DynamoDBSaver`) + `langchain-openai` `ChatOpenAI`로 OpenAI 호환 Chat Completions API / 로컬 개발: docker compose(AWS 호환 에뮬레이터) |

요구사항 ID(`BE-*`)는 [`REQUIREMENTS.md`](./REQUIREMENTS.md)를 가리킨다. 일정(날짜)은 팀 인원과 기간이 정해지면 6장 표에 채운다.

---

## 1. 개발 원칙

1. **계약 우선.** API 계약은 `MVP.md` §7이 기준이다. 프론트는 이 계약으로 MSW 목업을 만들어 병행 개발하므로, 응답 형태를 바꿀 때는 `MVP.md`를 먼저 고치고 프론트 담당에게 알린 뒤 코드에 반영한다.
2. **정합성 로직은 한 곳에.** 이어받기·포기·완료 트랜잭션은 `src/domain/claim_service.py`에만 둔다. API 완료와 사진 처리 Lambda는 같은 `complete_claim()`을 호출한다.
3. **개발은 로컬 Docker, 판정은 배포된 스택.** 예산 문제로 개발과 단위·로컬 통합 테스트는 docker compose 로컬 환경(10장)에서 한다. 동시성·멱등성은 에뮬레이터로 증명할 수 없으므로 T1–T22 판정, 데모, 시범 운영은 실제 AWS에 SAM으로 배포한 `dev`/`demo` 스택에서 한다.
4. **처음부터 배포 가능하게.** 1단계부터 `sam deploy`가 되는 상태를 유지하고, 각 단계 끝에 `dev` 스택에 배포한다. 같은 핸들러 코드가 로컬과 AWS에서 돌도록 엔드포인트는 환경 변수로만 바꾼다.

## 2. 저장소 구조

```
template.yaml               # SAM: HttpApi, Cognito, DynamoDB(HaruTable, HaruAgentCheckpoints), S3(웹·사진), CloudFront(OAC), SQS+DLQ, Lambda, 알람
samconfig.toml              # dev / demo
docker-compose.yml          # 로컬 개발 환경 (10장)
.env.example                # LLM_* 등 로컬 값 예시 (실제 .env는 gitignore)
src/
  common/                   # auth.py, ddb.py(트랜잭션·재시도 헬퍼), errors.py, responses.py, validation.py, metrics.py
  handlers/                 # me.py, plans.py, ai_card.py(/ai/card-sessions*), claims.py, inbox.py, pre_signup.py
  agent/                    # graph.py(StateGraph·조건부 엣지·체크포인터·레벨 0), nodes.py(draft, decide, check_draft, ask_planner, gate, fallback, pause),
                            # tools.py(도구 스키마·게이트 판정), state.py(AgentState), llm.py(ChatOpenAI 생성·Fake 채팅 모델)
  domain/                   # claim_service.py, plan_service.py, inbox_service.py,
                            # fidelity.py(find_not_in_source(sources, targets)), privacy.py(scan_privacy)
  workers/photo_processor.py
  prompts/                  # day_card_agent.md(에이전트), day_card.md(레벨 0·단발 기준선)
layers/pillow/
local/
  init/                     # 에뮬레이터 초기화 스크립트 (테이블·버킷·큐·유저 풀·시드 사용자)
  adapter/                  # HTTP 어댑터 + SQS 폴러 (Dockerfile)
tests/
  unit/                     # pytest, Fake 채팅 모델, botocore Stubber, 필요 시 로컬 에뮬레이터
  local/                    # docker compose 대상 로컬 통합 테스트 (조건식 로직, 사진 파이프라인 흐름, 세션 API)
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
- `agent/graph.py`·`nodes.py`: LangGraph `StateGraph`로 노드를 명시적으로 연결한다(`MVP.md` §7.2 도식). LLM 노드는 `draft`, `decide`뿐이고, 예산·시간 가드(`pause`)·`finish`는 조건부 엣지로 처리한다. 체크포인터는 `DynamoDBSaver`(`HaruAgentCheckpoints`, `consistent_read=True`).
- `agent/tools.py`: 도구 3개와 서버 게이트 판정은 DynamoDB를 직접 부르지 않는 순수 함수다. 단위 테스트는 Fake 채팅 모델로 돌린다.
- `agent/llm.py`: `LLM_*` 설정으로 `ChatOpenAI`를 만들고(`timeout`·`max_retries=0` 명시), AWS에서는 SSM SecureString 키를 콜드 스타트 때 1회 읽어 캐시한다. `LLM_PROVIDER=fake`면 시나리오별 Fake 채팅 모델을 돌려준다.
- `handlers/ai_card.py`: 턴 잠금, `AiQuota`, 그래프 실행(`thread_id = sessionId`, `durability="sync"`, `tracing_context(enabled=False)`), 그래프 상태 → `SessionView` 변환, trace 생성.
- `responses.py`: Plan·Claim·Record·InboxItem 직렬화. 허용 목록 방식으로 내부 속성을 걸러낸다(BE-PLAN-08).

## 3. 단계별 계획

각 단계는 "작업 → 산출물 → 검증 → 프론트 연동 지점" 순서로 적는다. 단계 순서는 `MVP.md` §14를 따른다.

### 1단계. SAM 골격과 인증

| 작업 | 요구사항 |
|---|---|
| `template.yaml` 골격: Globals(Python 3.12, 리전, 환경 변수. `LANGSMITH_TRACING=false`, `LANGCHAIN_TRACING_V2=false` 포함), HttpApi + JWT authorizer, CORS | BE-AUTH-03, BE-INFRA-03, BE-AI-15 |
| Cognito User Pool(이메일 로그인, `nickname` 필수) + App Client(시크릿 없음) + PreSignUp Lambda | BE-AUTH-01, 02 |
| DynamoDB `HaruTable`(PK/SK, GSI1, GSI2, TTL 속성 `ttl`) | §6 |
| `common/` 모듈, `GET /me` | BE-AUTH-04, 05 |
| `samconfig.toml`(dev/demo), 출력값 정의 | BE-INFRA-01, 04 |
| 로컬 개발 환경 골격: `docker-compose.yml`, init 스크립트(HaruTable+GSI+TTL, `HaruAgentCheckpoints`, 유저 풀+시드 사용자), HTTP 어댑터(payload 2.0 변환, JWKS 검증), `DynamoDBSaver` 로컬 연결 확인(Q21) | BE-INFRA-08 |

- 산출물: `dev` 스택, `ApiUrl`/`UserPoolId`/`UserPoolClientId` 출력.
- 검증: T1(토큰 없이 401), 가입 직후 로그인 가능, `GET /me` 닉네임 확인(`dev` 스택). 로컬에서는 시드 사용자 토큰으로 `GET /me`가 같은 응답을 내는지 확인한다(Q17).
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

6단계는 6a(레벨 0과 계약)와 6b(LangGraph 에이전트 그래프)로 나눈다. 6a로 계약과 화면을 먼저 고정하고, 6b는 계약을 바꾸지 않고 내부만 교체한다.

#### 6a. 레벨 0, card-sessions 계약, AiQuota (3단계와 병행)

| 작업 | 요구사항 |
|---|---|
| LLM 공급자·모델 후보 확정(Q6, Q7): 데이터 정책·처리 국가 확인, API 키를 SSM SecureString으로 등록, `ssm:GetParameter` IAM, `llm.py`(`ChatOpenAI` 생성, 타임아웃·재시도 명시), 파라미터 지원 실측(Q9) | BE-AI-02, BE-INFRA-05, 06, 07 |
| `/ai/card-sessions*` 3개 경로, `SessionView` 직렬화, `AiSession` 저장(TTL 30일), 소유자 아니면 404 | BE-AI-01, 07 |
| `turns` 경로의 분기: 세션 없음·소유자 아님 404, `turn ≠ expectedTurn` 200, `READY`·`FAILED` 409 `AI_SESSION_CLOSED`(레벨 0은 생성 즉시 `READY`이므로 같은 `turn`의 `turns`는 `AI_SESSION_CLOSED`). 턴 잠금·`AI_SESSION_BUSY`는 6b | BE-AI-09 |
| 레벨 0 파이프라인(`AGENT_MODE=pipeline`): `graph.py` 골격(`draft` → `check_draft` → 규칙 노드 → `gate` → 통과 END / 위반 시 바로 `fallback`)으로 `prompts/day_card.md` 단발 생성 → pydantic 검증(실패 시 남은 12초 이상이면 1회 재시도) → `check_draft` → `gate` → 필요하면 결정적 폴백 → `READY`. interrupt가 없으므로 체크포인터 없이 컴파일한다. 6b는 규칙 노드 자리에 `decide`를 연결하고 체크포인터를 붙인다 | BE-AI-11, 13 |
| 의존성 고정(`==`)과 빌드: 에이전트 함수 전용 requirements.txt, `sam build --use-container`, 패키지 크기(`du -sh .aws-sam/build/AiCardFunction`)·Init Duration 기록(Q18). 레벨 0도 같은 의존성을 쓰므로 빌드·import 문제를 여기서 먼저 드러낸다 | BE-AI-14 |
| 타임아웃 규칙(Lambda 27s, connect 2 / read 8, 남은 12초 미만이면 호출 중단)과 에러 코드 | BE-AI-05 |
| `AiQuota`(하루 10세션, 429 `AI_QUOTA_EXCEEDED`), 경로별 스로틀 | BE-AI-10, BE-INFRA-06 |
| `POST /plans`의 `authoring.aiSessionId`로 AI 세션 지표 계산 | BE-PLAN-03 |
| 메트릭 `AiLatencyMs`, `AiFailure`, `AiInvalidOutput`, 통합 테스트 T12 | BE-OBS-01 |

- 산출물: 레벨 0으로 동작하는 card-sessions API, `AiCardFunction` 패키지 크기와 Init Duration 기록. 프론트가 이 시점부터 실제 API로 3-1 화면을 붙인다.

#### 6b. 에이전트 그래프(LangGraph), 게이트, 잠금

| 작업 | 요구사항 |
|---|---|
| `agent/graph.py`·`nodes.py`: LangGraph `StateGraph`(draft → check_draft → decide ↔ check_draft/ask_planner/gate → fallback/END), `pause` 노드(시간 가드), 세션 예산 조건부 엣지, 도구 호출 없는 응답 재촉 1회, 마지막 호출·`finishRequested`에서 `ask_planner` 제거, 게이트 거부 2회 후 결정적 폴백, LLM 호출 오류 → `fallback`(후보 카드 있음)/END(없음), `decide` 안의 `ask_planner` 거부 판정, 레벨 0 분기 | BE-AI-04, 08, 13 |
| 체크포인터: `HaruAgentCheckpoints` SAM 리소스, `DynamoDBSaver(consistent_read=True, ttl_seconds)`, `durability="sync"`, 재개 전 `get_state` 확인(Q19), 체크포인트 삭제 정책(Q20) | BE-AI-14, BE-INFRA-07 |
| `agent/tools.py`: `check_draft`, `ask_planner` 거부 규칙(`decide` 노드에서 호출), `submit_card`(서버 게이트 판정). 모두 순수 함수 | BE-AI-04 |
| `prompts/day_card_agent.md`: 데이터 블록(`<story>`, `<answer>`), 질문 우선순위, "빼기보다 원문 표현으로 바꾸기" | BE-AI-03 |
| 턴 잠금 조건부 갱신(`ownerId = :me` 포함, 잠금과 함께 `status = WORKING`), `expectedTurn` 멱등, `AI_SESSION_BUSY` | BE-AI-09 |
| (선택) 공급자가 OpenAI일 때 `/v1/moderations` 참고 플래그 | BE-AI-12 |
| 메트릭 `AgentModelCalls`, `AgentTokens`, `AgentTurnMs`, `AgentQuestionsAsked`, `AgentGateRejected`, `AgentDegraded`, `AgentSelfRepairs`, `AgentCheckpointBytes` | BE-OBS-01 |
| 통합 테스트 T18–T22, 오프라인 평가 15건(`scripts/eval_agent.py`) | BE-OBS-04 |

- 개발은 로컬 Docker와 Fake 채팅 모델(`LLM_PROVIDER=fake`)로 먼저 하고, 3단계(T3–T7)를 밀어내지 않는다. Fake 시나리오: `happy`, `ask_once`, `no_tool_call`, `no_tool_call_twice`, `bad_json_args`, `unknown_tool`, `parallel_calls`, `gate_reject_twice`, `timeout`, `http_500`, `http_401`, `http_400`. 호출마다 받은 메시지와 도구 목록을 기록해, 마지막 호출과 `finishRequested`에서 `ask_planner`가 빠졌는지 단위 테스트로 확인한다.
- 오프라인 평가: `tests/fixtures/stories.json` 15건(데모 사연, 비공개 장소·연락처·인젝션 함정 사연 포함)을 같은 모델로 ① 기존 단발 프롬프트 ② 레벨 0 ③ 에이전트에 돌려 `NOT_IN_SOURCE`·`MISSING`·`PRIVACY` 잔존 수, 호출 수, 지연을 비교한다. 에이전트가 낫지 않은 유형도 그대로 적는다. 결과로 공급자·모델(Q6)을 고른다.
- **리허설 전에 T18–T22를 통과하지 못하면 `AGENT_MODE=pipeline`(레벨 0)으로 출시한다.** 계약과 화면은 그대로이고 질문 말풍선만 빠진다. 이 전환은 `decide`(에이전트 동작)의 불안정을 피할 뿐이고, 레벨 0도 같은 LangGraph 의존성 위에서 돌므로 의존성 문제는 피하지 못한다. 그래서 빌드·import·콜드 스타트는 6a에서 먼저 검증한다.
- **프론트 연동 S4**: 6a 배포 후 프론트가 AI 정리 화면(3-1)을 실제 API로 전환한다. 6b는 계약 변경 없이 서버 내부만 교체하고, 6b 배포 후 프론트가 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로를 실제 API로 다시 확인한다.

### 7단계. 웹 배포, 관측, 시연 준비

| 작업 | 요구사항 |
|---|---|
| 웹 버킷 + CloudFront(OAC, SPA 에러 응답 → `/index.html`), CORS Origin에 배포 도메인 추가 | BE-INFRA-02, 03 |
| 알람(API 5xx, Lambda Errors/Duration, `AgentTokens`), AWS Budgets 월 알림, 대시보드(AI 세션 상태 분포 포함) | BE-OBS-02, 03 |
| `scripts/seed.py`, `scripts/report.py` | BE-OBS-04, 05 |
| `demo` 환경으로 빈 계정 배포 → T1–T22 전체 실행 → 결과 로그 보관 | BE-INFRA-01 |
| 프론트와 90초 시연 리허설 | §6 인수 기준 |
| AI 시연 안정장치: 리허설 10회 이상으로 질문 1개 경로 재현 확인, 선택한 공급자에서 턴 p95와 `read_timeout` 8초 안 완료 여부 측정(Q12), 시연 직전 1회 호출로 콜드 스타트 제거(LangGraph import 포함), 지연이 길면 `AGENT_MODE=pipeline` 전환 | BE-AI-05, 11 |

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

프론트의 로컬 백엔드(docker compose) 연결 모드와 로컬 인증 방식은 10.6을 따른다.

계약을 바꿔야 할 때: (1) `MVP.md` §7 수정 PR → (2) 프론트 담당 리뷰 승인 → (3) 백엔드 구현 PR. 프론트 MSW 핸들러도 같은 시점에 고친다.

## 5. 테스트 전략

| 수준 | 도구 | 대상 |
|---|---|---|
| 단위 | pytest, Fake 채팅 모델, botocore Stubber, (필요 시) 로컬 에뮬레이터 | 입력 검증 경계값, `find_not_in_source`, `scan_privacy`, 직렬화 허용 목록, 트랜잭션 분기 로직(TransactionConflict는 Stubber로 주입), 에이전트 그래프(예산·게이트·폴백·`pause`·interrupt 재개, Fake 시나리오), PreSignUp 핸들러, 사진 처리(샘플 이미지·GPS EXIF 이미지·손상 파일) |
| 로컬 통합 | pytest + requests, docker compose(10장) | 세션 API 흐름(체크포인터 포함), 조건식 로직(동시 이어받기 1건 성공·나머지 409, 완료 멱등), 사진 업로드 → 알림 → 폴러 → 처리 흐름. **T 시나리오의 합격 판정에는 쓰지 않는다** |
| 통합 | pytest + requests, `concurrent.futures` | 실제 AWS `dev` 스택 대상 T1–T22. 테스트용 Cognito 사용자를 `AdminCreateUser`로 만들고 ID 토큰을 발급해 쓴다. |
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

T9(`FAULT_INJECT=photo`), T12(`LLM_API_KEY_PARAM`이 잘못된 키를 가리키게 함), T21(`FAULT_INJECT=ai-slow`)은 환경 설정을 바꿔야 하므로 별도 마커(`@pytest.mark.fault`)로 분리해 마지막에 실행한다.

## 6. 일정 표 (팀이 채움)

| 단계 | 담당 | 시작 | 완료 | 연동 지점 |
|---|---|---|---|---|
| 1. SAM 골격·인증 | | | | S1 |
| 2. 계획·멱등 키 | | | | |
| 3. 이어받기·포기·완료 | | | | |
| 4. 기록(글)·수신함 | | | | S2 |
| 5. 사진 파이프라인 | | | | S3 |
| 6a. AI 레벨 0·계약·AiQuota | | | | S4 |
| 6b. AI 에이전트 그래프(LangGraph)·T18–T22·오프라인 평가 | | | | |
| 7. 웹 배포·관측·시연 | | | | S5 |

병렬 작업이 가능한 조합: 2단계 이후 "3단계(트랜잭션)"와 "6a단계(AI 레벨 0·계약)"는 서로 의존하지 않으므로 인원이 2명 이상이면 나눠서 진행한다. 6b는 Fake 채팅 모델로 개발하되 3단계 T3–T7을 우선한다. 5단계의 사진 처리 로직(Pillow 축소·메타데이터 제거)도 3단계와 병행해 단위 테스트까지 먼저 만들 수 있다.

추가 공수(기존 6단계 대비, 인일): 6a +1, 6b +2.5, 오프라인 평가 15건과 스크립트 +1. 계획 대표 사진·`GET /me/plans`를 이번 일정에서 빼서 약 1.5~2인일을 상쇄한다. 선택 작업: `check_invariants` +0.5~1, P1 질문 도우미 약 +3. 로컬 개발 환경(compose·init·어댑터)과 LangGraph 의존성 빌드와 크기·콜드 스타트 측정(6a), 체크포인터 연결(6b) 공수는 1단계·6a 시작 때 산정해 이 표에 더한다.

## 7. 작업 규칙 (제안)

- 브랜치: `main`은 배포 가능한 상태 유지. 작업은 `feat/<단계>-<요약>`(예: `feat/3-claim-transaction`) 브랜치에서 하고 PR로 병합한다.
- PR: 관련 요구사항 ID와 통과한 테스트 시나리오(T 번호)를 본문에 적는다. 리뷰어 1명 이상.
- 커밋 메시지: `feat:`, `fix:`, `test:`, `infra:`, `docs:` 접두어.
- 비밀값: 계정 ID·토큰·테스트 비밀번호·LLM API 키는 저장소에 넣지 않는다. 로컬 값은 `.env`(gitignore, 예시는 `.env.example`), AWS 값은 SSM Parameter Store에 둔다.
- 이미지·패키지 버전: compose 이미지 태그와 에이전트 의존성은 정확한 버전으로 고정한다(`latest`, 범위 지정 금지. 단 `boto3>=1.43.64`는 하한만).

## 8. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| 트랜잭션 분기 오류로 이어받기 2명 확정 또는 수신함 중복 | 핵심 목표(G1, G2) 실패 | 3단계에서 T3–T7을 먼저 통과시키고, 이후 단계 PR마다 `test_concurrency.py` 재실행 |
| 외부 LLM 공급자로 사연 전송. 공급자마다 처리 국가·보존·학습 정책이 다름 | 동의 문구 누락, 개인정보 위험 | 6a 시작 전 Q6·Q7 확정. 무료 티어 중 입력을 학습·검토에 쓰는 곳은 제외, 동의 문구에 공급자명·처리 국가·보존 기간 기재, 마스킹 여부 결정(Q10) |
| 공급자·서버마다 `tool_choice`·`parallel_tool_calls`·`max_tokens`·usage 지원이 다름 | 제출 강제 실패, 예산 계산 오류 | 그래프는 `tool_choice`에 의존하지 않고 도구 목록 조정·재촉 1회·결정적 폴백으로 처리. 서버 검증(허용 목록, 인자 파싱, 스키마). 공급자별 설정 키와 실측(Q9) |
| AI 턴 지연으로 30초 초과 | 코드 없는 500 | Lambda 27s, read timeout 8s, 남은 시간 12초 미만이면 모델 호출 없이 `WORKING` 응답 → 프론트 자동 continue, 연속 2회면 `finish`(BE-AI-05). 리허설 p95 측정, 초과하면 레벨 0 |
| 경량 모델의 한국어 도구 호출 불안정(반복 호출, 제출 미루기, 텍스트로만 응답) | 카드 품질 저하, 호출 증가 | 도구 호출 없는 응답 재촉 1회, 마지막 호출에서 ask 제거, 세션 예산, 결정적 폴백, 오프라인 평가로 모델 선정 |
| LangGraph 의존성으로 패키지 크기(압축 해제 약 90~170MB 추정)·콜드 스타트 증가, 네이티브 wheel 아키텍처 불일치 | 첫 턴 지연, import 실패. 레벨 0도 LangGraph에 의존하므로 `AGENT_MODE=pipeline` 전환으로 회피되지 않음 | 에이전트 함수에만 의존성 포함, `sam build --use-container`, 6a(레벨 0 첫 배포)에서 빌드·import·크기·Init Duration을 먼저 검증(Q18), 메모리 1024MB부터, 시연 직전 워밍 호출 |
| LangGraph 릴리스 주기가 빠르고 interrupt 재개 의미(노드 재실행, 인덱스 짝짓기)를 잘못 쓰기 쉬움 | 재개 시 중복 실행·상태 꼬임 | 버전 `==` 고정, `interrupt()` 앞에 부수 효과 금지, 시간 가드는 전용 `pause` 노드, 재개 전 `get_state` 확인(Q19), interrupt 재개 단위 테스트 |
| 체크포인트 아이템이 400KB를 넘음(S3 오프로드 끔) | 턴 저장 실패 | 세션 예산으로 상한, `AgentCheckpointBytes` 지표, 필요하면 오프로드를 별도 버킷으로 켬 |
| LangSmith 추적이 실수로 켜져 사연 원문이 외부로 나감 | 개인정보 유출 | 두 환경 변수를 `false`로 명시, API 키 미설정, `tracing_context(enabled=False)`(BE-AI-15) |
| 에이전트 작업이 정합성 핵심(G1–G4) 일정을 밀어냄 | 핵심 목표 지연 | 3단계 T3–T7 우선. 6a는 병행, 6b는 Fake 채팅 모델로 개발. 마감 시 에이전트 동작이 불안정하면 `AGENT_MODE=pipeline`으로 출시(의존성 문제는 이 전환으로 피할 수 없으므로 6a에서 먼저 검증) |
| 에이전트가 레벨 0보다 낫지 않음 | 에이전트 도입 근거 약화 | 오프라인 평가 표로 비교. 차이가 없으면 질문 기능만 남기고 `decide` ↔ `check_draft` 반복을 줄인다 |
| 명사 휴리스틱 오탐(빵집·빵 가게)으로 과잉 수정·불필요한 질문 | 작성 시간 증가 | "빼기보다 원문 표현으로 바꾸기" 프롬프트, 정규화 사전 픽스처, 메모 '다시 넣기' |
| 규칙 기반 개인정보 탐지 누락 | 공개 Plan에 개인정보 노출 | 탐지는 보조라고 명시. 사연 입력 안내와 계획자 확인 유지, 누락 사례를 픽스처에 추가 |
| 로컬 에뮬레이터와 실제 AWS의 동작 차이(TransactionConflict 미재현(DynamoDB Local 기준, Floci 미확인), GSI 즉시 일관, JWT authorizer 부재 등) | 로컬 통과 후 AWS에서 실패 | T 판정은 AWS 스택에서만, 10.4의 AWS 전용 검증 항목을 단계별 `dev` 배포 때 확인 |
| Floci가 2026-02에 생긴 젊은 프로젝트 | 로컬 환경 중단 | 이미지 태그 고정, 대체 구성 A(10.2) 준비 |
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
- [ ] 설정값(`LLM_*`, `AGENT_MODE`, 기한 일수 등) 파라미터 분리, LangSmith 환경 변수 `false` 명시
- [ ] `docker compose up`으로 로컬 개발 환경 기동, 단위·로컬 통합 테스트 통과
- [ ] 오프라인 평가(`scripts/eval_agent.py`, 15건) 결과표 보관

## 10. 로컬 개발 환경 (docker compose)

예산 문제로 개발 단계에서는 AWS 서비스 대신 AWS 호환 소프트웨어의 Docker 컨테이너를 쓴다. 로컬 환경은 **개발과 단위·로컬 통합 테스트용**이다. 데모, 시범 운영, 동시성 통합 테스트(T3/T5 등)를 포함한 T1–T22 판정은 실제 AWS에 SAM으로 배포한 스택에서 한다. 데모까지 Docker로 운영하는 대안은 [`REQUIREMENTS.md`](./REQUIREMENTS.md) Q15.

### 10.1 구성

| AWS 서비스 | 로컬 대체 | 제약 |
|---|---|---|
| DynamoDB(HaruTable, `HaruAgentCheckpoints`) | `floci/floci:2.5.0-compat` (MIT, 계정·토큰 없음) | TransactionConflict를 재현한다는 언급이 없고, 실제 병렬성·GSI 지연은 재현되지 않는다. TTL은 60초 주기 스윕. `DynamoDBSaver` 동작은 Q21 |
| S3(presigned PUT/GET, ObjectCreated → SQS, CORS, 수명 주기) | 같은 Floci 컨테이너 | 서명 검사는 `FLOCI_AUTH_VALIDATE_SIGNATURES=true`일 때만. presign은 브라우저가 접근할 수 있는 `http://localhost:4566` 호스트와 경로 방식 주소(`addressing_style='path'`)로 만든다(host도 서명에 들어간다) |
| SQS(+DLQ, maxReceiveCount, ApproximateReceiveCount) | 같은 Floci 컨테이너 | Lambda 이벤트 소스 매핑은 어댑터의 폴러로 대신한다 |
| Cognito User Pool | 같은 Floci 컨테이너 + 시드 사용자(`AdminCreateUser`, `AdminSetUserPassword --permanent`, `nickname`) | 트리거는 Floci에 Lambda를 배포해야 동작하므로 PreSignUp은 단위 테스트와 AWS에서 확인. `nickname` 클레임 포함 여부는 Q17 |
| API Gateway HTTP API(JWT authorizer) + Lambda | 자체 빌드 Python 3.12 HTTP 어댑터(`local/adapter`) | 실제 authorizer·IAM·콜드 스타트·레이어 호환은 AWS에서 확인 |
| CloudWatch EMF | stdout 로그(`AWS_EMF_ENVIRONMENT=Local`) | 지표 이름·차원 오타만 잡는다 |
| CloudFront + 웹 버킷 | 호스트의 Vite dev 서버(5173) | OAC·SPA 대체 응답·캐시는 AWS에서만 |
| LLM | Fake 채팅 모델(기본), 외부 OpenAI 호환 API(개발 키), 선택 `ollama` 프로필 | 10.5 |
| SSM Parameter Store(LLM 키) | `.env`의 `LLM_API_KEY` | |

어댑터(`local/adapter`)가 하는 일
1. HTTP 요청을 API Gateway v2(payload 2.0) 이벤트로 바꿔 같은 Lambda 핸들러를 호출한다. 핸들러에 넘기는 context는 `get_remaining_time_in_millis()`를 제공한다(기본 27초, 12초 규칙 확인용).
2. `Authorization` ID 토큰을 검증해 `iss`, `aud`(= client id), `exp`, `token_use=id`를 확인하고, 결과를 `requestContext.authorizer.jwt.claims`에 넣는다. 실패하면 401 `UNAUTHORIZED`. `STAGE=local`에서는 Floci User Pool JWKS와 dev JWT 공개키를 둘 다 신뢰한다. 브라우저는 `http://localhost:4566`에서, 어댑터는 `http://aws:4566`으로 접근하므로 `iss` 기대값과 JWKS URL은 `AWS_ENDPOINT_URL`에서 만들지 않고 별도 설정(`LOCAL_JWT_ISSUER`, `LOCAL_JWKS_URL`)으로 둔다. init 스크립트가 이 값을 출력한다.
3. SQS 폴러: `ReceiveMessage(AttributeNames=[All])`로 받은 메시지를 Lambda SQS 이벤트 형태(`attributes.ApproximateReceiveCount` 포함)로 만들어 사진 처리 핸들러에 넘긴다. 성공하면 삭제하고, 예외면 그대로 둔다. 가시성 타임아웃 뒤 다시 받고, maxReceiveCount를 넘으면 에뮬레이터가 DLQ로 옮긴다.
4. CORS 허용 Origin `http://localhost:5173`(또는 Vite 프록시).
5. `STAGE=local`에서만 켜지는 dev JWT 발급 엔드포인트(자동화 테스트용, 로컬 RSA 키로 `sub`/`nickname`/`aud`/`iss` 서명). 이 키의 공개키를 2번 검증의 신뢰 목록에 넣으므로 dev JWT도 같은 검증을 통과한다.

`sam local start-api`는 HTTP API JWT authorizer를 실행하지 않고(Lambda authorizer만 지원, `requestContext.authorizer.jwt.claims`가 비어 있음) 요청마다 컨테이너를 띄워 느리므로 기본 경로로 쓰지 않는다. LocalStack은 2026-03-23부터 계정·인증 토큰이 필요하고 무료 Hobby 플랜(비상업용)에 HTTP API v2와 Cognito가 없어 기본에서 뺀다. MinIO는 무료 Docker 이미지 배포가 중단됐고 SQS 알림 대상이 없어 쓰지 않는다.

```yaml
# docker-compose.yml (예시. 태그는 작성 시점에 고정, latest 금지)
services:
  aws:            # DynamoDB·S3·SQS·Cognito
    image: floci/floci:2.5.0-compat       # 초기화 스크립트에서 aws CLI를 쓰려면 -compat
    ports: ["4566:4566"]
    environment:
      FLOCI_STORAGE_MODE: hybrid          # 재시작 후에도 데이터 유지(테스트는 memory)
      FLOCI_AUTH_VALIDATE_SIGNATURES: "true"   # presigned 서명/ContentLength 검사
    volumes:
      - ./local/init:/etc/floci/init/ready.d   # HaruTable+GSI+TTL, HaruAgentCheckpoints(PK/SK, TTL ttl),
                                               # 버킷+CORS+수명 주기+알림, 큐+DLQ(maxReceiveCount 3), 유저 풀+시드 사용자
      - floci-data:/app/data
  api:            # HTTP 어댑터 + SQS 폴러(같은 Lambda 핸들러)
    build: ./local/adapter                # python:3.12 + 프로젝트 requirements(에이전트 의존성 포함)
    ports: ["3000:3000"]
    env_file: .env                        # LLM_API_KEY 등(gitignore)
    environment:
      STAGE: local
      AWS_ENDPOINT_URL: http://aws:4566
      PUBLIC_S3_ENDPOINT: http://localhost:4566   # presign 전용
      CHECKPOINT_DDB_ENDPOINT: http://aws:4566
      CHECKPOINT_TABLE: HaruAgentCheckpoints
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_REGION: ap-northeast-2
      AWS_EMF_ENVIRONMENT: Local
      LANGSMITH_TRACING: "false"
      LANGCHAIN_TRACING_V2: "false"
      LLM_PROVIDER: ${LLM_PROVIDER:-fake}
      LLM_BASE_URL: ${LLM_BASE_URL:-}     # OpenAI / 호환 API / 맥 네이티브 Ollama http://host.docker.internal:11434/v1
                                          # / llm-local 프로필 컨테이너 http://ollama:11434/v1
      LLM_MODEL: ${LLM_MODEL:-}
    depends_on: [aws]
  ollama:         # 선택(Linux + NVIDIA, NVIDIA Container Toolkit 필요): docker compose --profile llm-local up
    image: ollama/ollama:0.40.17
    profiles: ["llm-local"]
    environment:
      OLLAMA_CONTEXT_LENGTH: "16384"      # 기본 4k로는 프롬프트·도구 스키마·히스토리가 넘칠 수 있다
    volumes:
      - ollama-models:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
volumes: { floci-data: {}, ollama-models: {} }
# 프런트: 호스트에서 `npm run dev`(Vite 5173), API = http://localhost:3000
```

### 10.2 대체 구성

- 대체 구성 A(Floci에 문제가 생길 때): `amazon/dynamodb-local:3.3.2`, `softwaremill/elasticmq-native:1.7.1`, `localstack/localstack:4.14`(S3만, 토큰 없음, 동결 버전, 보안 패치 없음)에 어댑터의 dev JWT를 붙인다. LocalStack S3에서 ElasticMQ로 버킷 알림을 직접 보낼 수 없으므로 S3와 SQS를 둘 다 LocalStack에 두거나, 업로드 확인 시 어댑터가 S3 이벤트 형태의 메시지를 SQS에 넣는다. `DynamoDBSaver`의 업스트림 적합성 테스트가 `amazon/dynamodb-local`에서 돌므로 체크포인터는 이 구성에서 연결이 확인된 상태다.
- Floci만 바꾸고 싶을 때: `FLOCI_SERVICES_DYNAMODB_BACKEND=local`로 DynamoDB 요청을 DynamoDB Local에 넘길 수 있다.
- 대체 구성 B: 팀원 전원이 LocalStack Student 플랜(GitHub Student Developer Pack 인증, 사람마다 계정)을 받으면 LocalStack 하나로 HTTP API와 Cognito까지 쓸 수 있다(토큰 관리 필요).

### 10.3 설정 전환

같은 핸들러 코드를 쓰고 환경 변수만 바꾼다. boto3는 `AWS_ENDPOINT_URL`(서비스별 `AWS_ENDPOINT_URL_S3` 등)로 엔드포인트를 바꾼다.

| 키 | 로컬 | AWS(SAM) |
|---|---|---|
| `AWS_ENDPOINT_URL` | `http://aws:4566` | 정의하지 않음(SAM 환경 변수에 넣지 않음. 앱 코드가 아니라 SDK가 직접 읽는 표준 변수다) |
| `PUBLIC_S3_ENDPOINT` | `http://localhost:4566` (presign 전용) | 빈 값(코드가 `None`으로 처리) |
| `CHECKPOINT_DDB_ENDPOINT` | `http://aws:4566` | 빈 값(코드가 `None`으로 바꿔 `DynamoDBSaver`에 넘김) |
| `LOCAL_JWT_ISSUER` / `LOCAL_JWKS_URL` | init 스크립트가 출력한 Floci User Pool 값 | 정의하지 않음(어댑터 전용) |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | 더미 `test` | Lambda 실행 역할 |
| `LLM_PROVIDER` | `fake`(기본) / `openai_compat` | `openai_compat` |
| `LLM_BASE_URL` / `LLM_MODEL` | 개발 키 공급자 또는 Ollama | `samconfig.toml` 파라미터(Q6) |
| `LLM_API_KEY` / `LLM_API_KEY_PARAM` | `.env`의 `LLM_API_KEY` | SSM SecureString 이름 |
| `LLM_READ_TIMEOUT_S` | `8`(네이티브 Ollama는 `30`~`60`, 어댑터 context 시간도 함께 늘린다) | `8` |
| `AWS_EMF_ENVIRONMENT` | `Local` | 정의하지 않음(SDK 표준 변수) |
| `LANGSMITH_TRACING` / `LANGCHAIN_TRACING_V2` | `false` / `false` | `false` / `false` |

### 10.4 AWS(SAM 배포)에서만 검증하는 항목

로컬에서 재현되지 않으므로 해당 단계의 `dev` 배포 때 확인한다.

1. HTTP API JWT authorizer의 실제 동작: aud·만료 검사, 401 응답 형태, API Gateway 수준의 CORS preflight와 오류 응답의 CORS 헤더
2. Cognito: PreSignUp 트리거 연결(autoConfirmUser/autoVerifyEmail), `nickname` 필수 속성, 실제 ID 토큰 클레임과 만료
3. DynamoDB: **T3/T5 동시성 판정**(TransactionConflict, 실제 병렬성과 지연), GSI 최종 일관성 지연, 실제 TTL 삭제 시점, 스로틀링과 온디맨드 한도, IAM 최소 권한(`LeadingKeys` 포함), 체크포인트 테이블의 `consistent_read` 재개
4. S3: presigned PUT의 ContentType/ContentLength 서명 강제, CloudFront Origin 기준 버킷 CORS, 퍼블릭 차단과 OAC, 7일 수명 주기 만료, 버킷 알림과 큐 정책(순환 의존성 회피 구성)
5. SQS와 Lambda 이벤트 소스 매핑: 배치, 가시성 타임아웃(6배), maxReceiveCount 3에서 DLQ 이동, `ApproximateReceiveCount >= 3` 처리, DLQ 알람(T9)
6. Lambda 런타임: Python 3.12와 Pillow 레이어·LangGraph 네이티브 wheel의 아키텍처 호환, 메모리와 타임아웃, 콜드 스타트(Init Duration), 환경 변수와 역할, SSM 키 읽기
7. CloudWatch: EMF 메트릭 추출, 대시보드, 알람
8. CloudFront: OAC, SPA 대체 응답, 캐시 무효화, HTTPS 도메인
9. SAM 템플릿 자체의 배포 가능성(로컬에서는 `sam validate --lint`까지만)
10. 데모에 쓸 실제 LLM 공급자의 도구 호출 동작, 지연, 요금 한도, 데이터 처리 위치(로컬 Ollama로 대신할 수 없다)

### 10.5 로컬 LLM

- 기본은 Fake 채팅 모델(`LLM_PROVIDER=fake`, `FAKE_LLM_SCENARIO`)이다. 실모델 확인은 외부 OpenAI 호환 API(개발 키)나 로컬 Ollama로 한다(Q16).
- Ollama `/v1`은 `tools`는 지원하지만 `tool_choice`는 지원하지 않는다. `LLM_TOOL_CHOICE`는 비워 두고, 도구 호출 없는 응답은 재촉 1회 → 결정적 폴백 경로로 처리된다. `max_completion_tokens`가 무시될 수 있으므로 `LLM_EXTRA_BODY={"max_tokens":800}`을 함께 보낸다(Q9). 키가 필요 없지만 `LLM_API_KEY`에는 더미 값(`ollama`)을 넣는다.
- macOS Docker Desktop에서는 GPU를 쓸 수 없다. 맥에서는 Ollama를 네이티브로 설치(Metal)하고 컨테이너에서 `http://host.docker.internal:11434/v1`로 붙는다. Linux + NVIDIA는 NVIDIA Container Toolkit을 설치하고 `llm-local` 프로필 컨테이너(GPU 예약 포함)로 GPU를 쓸 수 있다. 이때 `LLM_BASE_URL`은 `http://ollama:11434/v1`이다. CPU만으로는 호출 1회가 read 8초와 Lambda 27초를 넘기기 쉬우므로(추정) 전체 세션 확인에 쓰지 않는다.
- 기본 컨텍스트가 4k(VRAM 24GiB 미만)라 시스템 프롬프트·도구 3개 스키마·사연·히스토리를 넣으면 넘칠 수 있다. 서버를 `OLLAMA_CONTEXT_LENGTH=16384` 정도로 띄운다.
- 모델 후보(Ollama `tools` 태그): `qwen2.5:3b`(1.9GB)·`qwen2.5:7b`(4.7GB)가 공식 지원 언어에 한국어를 명시한다. `llama3.2:3b`는 공식 지원 언어에 한국어가 없고, `exaone3.5`는 `tools` 태그가 없어 도구 루프에 쓸 수 없다.

### 10.6 프론트 연동 모드

| 모드 | API | 인증 | 쓰는 때 |
|---|---|---|---|
| MSW 목업 | 브라우저 안 목업 | 목업 | S0부터, 계약 기준 화면 개발 |
| 로컬 백엔드 | `http://localhost:3000`(어댑터) 또는 Vite 프록시 | Floci Cognito(`http://localhost:4566`)의 시드 사용자, 또는 dev JWT | 백엔드 기능이 로컬에 올라온 뒤 실제 흐름 확인 |
| `dev`/`demo` 스택 | SAM 출력 `ApiUrl` | 실제 Cognito(`UserPoolId`, `UserPoolClientId`) | S1–S5 연동, 리허설, 시연 |

- 로컬 백엔드 모드에서 프론트는 API 주소와 Cognito 설정(User Pool ID, App Client ID, 엔드포인트)을 init 스크립트가 출력한 로컬 값으로 바꾼다. 로그인 SDK가 사용자 지정 엔드포인트를 지원하는지(Amplify v6의 `userPoolEndpoint` 등)는 프론트 저장소에서 확인한다(Q17). 지원하지 않으면 로컬 모드에서는 dev JWT를 쓴다.
- 사진 업로드는 presigned URL의 호스트(`http://localhost:4566`)로 직접 PUT한다. init 스크립트가 사진 버킷 CORS에 `http://localhost:5173`을 넣는다.
- 로컬 백엔드 모드의 동작이 계약 일치의 근거는 아니다. 계약 비교(S2·S3·S4)는 `dev` 스택 응답으로 한다.
