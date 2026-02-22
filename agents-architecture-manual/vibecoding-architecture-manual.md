# AI 주도 개발 — 디렉토리 아키텍처 매뉴얼

> Claude Code + Gemini를 함께 쓰는 멀티 AI 개발 환경에서,
> **컨텍스트를 효율적으로 관리**하고 **토큰을 절약**하기 위한 디렉토리 설계 원칙과 구조.

---

## 1. 왜 이 구조가 필요한가

AI 코딩 어시스턴트를 쓸 때 가장 큰 병목은 **컨텍스트 오염**이다.

| 문제 | 증상 |
|------|------|
| 관련 없는 파일까지 컨텍스트에 포함 | 토큰 낭비, 응답 품질 저하 |
| 설정이 각 서비스 repo 안에 분산 | AI 세팅 변경 시 여러 repo 수동 수정 |
| Claude / Gemini 간 역할 경계 불명확 | 작업 중복, 핸드오프 혼선 |
| 서비스별 컨텍스트를 항상 전부 로드 | 불필요한 토큰 소모 |

**해결 방향**: 설정을 `docs` 단일 레포에서 중앙 관리하고, 심링크로 각 프로젝트에 연결한다.
AI는 항상 **필요한 컨텍스트만 골라 로드**할 수 있게 된다.

---

## 2. 전체 워크스페이스 구조

```
workspace/
├── docs/                              ← Claude 설정 중앙 허브 (git 추적)
│   ├── SETUP_WORKSPACE.md             ← 신규 PC 세팅 가이드
│   ├── project-agent-config/          ← 프로젝트별 AI 설정 모음
│   │   └── {project-name}/
│   │       ├── CLAUDE.md
│   │       ├── agents/
│   │       ├── skills/
│   │       ├── prds/
│   │       ├── handoffs/
│   │       └── service-context/
│   └── agents-cowork/                 ← 워크플로우 다이어그램, 개발일지
│
├── project-{name}/                    ← 개발 프로젝트 (git 미추적 상위 폴더)
│   ├── service-a/                     ← 개별 git 레포
│   ├── service-b/
│   ├── CLAUDE.md                      ← 심링크 → docs
│   ├── .claude/
│   │   ├── agents/                    ← 심링크 → docs
│   │   └── skills/                    ← 심링크 → docs
│   ├── prds/                          ← 심링크 → docs
│   └── handoffs/                      ← 심링크 → docs
│
└── individual-{name}/                 ← 개인 학습, CS 정리 등
```

---

## 3. project-agent-config 상세 구조

프로젝트 하나(my-backoffice)를 예시로:

```
docs/project-agent-config/my-backoffice/
│
├── CLAUDE.md                          ← 오케스트레이터 지침
│                                         (아키텍처, 포트, 에이전트 역할, 복잡도 판단 기준)
├── README.md                          ← 세팅 방법 가이드
│
├── agents/                            ← 서브에이전트 정의
│   ├── planner.md                     ← Opus: 아키텍처 설계, PRD, 핸드오프
│   ├── be-agent.md                    ← Sonnet: BE 구현 + 코드 리뷰
│   ├── fe-agent.md                    ← Sonnet: FE 구현 + 코드 리뷰
│   └── qa.md                          ← Sonnet: 테스트 계획, QA 검증
│
├── skills/                            ← 글로벌 스킬 (프로젝트 전체에서 호출)
│   ├── prd-writer/SKILL.md
│   ├── handoff-generator/SKILL.md
│   └── code-reviewer/SKILL.md
│
├── prds/                              ← PRD 문서
│   ├── lotto/prd-lotto-service.md
│   └── lotto-upgrade/prd-lotto-upgrade.md
│
├── handoffs/                          ← 에이전트 간 핸드오프 문서
│   ├── lotto/
│   │   ├── handoff-lotto-backend-v1.md
│   │   └── handoff-lotto-frontend-v1.md
│   └── lotto-upgrade/
│       └── handoff-lotto-upgrade-backend-v1.md
│
└── service-context/                   ← 서비스별 로컬 컨텍스트
    ├── auth-server/
    │   ├── CLAUDE.md                  ← auth 도메인 아키텍처 패턴
    │   └── auth-domain/SKILL.md       ← auth 도메인 전용 스킬
    ├── gateway/
    │   ├── CLAUDE.md
    │   └── gateway-domain/SKILL.md
    ├── lotto-server/
    │   ├── CLAUDE.md
    │   └── lotto-domain/SKILL.md
    ├── scraper-server/
    │   ├── CLAUDE.md
    │   └── scraper-domain/SKILL.md
    └── frontend/
        ├── CLAUDE.md
        └── frontend-domain/SKILL.md
```

---

## 4. 심링크 아키텍처

### 핵심 개념

`docs`에서 설정을 한 번만 수정하면 모든 프로젝트에 즉시 반영된다.
각 서비스 repo 안에 `.claude/`를 두지 않는다.

