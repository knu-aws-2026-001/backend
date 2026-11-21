# 이어온 하루 — Backend

취소된 동네 산책을 이어받고, 그날의 이야기를 돌려주는 모바일 웹의 서버리스 API입니다.
AWS Skill Training 1조 프로젝트 **"취소된 하루를 양도합니다"** 의 백엔드 저장소입니다.

계획자가 남긴 산책 계획을 **한 명의 실행자만** 이어받고, 실행자의 방문 기록(글 + 선택 사진 1장)이 **정확히 한 번** 계획자의 수신함에 도착하도록 처리합니다.

## 문서

아래 순서로 읽으면 됩니다.

| 문서 | 내용 |
|---|---|
| [`MVP.md`](./MVP.md) | MVP 범위, 도메인 모델, DynamoDB 설계, **API 계약(§7)**, AI 하루 카드 정리 에이전트(§7.2), 사진 파이프라인, 검증 시나리오 T1–T22 |
| [`docs/REQUIREMENTS.md`](./docs/REQUIREMENTS.md) | 요구사항 명세서 (`BE-*` ID, 인수 기준, 확인이 필요한 사항) |
| [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) | 개발계획서 (7단계 작업(6단계는 6a·6b), 프론트 연동 지점 S0–S5, 테스트 전략, 로컬 개발 환경(10장)) |

## 관련 저장소

