# Phase 6: HTTPS + 도메인 연결 계획서 (moneyttuk.com)

> **목표:** moneyttuk.com 프론트(Vercel) + api.moneyttuk.com 백엔드(EC2)를 **완전 무료 범위**에서 HTTPS로 통합 운영
> **작성일:** 2026-05-04
> **대상 환경:** EC2 t3.micro (Ubuntu, 43.202.153.119) / RDS MySQL t4g.micro / Vercel Free / moneyttuk.com (Vercel DNS)

---

## 1. 아키텍처 구성도

```
                    ┌──────────────────────────────────┐
                    │         사용자 (브라우저)          │
                    └───────────┬──────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
     ┌─────────────────┐                ┌──────────────────┐
     │ moneyttuk.com   │                │ api.moneyttuk.com│
     │   (HTTPS)       │                │     (HTTPS)      │
     │   Vercel CDN    │                │  EC2 43.202.x.x  │
     └────────┬────────┘                └─────────┬────────┘
              │                                   │
              │ Next.js                           │ :443
              │                                   ▼
              │                            ┌──────────────┐
              │                            │   nginx      │
              │                            │  (Docker)    │
              │                            │ - SSL term.  │
              └─────── fetch ─────────────►│ - Reverse    │
                       (api.moneyttuk.com) │   Proxy      │
                                           └──────┬───────┘
                                                  │ http://auth-server:9001
                                                  ▼
                                           ┌──────────────┐
                                           │ auth-server  │
                                           │ Spring Boot  │
                                           │   :9001      │
                                           └──────┬───────┘
                                                  │ JDBC
                                                  ▼
                                           ┌──────────────┐
                                           │  RDS MySQL   │
                                           │  (private)   │
                                           └──────────────┘

[DNS]
moneyttuk.com         A     76.76.21.21              (Vercel - 기존)
www.moneyttuk.com     CNAME cname.vercel-dns.com     (Vercel - 기존)
api.moneyttuk.com     A     43.202.153.119           (EC2 - 신규 추가) ◄──

[SSL]
- moneyttuk.com       → Vercel 자동 발급 (Let's Encrypt, 기존)
- api.moneyttuk.com   → EC2 nginx + certbot 직접 발급 (Let's Encrypt, 신규)
```

### gateway 처리 방침

자체 구축한 gateway 모듈은 **AWS에 배포하지 않는다.**
단일 백엔드(auth-server)만 운영하므로 nginx 리버스 프록시로 충분히 대체 가능.

| 방식 | 채택 여부 | 이유 |
|------|-----------|------|
| nginx 단일 라우팅 | **채택** | 비용 0, 구성 단순, 레이턴시 최소 |
| Vercel API Routes BFF | 보류 | 함수 호출 한도, 토큰 보호 요건 생기면 점진 도입 |
| API Gateway | 미사용 | 프리티어 초과 우려, 비용 발생 |
| Route 53 | 미사용 | $0.5/hosted zone 유료 |

---

## 2. DNS 설정 (Vercel DNS)

moneyttuk.com 네임서버가 Vercel을 가리키므로 **Vercel Dashboard → Domains → moneyttuk.com → DNS Records** 에서 작업.

### 추가할 레코드

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `api` | `43.202.153.119` | 60 (검증용) → 3600 |

### 절차

1. Vercel Dashboard → 프로젝트 → **Settings → Domains**
2. moneyttuk.com 옆 **DNS** 또는 **Manage DNS Records** 클릭
3. **Add Record** 선택
   - Type: `A`
   - Name: `api`
   - Value: `43.202.153.119`
   - TTL: `60`
4. Save

### 검증

```bash
# 로컬에서 (전파 확인)
dig +short api.moneyttuk.com
# → 43.202.153.119 가 나와야 함

nslookup api.moneyttuk.com 8.8.8.8
```

> DNS 전파에 보통 1~5분, 길게는 30분 걸린다. 전파 확인 후 certbot 진행.

---

## 3. EC2 디렉터리 구조 준비

```bash
# EC2 SSH 접속
ssh -i ~/.ssh/moneyttuk.com.pem ubuntu@43.202.153.119

# 디렉터리 생성
mkdir -p ~/auth-server/nginx/conf.d
mkdir -p ~/auth-server/certbot/{conf,www,logs}
```

최종 구조:

```
~/auth-server/
├── docker-compose.yml
├── .env
├── nginx/
│   └── conf.d/
│       └── api.moneyttuk.com.conf
└── certbot/
    ├── conf/          ← /etc/letsencrypt 마운트
    ├── www/           ← ACME http-01 challenge 루트
    └── logs/
```

