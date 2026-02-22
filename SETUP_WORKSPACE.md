# Workspace Setup Guide

새 PC에서 개발 환경을 처음 세팅하는 절차와 원칙.

---

## 1. 세팅 순서

```
1. workspace 폴더 생성
2. docs clone
3. Claude 실행
4. 이 파일(SETUP_WORKSPACE.md) 참조해서 세팅 진행
5. 프로젝트별 설정은 project-agent-config/{project}/ 참조
```

---

## 2. 필수 설치

| 항목 | 용도 |
|------|------|
| Claude Code CLI | AI 코딩 어시스턴트 |
| Node.js 20+ | Frontend, Scraper Server |
| Java 17 | Spring Boot 서비스 |
| Docker | 로컬 DB (MySQL, MongoDB) |
| Git | 버전 관리 |

---

## 3. 워크스페이스 폴더 구조

```
workspace/
├── docs/                          ← 문서 레포 (먼저 clone)
├── project-{name}/                   ← 개발 프로젝트 (단일/멀티 레포 무관)
│   ├── service-a/                    ← 개별 git 레포 (멀티 레포인 경우)
│   └── service-b/
└── individual-{name}/                ← 개인 학습, CS 정리, 개인 용도
```

- `project-*` : 개발 프로젝트 단위 (단일 레포든 멀티 레포든 무관)
- `individual-*` : 개인 학습, CS 정리, 아이디어 뱅크 등 개인 용도
- 멀티 레포 프로젝트는 상위 폴더(git 미추적) 안에서 각 레포를 clone

---

## 4. Claude 설정 관리 원칙

모든 Claude 설정은 **docs에서 일괄 관리**하고 git으로 추적.
각 서비스 레포 안에 `.claude/`를 두지 않는다.

```
docs/
└── project-agent-config/
    └── {project-name}/
        ├── CLAUDE.md           ← 프로젝트 전체 오케스트레이션 지침
        ├── README.md           ← 세팅 방법 가이드
        ├── agents/             ← 서브에이전트 정의
        ├── skills/             ← 글로벌 스킬 정의
        ├── prds/               ← PRD 문서
        ├── handoffs/           ← 에이전트 간 핸드오프 문서
        └── service-context/    ← 서비스별 .claude 내용
            └── {service}/
                ├── CLAUDE.md
                └── skills/
```

---

## 5. 세팅 시 Claude 역할

Claude는 `project-agent-config/{project}/` 를 소스로 삼아 **심링크**를 생성한다.
심링크 방식이므로 docs 수정 시 각 프로젝트에 즉시 반영된다.

### 사전 조건
- Windows 개발자 모드 활성화 (설정 → 개인 정보 및 보안 → 개발자용 → 개발자 모드 ON)
- 또는 관리자 권한으로 터미널 실행

### 심링크 생성 명령 (프로젝트 세팅 시 실행)

```bash
# CLAUDE.md (파일 심링크)
mklink "{project-root}\CLAUDE.md" \
       "D:\workspace\docs\project-agent-config\{project}\CLAUDE.md"

# agents/ (디렉토리 심링크)
mklink /D "{project-root}\.claude\agents" \
          "D:\workspace\docs\project-agent-config\{project}\agents"

# skills/ (디렉토리 심링크)
mklink /D "{project-root}\.claude\skills" \
          "D:\workspace\docs\project-agent-config\{project}\skills"
```

### 심링크 생성 명령 (prds, handoffs 추가)

```bash
# prds/ (디렉토리 심링크)
mklink /D "{project-root}\prds" \
          "D:\workspace\docs\project-agent-config\{project}\prds"

# handoffs/ (디렉토리 심링크)
mklink /D "{project-root}\handoffs" \
          "D:\workspace\docs\project-agent-config\{project}\handoffs"
```

### 심링크 대상 정리

| 심링크 위치 | 소스 (docs) |
|-------------|----------------|
| `project-{name}/CLAUDE.md` | `project-agent-config/{project}/CLAUDE.md` |
| `project-{name}/.claude/agents/` | `project-agent-config/{project}/agents/` |
| `project-{name}/.claude/skills/` | `project-agent-config/{project}/skills/` |
| `project-{name}/prds/` | `project-agent-config/{project}/prds/` |
| `project-{name}/handoffs/` | `project-agent-config/{project}/handoffs/` |

### 주의사항
- 심링크는 git이 파일로 인식하므로 각 프로젝트 `.gitignore`에 추가
  ```
  CLAUDE.md
  .claude/
  prds/
  handoffs/
  ```

---

## 6. 파일 생성 원칙

### CLAUDE.md
- 프로젝트 전체 구조, 서비스 포트, 기술 스택 포함
- 오케스트레이터 행동 원칙 (작업 복잡도 판단, 에이전트 위임 기준) 포함
- 스킬 강제 규칙 명시

### agents/
- 파일명: `{role}.md`
- 프론트매터: `name`, `description`, `model` 필수
- 역할 명확히 분리 (구현 / 리뷰 / 기획 / QA)

### skills/
- 폴더명: `{skill-name}/SKILL.md`
- 사용 시점, 출력 위치, 문서 구조 템플릿 포함
- 네이밍 규칙 명시

### service-context/{service}/CLAUDE.md
- 해당 서비스의 도메인 컨텍스트만 포함
- 아키텍처 패턴, 핵심 패턴, 금지 패턴 명시

### handoff 파일
- 위치: `project-agent-config/{project}/handoffs/{feature}/` (프로젝트 루트에 심링크)
- 기능/서비스별 서브폴더 생성 후 그 안에 md 작성
- 네이밍: `handoff-{service}-{layer}-v{N}.md`
- Q&A 섹션 포함 (QUESTIONS → ANSWERS → CONFIRMED → RESOLVED)
- 에이전트(Gemini 등)는 `{project-root}/handoffs/` 상대 경로로 접근

```
handoffs/
└── {feature}/
    └── handoff-{service}-{layer}-v{N}.md
```

### PRD 파일
- 위치: `project-agent-config/{project}/prds/{feature}/` (프로젝트 루트에 심링크)
- 기능/서비스별 서브폴더 생성 후 그 안에 md 작성
- 네이밍: `prd-{feature-name}.md`

```
prds/
└── {feature}/
    └── prd-{feature-name}.md
```

---

## 7. 프로젝트별 세팅 가이드

| 프로젝트 | 가이드 |
|----------|--------|
| my-backoffice | `docs/project-agent-config/my-backoffice/README.md` |

> 새 프로젝트 추가 시 `project-agent-config/{project-name}/` 형식으로 추가.