- 프론트엔드: [knu-aws-2026-001/frontend](https://github.com/knu-aws-2026-001/frontend)
- **이 저장소의 [`MVP.md`](./MVP.md) §7이 API 계약의 기준입니다.** 프론트엔드는 이 계약으로 MSW 목업을 만들어 병행 개발합니다. 응답 형태를 바꿀 때는 `MVP.md`를 먼저 고치고 프론트 담당의 리뷰를 받은 뒤 코드에 반영합니다.

## 아키텍처

```
브라우저 ──▶ CloudFront ──▶ S3 (정적 파일)
   ├─ Cognito User Pool (로그인 → ID 토큰)
   └─▶ API Gateway (HTTP API, JWT authorizer)
          └─▶ Lambda ──▶ DynamoDB (단일 테이블)
                 ├─▶ 하루 카드 에이전트 (LangGraph, 체크포인트 테이블)
                 │     └─▶ 외부 OpenAI 호환 LLM API (Chat Completions tools)
                 └─▶ S3 사진 버킷 presigned URL 발급

사진: 브라우저 ──PUT──▶ S3 ──▶ SQS ──▶ 사진 처리 Lambda (검사·축소·메타데이터 제거 → 완료 트랜잭션)
                                 └─▶ DLQ ──▶ CloudWatch 알람
```

자세한 구성은 [`MVP.md`](./MVP.md) §4를 참고합니다.

| 영역 | 선택 |
|---|---|
| 런타임 | Python 3.12 (boto3, pydantic, Pillow 레이어) |
| IaC | AWS SAM (`dev` / `demo` 환경) |
| 리전 | `ap-northeast-2` (LLM은 외부 공급자라 사연이 공급자 서버로 전송됨. 처리 국가·보존 정책은 공급자마다 다름) |
| AI 에이전트 | LangGraph `StateGraph` + `DynamoDBSaver` 체크포인터, `langchain-openai` `ChatOpenAI`로 OpenAI 호환 API 호출. 공급자는 `LLM_BASE_URL`·API 키·`LLM_MODEL` 설정으로 교체 (`AGENT_MODE=pipeline`이면 레벨 0) |
| 로컬 개발 | docker compose: AWS 호환 에뮬레이터(Floci) + HTTP 어댑터 + Fake LLM. 데모·시범 운영·T1–T22 판정은 실제 AWS(SAM) |

## 진행 상황

[`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 3장의 단계입니다.

- [ ] 1단계. SAM 골격, Cognito, HTTP API + JWT authorizer, DynamoDB, `GET /me`, 로컬 개발 환경 골격
- [ ] 2단계. 계획 생성·목록·상세, 멱등 키
- [ ] 3단계. 이어받기·포기·완료 트랜잭션 + 동시성 통합 테스트
- [ ] 4단계. 방문 기록(글), 수신함·읽음·답장
- [ ] 5단계. 사진 버킷·SQS·DLQ·처리 Lambda
- [ ] 6a단계. LLM 공급자 확정, LangGraph 의존성 고정·빌드·크기 측정, AI 레벨 0 파이프라인, card-sessions 계약, AiQuota
- [ ] 6b단계. LangGraph 에이전트 그래프·체크포인터·서버 게이트·턴 잠금, T18–T22, 오프라인 평가
- [ ] 7단계. CloudFront·웹 버킷, 알람·대시보드, seed/report 스크립트, 시연 리허설

## 사전 준비

- Docker / Docker Compose (로컬 개발 환경, `sam build --use-container`로 Pillow 레이어·에이전트 의존성을 빌드할 때)
- Python 3.12
- AWS CLI, AWS SAM CLI (`dev`/`demo` 스택 배포 시)
- OpenAI 호환 LLM API 키 (실모델 확인 시. 로컬은 `.env`의 `LLM_API_KEY`, AWS는 SSM Parameter Store SecureString). 키 없이도 Fake LLM(`LLM_PROVIDER=fake`)으로 개발할 수 있습니다
- (선택) Ollama: 맥은 네이티브 설치 후 `http://host.docker.internal:11434/v1`로 연결 (Docker Desktop for Mac은 GPU 미지원)

## 로컬 개발

1단계 이후 추가 (`docker compose up` → 에뮬레이터 init 스크립트가 테이블·버킷·큐·유저 풀·시드 사용자 생성 → API `http://localhost:3000`, 프론트 Vite `http://localhost:5173`)

- 같은 Lambda 핸들러를 HTTP 어댑터가 호출합니다. 엔드포인트는 `AWS_ENDPOINT_URL`, `PUBLIC_S3_ENDPOINT`, `CHECKPOINT_DDB_ENDPOINT`, `LLM_BASE_URL` 환경 변수로만 바꿉니다.
- LangSmith 추적은 끕니다(`LANGSMITH_TRACING=false`, `LANGCHAIN_TRACING_V2=false`, API 키 미설정).
- 로컬 환경은 개발·단위/로컬 통합 테스트용입니다. 동시성(T3/T5), JWT authorizer, IAM, 알람 등은 로컬에서 재현되지 않으므로 [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 10.4의 항목은 AWS 스택에서 확인합니다.

## 배포

1단계 이후 추가

배포가 끝나면 스택 출력값 `ApiUrl`, `UserPoolId`, `UserPoolClientId`, `WebBucketName`, `DistributionId`, `DistributionDomain`을 프론트엔드 담당에게 전달합니다.

## 테스트

1단계 이후 추가

단위 테스트와 로컬 통합 테스트는 docker compose 환경과 Fake LLM으로 실행합니다. 통합 테스트(T1–T22)의 판정은 실제 AWS에 배포한 `dev` 스택을 대상으로 합니다. 동시성·멱등성은 로컬 에뮬레이터로 검증할 수 없기 때문입니다.

## 데모 데이터

1단계 이후 추가 (`scripts/seed.py`로 데모 계정 지연·수현·민준과 예시 계획을 생성할 예정)

## 작업 규칙

자세한 내용은 [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 7장을 따릅니다.

- `main`은 항상 배포 가능한 상태로 유지하고, 작업은 `feat/<단계>-<요약>` 브랜치(예: `feat/3-claim-transaction`)에서 PR로 병합합니다.
- PR 본문에 관련 요구사항 ID(`BE-*`)와 통과한 검증 시나리오(T 번호)를 적습니다.
- 커밋 메시지 접두어: `feat:`, `fix:`, `test:`, `infra:`, `docs:`
- AWS 계정 ID, 실제 리소스 ID, 토큰, 테스트 계정 비밀번호, LLM API 키는 저장소에 올리지 않습니다. 로컬 `.env`(gitignore)나 SSM Parameter Store에 둡니다.
