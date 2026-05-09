# API Reference

API 엔드포인트, DB 연결, 환경변수 레퍼런스.

---

## API Endpoints

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

---

## Database Connections

| Service | DB | Connection |
|---------|-----|-----------|
| Auth Server | MySQL | `jdbc:mysql://localhost:3306/member` (profile-based: local, dev) |
| Lotto Server | MySQL | Same DB as auth-server (profile-based) |
| Scraper Server | MongoDB | `MONGODB_URI` environment variable |

---

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
