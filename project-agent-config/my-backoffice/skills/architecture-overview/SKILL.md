# Architecture Overview

서비스 아키텍처 상세 레퍼런스. 포트, 통신 구조, 기술 스택, 구현 세부사항, 코딩 규칙 포함.

---

## Service Ports & Communication

| Service | Port | Tech |
|---------|------|------|
| Frontend (Nuxt) | 3000 | Nuxt 4 + Vue 3.5 |
| API Gateway | 8080 | Spring Cloud Gateway |
| Auth Server | 9001 | Spring Boot 3.5 |
| Scraper Server | 4000 | NestJS 11 |
| Lotto Server | 8082 | Spring Boot 3.5 |
| Resume Service | 8081 | (gateway config 참조) |

### Communication Flow
- Frontend → Gateway (8080) → Auth Server (9001) for authentication
- Frontend → Gateway (8080) → Lotto Server (8082) for lotto operations
- Frontend → Scraper Server (4000) directly for scraping operations
- Gateway routes:
  - `/auth/**` → auth-server:9001
  - `/resume/**` → resume-service:8081
  - `/lotto/**` → lotto-server:8082

---

## Tech Stack by Service

### Frontend (my-bo-frontend/frontend)
- Nuxt 4.2.2 with Vue 3.5
- Tailwind CSS 3.4 for styling (@nuxtjs/tailwindcss 6.14)
- Composables: `useAuth`, `useScraper`, `useExcel`, `usePagePermissions`, `lotto/useLotto`
- Key features: Job application tracking, platform login, data export to Excel, lotto number generation & purchase tracking

### Auth Server (auth-server)
- Spring Boot 3.5.0 with Java 17
- Spring Security + JWT authentication (Auth0 java-jwt 4.5.0)
- MySQL database with Spring Data JPA
- Hexagonal architecture (ports and adapters pattern)
- Package structure: `application` (use cases), `domain` (entities, VOs), `infra` (repositories), `web` (controllers)

### Gateway (gateway)
- Spring Cloud Gateway with WebFlux (reactive)
- Spring Boot 3.4.4 with Java 17
- Spring Cloud 2024.0.1
- CORS configured for localhost:3000

### Scraper Server (scraper-server)
- NestJS 11 with TypeScript
- Playwright 1.57 for browser automation
- MongoDB with Mongoose 9.1 for data persistence
- Scrapes job platforms: Wanted, JobKorea
- Supports both manual and automatic login modes

### Lotto Server (lotto-server)
- Spring Boot 3.5.0 with Java 17
- Spring Security + JWT authentication (Auth0 java-jwt 4.5.0)
- MySQL database with Spring Data JPA (same DB as auth-server)
- Hexagonal architecture (mirrors auth-server pattern)
- WebFlux WebClient for external API calls
- Lombok for boilerplate reduction
- Scheduled batch jobs (winning number fetch, data cleanup)
- External API: 동행복권 (dhlottery.co.kr)

---

## Frontend Structure

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

---

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

---

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
