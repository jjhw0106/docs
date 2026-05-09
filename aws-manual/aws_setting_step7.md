# Phase 7: Vercel 프론트 ↔ auth-server 연결

> **목표:** Vercel에 배포된 프론트(`https://www.moneyttuk.com`)에서 EC2 auth-server(`https://api.moneyttuk.com`)로 회원가입/로그인 요청이 끝까지 통하도록 정합.
> **선행:** Phase 6(HTTPS + 도메인 + nginx + certbot)까지 완료된 상태.
> **완료일:** 2026-05-09

---

## 1. 최종 통신 구조

```
[브라우저]
  └─→ https://www.moneyttuk.com           (Vercel)
        └─ fetch ─► https://api.moneyttuk.com   (EC2 nginx :443)
                     └─ proxy_pass ─► auth-server:9001
                                         └─ JDBC ─► RDS authdb
```

CORS 처리 위치는 **nginx 단일 책임**으로 고정. Spring Security의 CORS 필터는 비활성화.

---

## 2. 사전 작업 체크리스트

| 항목 | 완료 여부 |
|------|-----------|
| RDS 테이블 생성 (`member`, `refresh_token`, `winning_numbers`, `purchase_games`) | ✅ |
| EBS 8GB → 20GB 확장 + 파티션/파일시스템 확장 | ✅ |
| nginx CORS 블록 작성 (`api.moneyttuk.com.conf`) | ✅ |
| Spring CORS 비활성화 (`SecurityConfig.java`) | ✅ |
| Vercel 프론트 환경변수 `API_BASE_URL=https://api.moneyttuk.com` | ✅ |

---

## 3. nginx CORS 블록 (운영 적용본)

`~/auth-server/nginx/conf.d/api.moneyttuk.com.conf` 의 443 server 블록 안에:

```nginx
# CORS 허용 origin 화이트리스트
set $cors_origin "";
if ($http_origin ~* "^https://(www\.)?moneyttuk\.com$") {
    set $cors_origin $http_origin;
}
if ($http_origin ~* "^http://localhost:3000$") {
    set $cors_origin $http_origin;
}

location / {
    # Preflight
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Requested-With" always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Max-Age 3600 always;
        return 204;
    }

    # Actual request
    add_header Access-Control-Allow-Origin $cors_origin always;
    add_header Access-Control-Allow-Credentials "true" always;
    add_header Access-Control-Expose-Headers "Authorization" always;

    proxy_pass         http://auth-server:9001;
    proxy_http_version 1.1;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host  $host;
}
```

---

## 4. Spring Security CORS 비활성화

**`auth-server/src/main/java/.../global/config/SecurityConfig.java`**

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http.csrf(AbstractHttpConfigurer::disable)
                .cors(AbstractHttpConfigurer::disable)
                .securityMatcher("/**")
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/auth/**").permitAll()
                        .requestMatchers("/actuator/**").permitAll()
                        .anyRequest().authenticated())
                .build();
    }
}
```

> `corsConfigurationSource()` 빈은 통째로 제거.
> 중복 헤더 사고 재발 방지를 위해 Spring CORS는 켜지 않는다.

---

## 5. RDS 스키마 적용 (방법)

운영 RDS에 로컬 개발 DB와 동일한 스키마를 그대로 옮긴다.

**절차:**

1. **로컬 MySQL에 SSH/터널 없이 접속** 후 대상 DB 선택
   ```sql
   USE member;
   SHOW TABLES;
   ```

2. **각 테이블의 DDL 추출**
   ```sql
   SHOW CREATE TABLE <table_name>;
   ```
   - `mysqldump --no-data` 로도 가능. 단일 파일로 내릴 거면 이쪽이 편하다:
     ```bash
     mysqldump -u root -p --no-data member > schema.sql
     ```

3. **EC2에서 RDS mysql 셸 접속** (RDS는 EC2 SG에서만 3306 허용)
   ```bash
   mysql -h auth-server-db.cxoiks04o8uv.ap-northeast-2.rds.amazonaws.com \
         -u <앱계정> -p authdb
   ```

4. **2번에서 추출한 DDL을 RDS에서 실행** → `SHOW TABLES;` 로 검증

> AUTO_INCREMENT 초깃값, COLLATION, ENGINE 등은 로컬 그대로 유지.
> 운영 데이터는 별도로 INSERT/마이그레이션 (스키마와 분리).

---

## 6. RDS GUI 접속 (DBeaver SSH 터널)

운영 중 데이터 확인 용도.

| 항목 | 값 |
|------|-----|
| Connection Type | MySQL |
| Host | `auth-server-db.cxoiks04o8uv.ap-northeast-2.rds.amazonaws.com` |
| Port | `3306` |
| Database | `authdb` |
| SSH Tunnel | 사용 |
| SSH Host | `43.202.153.119` |
| SSH User | `ubuntu` |
| SSH Auth | Public Key (`~/.ssh/moneyttuk.com.pem`) |

> RDS Security Group은 EC2 SG에서만 3306 허용. SSH 터널이 정석.

---

## 7. 재배포 절차

EBS 확장 후엔 prune 의무 없음. 다만 빌드 캐시가 누적되면 `docker system prune -a -f` 실행.

```bash
# 로컬
git push origin master

# EC2
cd ~/auth-server
git pull origin master
df -h /                          # 여유 확인 (1GB 미만이면 prune)
docker compose up -d --build auth-server
```

---

## 8. End-to-End 검증

### CORS preflight

```bash
curl -i -X OPTIONS https://api.moneyttuk.com/auth/sign-up \
  -H "Origin: https://www.moneyttuk.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type"
```

기대:
- `204 No Content`
- `Access-Control-Allow-Origin: https://www.moneyttuk.com` (정확히 1개)
- `Access-Control-Allow-Credentials: true`

### 회원가입 흐름

1. 프론트(`https://www.moneyttuk.com`)에서 가입 폼 제출
2. 브라우저 Network 탭에서 `POST /auth/sign-up` → `200/201` 확인
3. RDS `member` 테이블에 INSERT 확인:

```sql
SELECT member_id, email, nickname, provider, created_at
  FROM member
  ORDER BY created_at DESC
  LIMIT 5;
```

### 로그인 흐름

1. 가입한 계정으로 로그인 시도
2. 응답에 JWT 발급 확인
3. RDS `refresh_token` 테이블에 row 추가 확인

---

## 9. 알려진 미해결 TODO

- `IllegalArgumentException` 등 도메인 예외를 `@RestControllerAdvice` + `@ExceptionHandler`로 매핑하여 400/404 반환 (현재는 500으로 노출됨)
- 프론트 에러 메시지 토스트/알림 UX 개선
- access token 만료 시 refresh 흐름 통합 검증

> 트러블슈팅 이력은 `troubleshooting.md` 참조.
