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
| [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) | 개발계획서 (7단계 작업(6단계는 6a·6b), 프론트 연동 지점 S0–S5, 테스트 전략) |

## 관련 저장소

- 프론트엔드: [knu-aws-2026-001/frontend](https://github.com/knu-aws-2026-001/frontend)
- **이 저장소의 [`MVP.md`](./MVP.md) §7이 API 계약의 기준입니다.** 프론트엔드는 이 계약으로 MSW 목업을 만들어 병행 개발합니다. 응답 형태를 바꿀 때는 `MVP.md`를 먼저 고치고 프론트 담당의 리뷰를 받은 뒤 코드에 반영합니다.

## 아키텍처

```
브라우저 ──▶ CloudFront ──▶ S3 (정적 파일)
   ├─ Cognito User Pool (로그인 → ID 토큰)
   └─▶ API Gateway (HTTP API, JWT authorizer)
          └─▶ Lambda ──▶ DynamoDB (단일 테이블)
                 ├─▶ Bedrock Converse tool use (하루 카드 에이전트, Global CRIS) + Guardrails(보조)
                 └─▶ S3 사진 버킷 presigned URL 발급

사진: 브라우저 ──PUT──▶ S3 ──▶ SQS ──▶ 사진 처리 Lambda (검사·축소·메타데이터 제거 → 완료 트랜잭션)
                                 └─▶ DLQ ──▶ CloudWatch 알람
```

자세한 구성은 [`MVP.md`](./MVP.md) §4를 참고합니다.

| 영역 | 선택 |
|---|---|
| 런타임 | Python 3.12 (boto3, pydantic, Pillow 레이어) |
| IaC | AWS SAM (`dev` / `demo` 환경) |
| 리전 | `ap-northeast-2` (Bedrock 경량 모델은 Global CRIS 추론 프로필로 호출, 국외 리전에서 처리될 수 있음) |
| AI 에이전트 | Converse API tool use 루프를 boto3로 직접 구현 (`AGENT_MODE=pipeline`이면 레벨 0) |

## 진행 상황

[`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 3장의 단계입니다.

- [ ] 1단계. SAM 골격, Cognito, HTTP API + JWT authorizer, DynamoDB, `GET /me`
- [ ] 2단계. 계획 생성·목록·상세, 멱등 키
- [ ] 3단계. 이어받기·포기·완료 트랜잭션 + 동시성 통합 테스트
- [ ] 4단계. 방문 기록(글), 수신함·읽음·답장
- [ ] 5단계. 사진 버킷·SQS·DLQ·처리 Lambda
- [ ] 6a단계. AI 레벨 0 파이프라인, card-sessions 계약, AiQuota
- [ ] 6b단계. AI 에이전트 루프·서버 게이트·턴 잠금, T18–T22, 오프라인 평가
- [ ] 7단계. CloudFront·웹 버킷, 알람·대시보드, seed/report 스크립트, 시연 리허설

## 사전 준비

- AWS CLI (배포할 계정의 자격 증명 설정)
- AWS SAM CLI
- Python 3.12
- Docker (Pillow 레이어를 `sam build --use-container`로 빌드할 때 필요)
- Amazon Bedrock 모델 접근 권한 (6a단계 전에 콘솔에서 신청). `aws bedrock list-inference-profiles --region ap-northeast-2`로 Global CRIS 프로필을 확인하고, SCP가 목적지 리전을 막지 않는지 점검

## 배포

1단계 이후 추가

배포가 끝나면 스택 출력값 `ApiUrl`, `UserPoolId`, `UserPoolClientId`, `WebBucketName`, `DistributionId`, `DistributionDomain`을 프론트엔드 담당에게 전달합니다.

## 테스트

1단계 이후 추가

통합 테스트(T1–T22)는 배포된 `dev` 스택을 대상으로 실행합니다. 동시성·멱등성은 로컬 목으로 검증할 수 없기 때문입니다.

## 데모 데이터

1단계 이후 추가 (`scripts/seed.py`로 데모 계정 지연·수현·민준과 예시 계획을 생성할 예정)

## 작업 규칙

자세한 내용은 [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 7장을 따릅니다.

- `main`은 항상 배포 가능한 상태로 유지하고, 작업은 `feat/<단계>-<요약>` 브랜치(예: `feat/3-claim-transaction`)에서 PR로 병합합니다.
- PR 본문에 관련 요구사항 ID(`BE-*`)와 통과한 검증 시나리오(T 번호)를 적습니다.
- 커밋 메시지 접두어: `feat:`, `fix:`, `test:`, `infra:`, `docs:`
- AWS 계정 ID, 실제 리소스 ID, 토큰, 테스트 계정 비밀번호는 저장소에 올리지 않습니다. 로컬 `.env`(gitignore)나 SSM Parameter Store에 둡니다.
