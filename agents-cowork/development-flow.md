# Development Flow - Agent Orchestration

## 바이브코딩(Claude <-> Gemini 협업) 워크플로우

![Development Flow](./md-images/diagram-agent-orchestration.svg)

---

## 플로우 요약

| 단계 | 참여자 | 산출물 및 경로 (예시: 로또 서비스) |
|---|---|---|
| 1. 기획 | 사용자 + Main Claude + 기획자 에이전트 | `my-docs/agents-cowork/lotto-service-prd.md` |
| 2. 핸드오프 | Main Claude | `my-docs/agents-cowork/lotto-service-backend-gemini-handoff.md`<br>`my-docs/agents-cowork/lotto-service-frontend-gemini-handoff.md` |
| 3. 개발 | Gemini BE 세션, Gemini FE 세션 (독립) | 각 서비스 소스 코드 및 단위 테스트 |
| 4. 코드 리뷰 | Claude BE 에이전트, Claude FE 에이전트 | 리뷰 코멘트 및 리팩토링된 코드 |
| 5. 통합 리뷰 | 사용자 + Main Claude | 최종 승인 시그널 |
| 6. QA | Main Claude | 테스트 결과 리포트 |
| 7. 배포 준비 | - | 배포용 아티팩트 및 `my-docs/` 문서 업데이트 |

---

## 단계별 상세 프로세스

### 1. 토큰 최적화 및 문서화 전략
* **클로드(Claude)** 코드는 서브에이전트 기능과 성능이 우수하나, 토큰이 매우 부족한 제약 존재.
* 토큰 절약을 위해 기획 단계에서 **PRD** 및 이에 기반한 **BE/FE 핸드오프(Handoff)** 파일을 각각 마크다운(.md)으로 작성.
* 제미나이(Gemini)가 해당 문서들을 참조하여 개발을 진행함으로써 불필요한 토큰 소모 방지.

### 2. 병렬 개발 및 독립 세션
* 각 **BE와 FE는 독립된 세션**에서 핸드오프 문서만을 참조하여 코드를 병렬로 작성.
* 영역 간 간섭 최소화 및 개발 효율 극대화.

### 3. 클로드 서브에이전트 기반 코드 리뷰 및 리팩토링 (4단계)
* 4단계 코드 리뷰는 **클로드의 백엔드 및 프론트엔드 서브에이전트**가 전담.
* 서브에이전트 간 소통을 통해 전체 시스템 정합성 확보 및 최적의 구조로 리팩토링 진행.

### 4. 최종 컨펌 및 QA 피드백 루프
* 개발 및 리뷰 완료 후 **Main 에이전트와 사용자(나)**의 최종 컨펌 진행.
* 승인 후 **QA** 단계 진입, 실패 시 다시 **4단계**로 돌아가 보완 및 리팩토링 수행.

### 5. 완료 및 배포 준비
* QA 성공 시 **배포 준비 완료 시그널** 전송.
* 동시에 **my-docs**의 관련 문서들을 최신 상태로 업데이트하며 프로세스 종료.