```
┌─────────────────────────────────────────────────┐
│  docs/project-agent-config/my-backoffice/        │
│                                                   │
│  CLAUDE.md   agents/   skills/   prds/  handoffs/ │
└──────┬────────────┬───────┬───────┬───────┬──────┘
       │            │       │       │       │
       │  (심링크)  │       │       │       │
       ▼            ▼       ▼       ▼       ▼
┌─────────────────────────────────────────────────┐
│  project-my-backoffice/  (작업 디렉토리)         │
│                                                   │
│  CLAUDE.md  .claude/agents/  .claude/skills/      │
│             prds/            handoffs/             │
└─────────────────────────────────────────────────┘
```

### 심링크 맵

| 프로젝트 내 경로 | 타입 | 소스 (docs) |
|---|---|---|
| `project-{name}/CLAUDE.md` | 파일 심링크 | `project-agent-config/{name}/CLAUDE.md` |
| `project-{name}/.claude/agents/` | 디렉토리 심링크 | `project-agent-config/{name}/agents/` |
| `project-{name}/.claude/skills/` | 디렉토리 심링크 | `project-agent-config/{name}/skills/` |
| `project-{name}/prds/` | 디렉토리 심링크 | `project-agent-config/{name}/prds/` |
| `project-{name}/handoffs/` | 디렉토리 심링크 | `project-agent-config/{name}/handoffs/` |

### 심링크 생성 명령 (Windows)

```powershell
# 파일 심링크 (CLAUDE.md)
New-Item -ItemType SymbolicLink `
  -Path   "D:\workspace\project-{name}\CLAUDE.md" `
  -Target "D:\workspace\docs\project-agent-config\{name}\CLAUDE.md"

# 디렉토리 심링크 (agents, skills, prds, handoffs)
New-Item -ItemType SymbolicLink `
  -Path   "D:\workspace\project-{name}\.claude\agents" `
  -Target "D:\workspace\docs\project-agent-config\{name}\agents"

New-Item -ItemType SymbolicLink `
  -Path   "D:\workspace\project-{name}\.claude\skills" `
  -Target "D:\workspace\docs\project-agent-config\{name}\skills"

New-Item -ItemType SymbolicLink `
  -Path   "D:\workspace\project-{name}\prds" `
  -Target "D:\workspace\docs\project-agent-config\{name}\prds"

New-Item -ItemType SymbolicLink `
  -Path   "D:\workspace\project-{name}\handoffs" `
  -Target "D:\workspace\docs\project-agent-config\{name}\handoffs"
```

> **사전 조건**: Windows 설정 → 개발자 모드 ON (또는 관리자 권한 터미널)

### .gitignore 처리

심링크는 git이 파일로 인식한다. 각 서비스 repo `.gitignore`에 추가:

```
CLAUDE.md
.claude/
prds/
handoffs/
```

---

## 5. 컨텍스트 로딩 전략

### Claude Code가 컨텍스트를 로드하는 방식

```
Claude 세션 시작
        │
        ▼
CLAUDE.md 자동 로드
(프로젝트 전체 구조, 포트, 기술스택, 오케스트레이터 원칙)
        │
        ├─ 단순 작업? → Main이 직접 처리
        │
        ├─ 중간 복잡도? → 서브에이전트 위임
        │     └─ agents/be-agent.md 또는 fe-agent.md 로드
        │
        └─ 복잡/신규 기능? → Gemini 핸드오프
              └─ /handoff-generator 스킬 호출
```

### 서비스별 컨텍스트 분리 (토큰 절약 핵심)

서비스 작업 시 해당 `service-context/{service}/CLAUDE.md`만 선택적으로 로드.
전체 프로젝트 컨텍스트를 한 번에 올리지 않는다.

```
작업: lotto-server 기능 추가
        │
        ├─ 로드: project CLAUDE.md        (전체 구조 파악)
        ├─ 로드: service-context/lotto-server/CLAUDE.md   (도메인 패턴)
        └─ 로드: lotto-domain/SKILL.md    (도메인 스킬)

        미로드: auth-server, gateway, scraper-server, frontend 컨텍스트
```

---

## 6. 에이전트 역할 분리

```
┌─────────────────────────────────────────────────────────┐
│                    Main Claude (오케스트레이터)           │
│                                                           │
│  • 작업 복잡도 판단                                       │
│  • 에이전트 위임 결정                                     │
│  • Gemini 핸드오프 생성                                   │
│  • 최종 리뷰 및 승인                                      │
└────────────────┬────────────────────────────────────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
  ┌─────────┐ ┌───────┐ ┌─────────────────────────────┐
  │ planner │ │  qa   │ │ be-agent / fe-agent          │
  │ Opus    │ │Sonnet │ │ Sonnet                       │
  │         │ │       │ │                             │
  │ PRD 작성│ │테스트 │ │ 중간복잡도 구현              │
  │ 아키텍처│ │계획   │ │ Gemini 코드 리뷰            │
  │ 설계    │ │QA검증 │ │                             │
  └─────────┘ └───────┘ └─────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Gemini (외부 AI)    │
                         │                     │
                         │ 고복잡도 구현        │
                         │ (핸드오프 문서 기반) │
                         └─────────────────────┘
```

### 복잡도별 처리 방식

