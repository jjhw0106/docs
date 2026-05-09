# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## EC2 배포 주의사항

auth-server를 EC2에 재배포할 때 **반드시** 아래 순서로 진행한다:

```bash
df -h /                          # 디스크 여유 확인
docker system prune -a -f        # 미사용 이미지/캐시 전체 삭제 (-a 필수)
docker compose up -d --build auth-server
```

> `-a` 없이 prune하면 이미지가 남아 디스크 부족으로 Gradle 빌드 실패함.
> 증상: `Could not receive a message from the daemon. BUILD FAILED`

---

## Project Overview

Microservices-based backoffice application: Nuxt frontend + Spring Cloud Gateway + Spring Boot auth/lotto services + NestJS scraper service.

## Architecture

| Service | Port | Tech |
|---------|------|------|
| Frontend | 3000 | Nuxt 4 + Vue 3.5 |
| API Gateway | 8080 | Spring Cloud Gateway |
| Auth Server | 9001 | Spring Boot 3.5 |
| Scraper Server | 4000 | NestJS 11 |
| Lotto Server | 8082 | Spring Boot 3.5 |

> 상세 아키텍처, 기술 스택, 코딩 규칙 → `/architecture-overview` 스킬 참조

## Project Structure

```
my-backoffice/
├── .claude/                    # Claude Code agent configurations (6 agents)
├── auth-server/                # Spring Boot auth service (port 9001)
├── gateway/                    # Spring Cloud Gateway (port 8080)
├── lotto-server/               # Spring Boot lotto service (port 8082)
├── scraper-server/             # NestJS scraper service (port 4000)
├── my-bo-frontend/frontend/    # Nuxt frontend (port 3000)
├── docs/                       # Dev logs, To-Do, setup guides, PRD & handoff docs
└── CLAUDE.md                   # Project instructions
```

## Skills 안내

필요한 정보는 아래 스킬을 호출하여 참조:

| 스킬 | 내용 |
|------|------|
| `/architecture-overview` | 서비스 통신 구조, 기술 스택 상세, 프론트엔드 파일 트리, 구현 세부사항, 코딩 규칙 |
| `/dev-commands` | 서비스별 빌드/실행/테스트 커맨드 |
| `/api-reference` | API 엔드포인트, DB 연결 정보, 환경변수 |
| `/prd-writer` | PRD 작성 |
| `/handoff-generator` | Gemini 핸드오프 문서 생성 |
| `/code-reviewer` | 코드 리뷰 |

---

## Git 워크플로우 규칙

- 새 프로젝트 git init + push 시, `.gitignore`가 없으면 **반드시 먼저 생성** 후 커밋/푸시 (node_modules, .env 등 유출 방지)

---

## 오케스트레이터 행동 원칙

### 작업 복잡도 판단 기준

| 복잡도 | 판단 기준 | 처리 방식 |
|--------|-----------|-----------|
| **단순** | 버그픽스, 소규모 기능, 파일 1-2개 수정 | Main이 직접 코드 작성 |
| **중간** | 여러 파일, 서비스 내 기능 추가 | `be-agent` 또는 `fe-agent` 서브에이전트에게 위임 |
| **복잡** | 신규 서비스, 아키텍처 변경, 대규모 기능 | Gemini 핸드오프 (`/handoff-generator` 스킬 사용) |

### 스킬 강제 규칙

- **PRD 작성** → `/prd-writer` 스킬 사용 필수
- **핸드오프 생성** → `/handoff-generator` 스킬 사용 필수
- **코드 리뷰** → `/code-reviewer` 스킬 사용 필수

### 에이전트 역할

| 에이전트 | 모델 | 역할 |
|----------|------|------|
| `planner` | claude-opus-4-6 | 아키텍처 설계, PRD 작성, 핸드오프 생성 |
| `be-agent` | claude-sonnet-4-6 | BE 구현 (중간 복잡도) + BE 코드 리뷰 |
| `fe-agent` | claude-sonnet-4-6 | FE 구현 (중간 복잡도) + FE 코드 리뷰 |
| `qa` | claude-sonnet-4-6 | 테스트 계획, QA 체크리스트, 통합 검증 |

### 서비스별 도메인 스킬

각 서비스 작업 시 해당 도메인 스킬 참조:
- auth-server 작업: `auth-server/.claude/skills/auth-domain/SKILL.md`
- gateway 작업: `gateway/.claude/skills/gateway-domain/SKILL.md`
- scraper-server 작업: `scraper-server/.claude/skills/scraper-domain/SKILL.md`
- lotto-server 작업: `lotto-server/.claude/skills/lotto-domain/SKILL.md`
- frontend 작업: `my-bo-frontend/frontend/.claude/skills/frontend-domain/SKILL.md`
