# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a microservices-based backoffice application for managing job applications, career tracking, and personal lotto management. The system consists of a Nuxt frontend, Spring Cloud Gateway, Spring Boot auth service, NestJS scraper service, and Spring Boot lotto service.

## Architecture

### Service Ports
- **Frontend** (Nuxt): Port 3000
- **API Gateway** (Spring Cloud Gateway): Port 8080
- **Auth Server** (Spring Boot): Port 9001
- **Scraper Server** (NestJS): Port 4000
- **Lotto Server** (Spring Boot): Port 8082
- **Resume Service**: Port 8081 (referenced in gateway config)

### Service Communication
- Frontend → Gateway (port 8080) → Auth Server (port 9001) for authentication
- Frontend → Gateway (port 8080) → Lotto Server (port 8082) for lotto operations
- Frontend → Scraper Server (port 4000) directly for scraping operations
- Gateway routes:
  - `/auth/**` → auth-server:9001
  - `/resume/**` → resume-service:8081
  - `/lotto/**` → lotto-server:8082

### Tech Stack by Service

**Frontend (my-bo-frontend/frontend)**
- Nuxt 4.2.2 with Vue 3.5
- Tailwind CSS 3.4 for styling (@nuxtjs/tailwindcss 6.14)
- Composables: `useAuth`, `useScraper`, `useExcel`, `usePagePermissions`, `lotto/useLotto`
- Key features: Job application tracking, platform login, data export to Excel, lotto number generation & purchase tracking

**Auth Server (auth-server)**
- Spring Boot 3.5.0 with Java 17
- Spring Security + JWT authentication (Auth0 java-jwt 4.5.0)
- MySQL database with Spring Data JPA
- Hexagonal architecture (ports and adapters pattern)
- Package structure: `application` (use cases), `domain` (entities, VOs), `infra` (repositories), `web` (controllers)

**Gateway (gateway)**
- Spring Cloud Gateway with WebFlux (reactive)
- Spring Boot 3.4.4 with Java 17
- Spring Cloud 2024.0.1
- CORS configured for localhost:3000

**Scraper Server (scraper-server)**
- NestJS 11 with TypeScript
- Playwright 1.57 for browser automation
- MongoDB with Mongoose 9.1 for data persistence
- Scrapes job platforms: Wanted, JobKorea
- Supports both manual and automatic login modes

**Lotto Server (lotto-server)**
- Spring Boot 3.5.0 with Java 17
- Spring Security + JWT authentication (Auth0 java-jwt 4.5.0)
- MySQL database with Spring Data JPA (same DB as auth-server)
- Hexagonal architecture (mirrors auth-server pattern)
- WebFlux WebClient for external API calls
- Lombok for boilerplate reduction
- Scheduled batch jobs (winning number fetch, data cleanup)
- External API: 동행복권 (dhlottery.co.kr)

## Project Structure

```
my-backoffice/
├── .claude/                    # Claude Code agent configurations (6 agents)
├── auth-server/                # Spring Boot auth service (port 9001)
├── gateway/                    # Spring Cloud Gateway (port 8080)
├── lotto-server/               # Spring Boot lotto service (port 8082)
├── scraper-server/             # NestJS scraper service (port 4000)
├── my-bo-frontend/frontend/    # Nuxt frontend (port 3000)
├── docs/                    # Dev logs, To-Do, setup guides, PRD & handoff docs
├── CLAUDE.md                   # Project instructions (English)
└── CLAUDE_kor.md               # Project instructions (Korean)
```

### Frontend Structure
```
my-bo-frontend/frontend/
├── pages/
│   ├── index.vue               # Root public page
│   ├── login.vue               # Login page
│   ├── signup.vue              # Signup page
│   ├── settings.vue            # User settings (dashboard layout)
│   ├── stocks.vue              # Stocks page (protected)
│   ├── lotto/
│   │   ├── index.vue           # Number generator (3 strategies: random/balanced/odd-even)
│   │   ├── dashboard.vue       # Lotto dashboard
│   │   ├── history.vue         # Purchase history with pagination & filtering
│   │   └── purchase.vue        # Number purchase registration
│   └── my-career/
│       ├── index.vue           # Overview dashboard (aggregate stats)
│       └── [platform].vue      # Per-platform view (jobkorea, wanted)
├── composables/
│   ├── useAuth.ts              # Login, signup, logout; cookie-based user state
│   ├── useScraper.ts           # Platform scraping, history fetch, multi-scrape
│   ├── useExcel.ts             # XLSX-based Excel download utility
│   ├── usePagePermissions.ts   # Protected routes list
│   └── lotto/
│       └── useLotto.ts         # Lotto CRUD: winning numbers, purchases, summary
├── components/
│   ├── domain/                 # Business-specific components
│   │   ├── LoginForm.vue
│   │   ├── SignupForm.vue
│   │   ├── PlatformLoginModal.vue
│   │   ├── ScrapeButton.vue
│   │   └── SubButton.vue       # Reusable secondary action button with slot
│   ├── layout/                 # Layout components
│   │   ├── AppHeader.vue       # Fixed top nav bar (shared across layouts)
│   │   ├── AppSidebar.vue      # Dashboard sidebar with accordion menu
│   │   └── HistoryFilter.vue   # Filter component for history views
│   └── lotto/                  # Lotto-specific components
│       ├── LottoBall.vue       # Number ball with color coding
│       ├── LottoNumberInput.vue
│       ├── PurchaseForm.vue
│       ├── PurchaseHistoryTable.vue
│       ├── PurchaseSummaryCards.vue
│       ├── RankBadge.vue
│       └── WinningNumberCard.vue
├── layouts/
│   ├── default.vue             # Public pages layout
│   └── dashboard.vue           # Authenticated pages (header + sidebar)
├── middleware/
│   └── auth.global.ts          # Global route guard
└── types/
    └── auth.ts
```

