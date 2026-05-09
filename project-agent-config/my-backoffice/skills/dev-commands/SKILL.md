# Dev Commands

서비스별 빌드, 실행, 테스트 커맨드 레퍼런스.

---

## Frontend (my-bo-frontend/frontend)

```bash
cd my-bo-frontend/frontend
npm install                 # Install dependencies
npm run dev                 # Start dev server (port 3000)
npm run build              # Build for production
npm run preview            # Preview production build
```

> No test framework configured yet.

---

## Auth Server (auth-server)

```bash
cd auth-server
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 9001)
./gradlew test            # Run tests (JUnit + Spring Boot Test)
```

---

## Gateway (gateway)

```bash
cd gateway
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 8080)
./gradlew test            # Run tests
```

---

## Scraper Server (scraper-server)

```bash
cd scraper-server
npm install               # Install dependencies
npm run start:dev         # Start in watch mode (port 4000)
npm run build             # Build the project
npm run start:prod        # Run production build
npm run test              # Run unit tests (Jest)
npm run test:watch        # Watch mode
npm run test:cov          # With coverage
npm run test:debug        # Debug mode
npm run test:e2e          # E2E tests
npm run lint              # Lint and fix code
npm run format            # Format code with prettier
```

---

## Lotto Server (lotto-server)

```bash
cd lotto-server
./gradlew build           # Build the project
./gradlew bootRun         # Run in development (port 8082)
./gradlew test            # Run tests (JUnit + Spring Boot Test)
```