---

## 4. nginx 설정 파일

**~/auth-server/nginx/conf.d/api.moneyttuk.com.conf**

인증서 발급 전후로 두 단계로 나눠서 작성한다.

### Step 1: 인증서 발급 전 (HTTP only)

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name api.moneyttuk.com;

    # Let's Encrypt http-01 challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 'awaiting-certificate';
        add_header Content-Type text/plain;
    }
}
```

### Step 2: 인증서 발급 후 (HTTPS 리버스 프록시)

```nginx
# HTTP → HTTPS 강제 리다이렉트
server {
    listen 80;
    listen [::]:80;
    server_name api.moneyttuk.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 본진
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name api.moneyttuk.com;

    ssl_certificate     /etc/letsencrypt/live/api.moneyttuk.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.moneyttuk.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # 보안 헤더
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 10m;

    # 헬스체크
    location = /healthz {
        access_log off;
        proxy_pass http://auth-server:9001/actuator/health;
    }

    # 전체 라우팅 → auth-server
    location / {
        proxy_pass         http://auth-server:9001;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # WebSocket 대비
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> **CORS 주의:** CORS 헤더는 Spring Boot 한 곳에서만 처리한다. nginx에서 `Access-Control-Allow-Origin` 추가 시 헤더 중복으로 브라우저가 거부함.

---

## 5. docker-compose.yml 수정

기존 auth-server 단독 구성에 nginx + certbot 추가.
**핵심 변경: auth-server의 `ports`를 `expose`로 교체** → 외부 직접 접근 차단.

```yaml
services:
  auth-server:
    image: auth-server:latest
    container_name: auth-server
    restart: unless-stopped
    expose:
      - "9001"                    # ports 제거, expose로 내부 네트워크만 노출
    env_file:
      - .env
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:9001/actuator/health"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  nginx:
    image: nginx:1.27-alpine
    container_name: nginx
    restart: unless-stopped
    depends_on:
      auth-server:
        condition: service_healthy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    networks:
      - app-net

  certbot:
    image: certbot/certbot:latest
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
      - ./certbot/logs:/var/log/letsencrypt
    # 평소엔 안 뜸. 발급/갱신 시에만 `docker compose run --rm certbot ...` 실행

networks:
  app-net:
    driver: bridge
```

---

## 6. Let's Encrypt 인증서 발급

### Step A: nginx를 80번으로 먼저 띄우기 (Step 1 conf 상태)

```bash
cd ~/auth-server
docker compose up -d nginx
docker compose logs -f nginx    # 80 listen 확인
```

외부에서 검증:

```bash
# 로컬 PC에서
curl -I http://api.moneyttuk.com/
# → HTTP/1.1 200 awaiting-certificate
```

### Step B: dry-run 먼저 (발급 한도 보호)
- 인증서 발급이 일 5회로 제한되어 있는데 실수 등으로, 횟수 초과를 방지하기 위해 pre-test하는것

```bash
docker compose run --rm certbot \
  certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email homini0106@gmail.com \
  --agree-tos --no-eff-email \
  --dry-run \
  -d api.moneyttuk.com
```

`The dry run was successful` 메시지 확인 후 정식 발급.

### Step C: 정식 발급

```bash
docker compose run --rm certbot \
  certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email homini0106@gmail.com \
  --agree-tos --no-eff-email \
  -d api.moneyttuk.com
```

발급 성공 시 `~/auth-server/certbot/conf/live/api.moneyttuk.com/` 아래에 `fullchain.pem`, `privkey.pem` 생성됨.

### Step D: nginx conf를 Step 2(HTTPS 포함)로 교체 후 reload

```bash
# conf 파일 수정 (Step 2 내용으로 교체)
nano ~/auth-server/nginx/conf.d/api.moneyttuk.com.conf

# 문법 검사
docker compose exec nginx nginx -t

# reload (무중단)
docker compose exec nginx nginx -s reload
```

### Step E: HTTPS 검증

```bash
curl -I https://api.moneyttuk.com/healthz
# → HTTP/2 200, Strict-Transport-Security 헤더 확인
```

### Step F: 자동 갱신 cron 등록

```bash
crontab -e
```

```cron
# 매일 03:30 갱신 시도 (실제 갱신은 만료 30일 이내일 때만)
30 3 * * * cd /home/ubuntu/auth-server && /usr/bin/docker compose run --rm certbot renew --quiet && /usr/bin/docker compose exec nginx nginx -s reload >> /home/ubuntu/auth-server/certbot/logs/renew.log 2>&1
```

인증서 만료일 확인:

```bash
docker compose run --rm certbot certificates
```

---

## 7. Spring Boot CORS 설정

### CorsConfig.java 추가

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "https://moneyttuk.com",
            "https://www.moneyttuk.com",
            "http://localhost:3000"    // 로컬 개발용, 필요 없으면 제거
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        config.setExposedHeaders(List.of("Authorization"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

### SecurityFilterChain에 적용

```java
http.cors(Customizer.withDefaults())
    .csrf(csrf -> csrf.disable())
    // ...
```

### application.yml 추가 (X-Forwarded 헤더 신뢰)

```yaml
server:
  forward-headers-strategy: native
```

> `allowCredentials(true)` 사용 시 `allowedOrigins`에 `*` 사용 불가 — 명시적 도메인 나열 필수.

재빌드 후 배포:

```bash
cd ~/auth-server
docker compose up -d --build auth-server
```

CORS preflight 검증:

```bash
curl -i -X OPTIONS https://api.moneyttuk.com/api/v1/health \
  -H "Origin: https://moneyttuk.com" \
  -H "Access-Control-Request-Method: GET"
# → Access-Control-Allow-Origin: https://moneyttuk.com 확인
```

---

## 8. Vercel 환경변수 변경

Vercel Dashboard → 프로젝트 → **Settings → Environment Variables**

| Key | Value | Scope |
|-----|-------|-------|
| `NEXT_PUBLIC_API_BASE_URL` | `https://api.moneyttuk.com` | Production, Preview |
| `NEXT_PUBLIC_API_BASE_URL` | `http://localhost:9001` | Development |

변경 후 재배포:

```
Vercel → Deployments → 최신 배포 옆 ⋯ → Redeploy
```

또는 git에 아무 commit push (자동 재배포 트리거).

---

## 9. Security Group 정리

발급/검증 완료 후 9001 포트 외부 노출 차단.

**AWS Console → EC2 → Security Groups → auth-server SG → Inbound Rules**

| 규칙 | 액션 |
|------|------|
| TCP 22 (SSH) | 유지 (가능하면 내 IP만) |
| TCP 80 (HTTP) | 유지 (certbot 갱신용) |
| TCP 443 (HTTPS) | 유지 |
| TCP 9001 | **제거** ← nginx 경유만 허용 |

---

## 10. 체크리스트

### 6.1 DNS
- [x] Vercel DNS에 `api` A 레코드 (43.202.153.119) 추가
- [x] `dig +short api.moneyttuk.com` → 43.202.153.119 반환 확인
- [ ] TTL 검증 완료 후 3600으로 상향

### 6.2 EC2 디렉터리 + nginx 준비
- [x] `nginx/conf.d/`, `certbot/conf/www/logs/` 디렉터리 생성
- [x] Step 1 nginx conf 작성 (HTTP only)
- [x] docker-compose.yml 수정 (nginx/certbot 추가, auth-server ports → expose)
- [x] `docker compose up -d nginx`
- [x] `curl -I http://api.moneyttuk.com/` → 200 확인

### 6.3 Let's Encrypt 발급
- [x] dry-run 성공
- [x] 정식 발급 성공, `fullchain.pem` 존재 확인
- [x] nginx conf Step 2로 교체
- [x] `nginx -t` 통과 후 reload
- [x] `curl -I https://api.moneyttuk.com/healthz` → HTTP/2 200 + 자물쇠
- [x] crontab 자동 갱신 등록

### 6.4 Spring Boot CORS
- [ ] `CorsConfig.java` 작성, allowedOrigins 설정
- [ ] `forward-headers-strategy: native` 추가
- [ ] 이미지 재빌드 + 배포
- [ ] OPTIONS preflight → `Access-Control-Allow-Origin` 헤더 확인

### 6.5 Vercel 연결
- [ ] `NEXT_PUBLIC_API_BASE_URL=https://api.moneyttuk.com` 등록
- [ ] 재배포 완료
- [ ] 브라우저에서 실제 API 호출 → 200, CORS 에러 없음

### 6.6 보안 정리
- [ ] Security Group에서 9001 인바운드 규칙 제거
- [ ] `.env` 권한 600, git 추적 제외 확인
- [ ] 보안 헤더 (`HSTS`, `X-Frame-Options` 등) 응답 확인

---

## 11. 비용 점검 (월 $0)

| 항목 | 무료 한도 |
|------|-----------|
| EC2 t3.micro | 프리티어 750h/월 ✅ |
| RDS t4g.micro | 프리티어 750h/월 ✅ |
| Elastic IP | EC2 연결 시 무료 ✅ |
| Route 53 | 미사용 ✅ |
| API Gateway | 미사용 ✅ |
| Let's Encrypt | 무료 ✅ |
| Vercel Hobby | 무료 ✅ |

> EC2 프리티어 만료(12개월) 후 약 $7~10/월 발생. 이때 Oracle Cloud Free Tier(ARM) 또는 Lightsail $3.5로 이전 고려.

---

## 12. 자주 묻는 질문 (Q&A)

**Q. A 레코드 추가가 무슨 의미인가요?**
> "이 도메인 이름 → 이 IP 주소로 연결해라"는 설정이다. `api.moneyttuk.com`이 EC2 엘라스틱 IP(43.202.153.119)를 가리키도록 등록하는 것으로, 도메인과 서버를 연결하는 작업이다.

**Q. A 레코드를 Vercel에서 설정하는 건가요? 도메인은 후이즈에서 샀는데요.**
> DNS 설정 위치는 도메인을 산 곳이 아니라 **네임서버가 어디냐**에 따라 결정된다. 네임서버가 Vercel로 위임되어 있으면 Vercel에서, 후이즈 그대로면 후이즈에서 설정한다. `nslookup -type=NS moneyttuk.com 8.8.8.8` 으로 확인 가능.

**Q. 서브도메인을 다는 건가요?**
> 맞다. `api.moneyttuk.com`이 서브도메인이다. `moneyttuk.com`(프론트, Vercel)과 `api.moneyttuk.com`(백엔드, EC2)을 분리해서 운영한다.

**Q. Vercel의 HTTPS 인증서는 따로 발급해야 하나요?**
> Vercel은 도메인 연결 시 Let's Encrypt 인증서를 자동으로 발급하고 갱신해준다. 별도 작업 불필요. EC2 쪽(`api.moneyttuk.com`)만 직접 certbot으로 발급한다.

**Q. Let's Encrypt 인증서는 유료인가요?**
> 완전 무료다. 90일마다 갱신이 필요하지만 crontab으로 자동화해두기 때문에 신경 쓸 필요 없다.

**Q. dry-run이 뭔가요?**
> 실제 인증서를 발급하지 않고 발급 가능 여부만 테스트하는 것이다. Let's Encrypt는 하루 발급 한도(5회)가 있어서, 설정 실수로 계속 실패하면 한도가 소진된다. dry-run은 한도에 포함되지 않으므로 정식 발급 전에 반드시 먼저 실행한다.

**Q. docker compose up 시 auth-server가 unhealthy로 뜨는 이유는?**
> docker-compose.yml이 변경되면 기존 컨테이너를 내리고 새로 띄운다. Spring Boot 재기동에 약 13초가 걸리는데, 그 사이에 헬스체크가 실패해서 unhealthy로 표시된다. 실제로는 기동 완료 후 정상 동작한다. 근본 원인은 컨테이너 이미지에 `curl`이 없어서 헬스체크 명령이 실패하는 것으로, 추후 `wget` 방식으로 교체 예정.

**Q. 이 구성에서 HTTPS 흐름이 어떻게 되나요?**
> 브라우저 → `moneyttuk.com`(Vercel, HTTPS 자동) → 프론트에서 `https://api.moneyttuk.com` 호출 → EC2 nginx(443, HTTPS) → Spring Boot(9001, 내부 네트워크). 외부에서 Spring Boot에 직접 접근하는 경로는 없다.

---

## 13. 트러블슈팅

| 증상 | 점검 |
|------|------|
| HTTPS timeout | Security Group 443 인바운드 확인 |
| certbot challenge 실패 | DNS 전파 미완료 / nginx 80 미동작 / `/.well-known/acme-challenge/` 라우팅 누락 |
| 502 Bad Gateway | auth-server 다운 / expose 누락 / 네트워크 분리 |
| CORS 에러 (헤더 중복) | nginx + Spring 양쪽에서 CORS 추가 중 → Spring 한 곳만 |
| Mixed Content | Vercel에서 `http://` 호출 → `https://api.moneyttuk.com`으로 변경 |
| 인증서 갱신 실패 | cron PATH 문제 → docker/docker compose 절대경로로 지정 |
| 9001 직접 접근됨 | Security Group에 9001 규칙 남아있음 → 삭제 |