## Development Commands

### Frontend (my-bo-frontend/frontend)
```bash
cd my-bo-frontend/frontend
npm install                 # Install dependencies
npm run dev                 # Start dev server (port 3000)
npm run build              # Build for production
npm run preview            # Preview production build
```

### Auth Server (auth-server)
```bash
cd auth-server
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 9001)
./gradlew test            # Run tests
```

### Gateway (gateway)
```bash
cd gateway
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 8080)
./gradlew test            # Run tests
```

### Scraper Server (scraper-server)
```bash
cd scraper-server
npm install               # Install dependencies
npm run start:dev         # Start in watch mode (port 4000)
npm run build             # Build the project
npm run start:prod        # Run production build
npm run test              # Run unit tests
npm run test:e2e          # Run e2e tests
npm run lint              # Lint and fix code
npm run format            # Format code with prettier
```

### Lotto Server (lotto-server)
```bash
cd lotto-server
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 8082)
./gradlew test            # Run tests
```

## Key Implementation Details

### Authentication Flow
1. User logs in via frontend → calls `/auth/login` through Gateway
2. Auth server validates credentials, issues JWT tokens (access + refresh)
3. Access tokens stored in cookies (HttpOnly), expiry 10min (access) / 1hr (refresh)
4. Frontend stores user info in `useCookie('user')`, appUserId in `localStorage('last_user_id')`

### Job Scraping Flow
1. Frontend calls scraper service directly (not through gateway)
2. Playwright launches browser (headless:false for debugging)
3. Scraper handles login (auto or manual) for each platform
4. Data extracted via DOM selectors with pagination support
5. Results saved to MongoDB, replacing previous data for that user/platform via `deleteMany({ appUserId, platform })`
6. Schema includes: appUserId, platformUserId, platform, company, position, status, appliedAt

### Scraper Platform Support
- **Wanted**: Scrapes application history with pagination (up to 20 pages), text-pattern-based extraction
- **JobKorea**: Scrapes application history with pagination (up to 5 pages), CSS-selector-based extraction
- Both support automatic login if credentials provided, otherwise manual login with 2-3 minute timeout

### Lotto Service Flow
1. **Number Generator** (frontend-only): 3 strategies (random, balanced, odd-even), 1-5 games
2. **Purchase Registration**: POST /lotto/purchases with round + games array, JWT authenticated
3. **Winning Number Fetch**: Cache-first (DB → external dhlottery API), auto-fetched by Saturday 9PM batch
4. **Rank Calculation**: Auto-calculated on batch run or when registering past rounds
5. **Purchase History**: Paginated, filterable by round range and result (WON/LOST/ALL)
6. **Summary Stats**: Total games, wins, investment (games x 2500), prize, win rate, rank distribution

### Database Connections
- Auth server: MySQL (`jdbc:mysql://localhost:3306/member`), profile-based (local, dev)
- Lotto server: MySQL (same DB as auth-server), profile-based
- Scraper server: MongoDB via `MONGODB_URI` environment variable

## Important Conventions

### Frontend
- Composables use shared reactive state (e.g., `isScraping` defined outside function)
- Layouts: `default.vue` for public pages, `dashboard.vue` for authenticated pages
- Pages follow Nuxt file-based routing
- AppHeader and AppSidebar are shared across dashboard layout
- Sidebar uses accordion menu with auto-open for active section

### Auth Server
- Follows hexagonal architecture: separate domain, application, infrastructure layers
- Value Objects (VOs) for domain primitives: Email, HashedPassword, Provider, Role, Status
- Use case interfaces defined in `application.port.in`
- Repository interfaces defined in `application.port.out`

