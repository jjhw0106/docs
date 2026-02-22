# my-backoffice — Claude 설정 & 세팅 가이드

my-backoffice는 MSA 구조로 단일 git 레포가 없음.
전체 오케스트레이션 CLAUDE.md와 .claude/ 설정을 여기서 일괄 관리.

---

## 포함 파일 구조

```
my-backoffice/
├── CLAUDE.md                         ← 프로젝트 전체 오케스트레이션 지침
├── README.md                         ← 이 파일
├── agents/                           ← .claude/agents/ 내용
│   ├── planner.md
│   ├── be-agent.md
│   ├── fe-agent.md
│   └── qa.md
├── skills/                           ← .claude/skills/ 내용
│   ├── prd-writer/SKILL.md
│   ├── handoff-generator/SKILL.md
│   └── code-reviewer/SKILL.md
└── service-context/                  ← 서비스별 .claude/ 내용
    ├── auth-server/
    │   ├── CLAUDE.md
    │   └── skills/auth-domain/SKILL.md
    ├── gateway/
    │   ├── CLAUDE.md
    │   └── skills/gateway-domain/SKILL.md
    ├── scraper-server/
    │   ├── CLAUDE.md
    │   └── skills/scraper-domain/SKILL.md
    ├── lotto-server/
    │   ├── CLAUDE.md
    │   └── skills/lotto-domain/SKILL.md
    └── frontend/
        ├── CLAUDE.md
        └── skills/frontend-domain/SKILL.md
```

---

## 새 PC 세팅 방법

docs clone 후 Claude에게 요청:

> "docs/project-main-claudeconfig/my-backoffice/README.md 읽고 project-my-backoffice 세팅해줘"

Claude가 아래를 자동 생성:

| 생성 위치 | 소스 |
|-----------|------|
| `project-my-backoffice/CLAUDE.md` | `CLAUDE.md` |
| `project-my-backoffice/.claude/agents/` | `agents/` |
| `project-my-backoffice/.claude/skills/` | `skills/` |
| `project-my-backoffice/auth-server/.claude/` | `service-context/auth-server/` |
| `project-my-backoffice/gateway/.claude/` | `service-context/gateway/` |
| `project-my-backoffice/scraper-server/.claude/` | `service-context/scraper-server/` |
| `project-my-backoffice/lotto-server/.claude/` | `service-context/lotto-server/` |
| `project-my-backoffice/my-bo-frontend/frontend/.claude/` | `service-context/frontend/` |

---

## 서비스 레포 clone 목록

```bash
cd ~/workspace/project-my-backoffice

git clone <auth-server-url> auth-server
git clone <gateway-url> gateway
git clone <scraper-server-url> scraper-server
git clone <lotto-server-url> lotto-server
git clone <frontend-url> my-bo-frontend
```

---

## 설정 변경 후 백업 방법

```bash
BASE=~/workspace/project-my-backoffice
BACKUP=~/workspace/docs/project-main-claudeconfig/my-backoffice

# CLAUDE.md
cp $BASE/CLAUDE.md $BACKUP/

# agents & skills
cp $BASE/.claude/agents/*.md $BACKUP/agents/
cp $BASE/.claude/skills/prd-writer/SKILL.md $BACKUP/skills/prd-writer/
cp $BASE/.claude/skills/handoff-generator/SKILL.md $BACKUP/skills/handoff-generator/
cp $BASE/.claude/skills/code-reviewer/SKILL.md $BACKUP/skills/code-reviewer/

# service-context
cp $BASE/auth-server/.claude/CLAUDE.md $BACKUP/service-context/auth-server/
cp $BASE/auth-server/.claude/skills/auth-domain/SKILL.md $BACKUP/service-context/auth-server/skills/auth-domain/
# ... 나머지 서비스 동일

cd ~/workspace/docs && git add . && git commit -m "chore: update my-backoffice claude config" && git push
```