| 복잡도 | 기준 | 처리 |
|--------|------|------|
| **단순** | 버그픽스, 파일 1~2개 수정 | Main 직접 처리 |
| **중간** | 여러 파일, 서비스 내 기능 추가 | `be-agent` / `fe-agent` 서브에이전트 위임 |
| **복잡** | 신규 서비스, 아키텍처 변경, 대규모 기능 | Gemini 핸드오프 (`/handoff-generator` 스킬) |

---

## 7. 스킬 시스템

스킬은 반복 작업을 표준화하는 명령어다.
`/skill-name` 형태로 호출하면 해당 `SKILL.md`의 템플릿과 규칙을 따라 실행된다.

### 글로벌 스킬 (프로젝트 전체 공통)

| 스킬 | 파일 위치 | 용도 |
|------|-----------|------|
| `/prd-writer` | `skills/prd-writer/SKILL.md` | PRD 문서 작성 |
| `/handoff-generator` | `skills/handoff-generator/SKILL.md` | Gemini 핸드오프 문서 생성 |
| `/code-reviewer` | `skills/code-reviewer/SKILL.md` | 코드 리뷰 체크리스트 실행 |

### 도메인 스킬 (서비스별)

| 스킬 | 파일 위치 | 용도 |
|------|-----------|------|
| auth-domain | `service-context/auth-server/auth-domain/SKILL.md` | auth 서비스 도메인 규칙 |
| gateway-domain | `service-context/gateway/gateway-domain/SKILL.md` | gateway 라우팅 패턴 |
| lotto-domain | `service-context/lotto-server/lotto-domain/SKILL.md` | lotto 도메인 규칙 |
| scraper-domain | `service-context/scraper-server/scraper-domain/SKILL.md` | scraper 패턴 |
| frontend-domain | `service-context/frontend/frontend-domain/SKILL.md` | FE 컴포넌트 규칙 |

### 스킬 강제 규칙

```
PRD 작성 요청      → /prd-writer 스킬 사용 필수
핸드오프 생성 요청 → /handoff-generator 스킬 사용 필수
코드 리뷰 요청     → /code-reviewer 스킬 사용 필수
```

---

## 8. 핸드오프 문서 흐름

Gemini에게 작업을 넘길 때 사용하는 핸드오프 문서의 생명주기:

```
1. Main Claude
   /handoff-generator 스킬 호출
          │
          ▼
2. handoffs/{feature}/handoff-{service}-{layer}-v1.md 생성
   (prds/ 문서 참조, 구현 스펙 + Q&A 섹션 포함)
          │
          ▼
3. Gemini 구현
   • 핸드오프 문서 읽고 구현
   • 질문 → QUESTIONS 섹션에 추가
          │
          ▼
4. Main Claude 답변
   • ANSWERS 섹션에 작성
   • Gemini CONFIRMED 후 → RESOLVED 체크 (Main만)
          │
          ▼
5. 구현 완료 + 리뷰
   be-agent /code-reviewer 스킬로 리뷰
          │
          ▼
6. 완료 처리
   handoffs/{feature}/ → reviewed/로 이동, 파일명에 -final 추가
```

---

## 9. 파일 네이밍 규칙

| 파일 종류 | 네이밍 규칙 | 예시 |
|-----------|-------------|------|
| 핸드오프 | `handoff-{service}-{layer}-v{N}.md` | `handoff-lotto-backend-v1.md` |
| PRD | `prd-{feature-name}.md` | `prd-lotto-upgrade.md` |
| 에이전트 | `{role}.md` | `be-agent.md`, `planner.md` |
| 스킬 | `{skill-name}/SKILL.md` | `handoff-generator/SKILL.md` |
| 서비스 컨텍스트 스킬 | `{domain}/SKILL.md` | `lotto-domain/SKILL.md` |

---

## 10. 신규 프로젝트 세팅 체크리스트

```
□ 1. workspace/{project-name}/ 폴더 생성
□ 2. docs/project-agent-config/{project-name}/ 폴더 생성
      □ CLAUDE.md 작성 (아키텍처, 포트, 에이전트 원칙)
      □ README.md 작성 (세팅 방법)
      □ agents/ 폴더 + 역할별 md 작성
      □ skills/ 폴더 + 글로벌 스킬 작성
      □ prds/ 폴더 생성
      □ handoffs/ 폴더 생성
      □ service-context/{service}/ 폴더 + CLAUDE.md + 도메인 스킬 작성
□ 3. project-{name}/.claude/ 폴더 생성
□ 4. 심링크 5개 생성
      □ CLAUDE.md (파일)
      □ .claude/agents/ (디렉토리)
      □ .claude/skills/ (디렉토리)
      □ prds/ (디렉토리)
      □ handoffs/ (디렉토리)
□ 5. 각 서비스 repo .gitignore에 심링크 경로 추가
□ 6. docs 변경사항 커밋 & 푸시
```

---

*이 문서는 Claude + Gemini 멀티 AI 개발 환경의 디렉토리 아키텍처 원칙을 정의합니다.*
*워크플로우 상세는 `agents-cowork/my-workflow.md` 참조.*