### Lotto Server
- Mirrors auth-server hexagonal architecture
- Domain entities: WinningNumbers, PurchaseGame
- Value Objects: LottoNumber (1-45), Round, Rank (FIRST-FIFTH, NONE)
- RankCalculator as pure domain logic
- Batch jobs: WinningNumberFetchJob (Sat 9PM), DataCleanupJob (daily 3AM)
- Currently uses `DEFAULT_MEMBER_ID = 1L` (not yet multi-user)

### Scraper Server
- Each platform has dedicated private scraping method
- Data saving separated from scraping logic
- Always close browser in finally block
- CORS enabled for all origins in development

## API Endpoints Summary

### Auth Server (via Gateway /auth/**)
- `POST /auth/login` - Login, returns JWT in HttpOnly cookie
- `POST /auth/sign-up` - User registration

### Scraper Server (direct, port 4000)
- `POST /scraper/:platform` - Trigger scrape (body: platformUserId, platformUserPw, appUserId)
- `GET /scraper/history/:userId` - Fetch apply history by appUserId

### Lotto Server (via Gateway /lotto/**)
- `GET /lotto/winning-numbers?round={round}` - Winning numbers (public)
- `POST /lotto/purchases` - Register purchase (JWT required)
- `GET /lotto/purchases?page&size&result&roundFrom&roundTo` - Purchase history (JWT required)
- `GET /lotto/purchases/summary` - Statistics summary (JWT required)
- `DELETE /lotto/purchases/{gameId}` - Delete purchase (JWT required)

## Testing

### Frontend
No test framework configured yet.

### Auth Server
JUnit Platform with Spring Boot Test support:
```bash
./gradlew test
```

### Gateway
```bash
./gradlew test
```

### Scraper Server
Jest for unit tests:
```bash
npm run test              # Run all tests
npm run test:watch        # Watch mode
npm run test:cov          # With coverage
npm run test:debug        # Debug mode
npm run test:e2e          # E2E tests
```

### Lotto Server
JUnit Platform with Spring Boot Test support:
```bash
./gradlew test
```

## Environment Variables

### Frontend
- `apiBase`: API Gateway URL (default: http://localhost:8080)

### Auth Server
- Database connection settings in application.yml (profile-based: local, dev)
- JWT secret: `JWT_SECRET` (default: myboproject)
- Spring profiles: `SPRING_PROFILES_ACTIVE` (default: local)

### Scraper Server
- `MONGODB_URI`: MongoDB connection string (required)
- `PORT`: Server port (default: 4000)

### Lotto Server
- Database connection settings in application.yml (profile-based: local, dev)
- JWT secret: shared with auth-server
- Spring profiles: `SPRING_PROFILES_ACTIVE` (default: local)

## Documentation

### docs/
- `To-Do.md` - Current task tracking
- `개발일지/` - Development logs (api/, front/ subdirectories)
- `개발환경 세팅/` - Environment setup guides
- `agents-cowork/` - Diagrams and workflow documentation
  - `diagrams/` - Architecture diagrams
  - `md-images/` - Diagram images
  - `my-workflow.md` - Agent workflow documentation
- `prd/` - Product requirement documents
- `architecture/adr/` - Architecture decision records
- `api-contracts/` - API contract definitions
- `handoff/` - Gemini handoff documents
  - `pending/` - Awaiting implementation or review
  - `reviewed/` - Completed and reviewed

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

### Gemini 핸드오프 Q&A 처리 순서

`QUESTIONS` → `ANSWERS` → `CONFIRMED` → `RESOLVED`

- QUESTIONS: Gemini가 작성
- ANSWERS: Main이 작성
- CONFIRMED: Gemini가 표시
- **RESOLVED: Main만 체크 가능** (모든 Q&A 완료 후)

### 핸드오프 네이밍 규칙

- 파일명: `handoff-{service}-{layer}-v{N}.md`
- service: `auth`, `gateway`, `scraper`, `lotto`, `frontend`
- layer: `backend`, `frontend`
- **버전업**: 스펙 변경 시만 (Q&A는 같은 파일에 append, 버전 유지)
- **완료 시**: `docs/handoff/reviewed/`로 이동 + 파일명에 `-final` 추가

### 문서 위치 규칙
- PRD → `docs/prd/`
- ADR → `docs/architecture/adr/`
- API 계약서 → `docs/api-contracts/`
- 핸드오프 → `docs/handoff/pending/`

### 서비스별 도메인 스킬

각 서비스 작업 시 해당 도메인 스킬 참조:
- auth-server 작업: `auth-server/.claude/skills/auth-domain/SKILL.md`
- gateway 작업: `gateway/.claude/skills/gateway-domain/SKILL.md`
- scraper-server 작업: `scraper-server/.claude/skills/scraper-domain/SKILL.md`
- lotto-server 작업: `lotto-server/.claude/skills/lotto-domain/SKILL.md`
- frontend 작업: `my-bo-frontend/frontend/.claude/skills/frontend-domain/SKILL.md`
