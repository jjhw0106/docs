# AWS 배포 통합 매뉴얼 (Spring Boot + Docker + nginx + RDS + Vercel)

> 단일 백엔드(Spring Boot) + 단일 RDS + Vercel 프론트 구성을 AWS에 무료/저비용으로 배포하기 위한 통합 가이드.
> 특정 도메인/IP/계정명은 모두 플레이스홀더(`<...>`)로 표기하므로 실제 값으로 치환해서 사용한다.

## 플레이스홀더

| 표기 | 의미 | 예시 |
|------|------|------|
| `<DOMAIN>` | 보유 도메인 | `example.com` |
| `<API_SUBDOMAIN>` | API용 서브도메인 | `api.example.com` |
| `<EC2_ELASTIC_IP>` | EC2 탄력적 IP | `43.x.x.x` |
| `<KEY_FILE>` | EC2 키 페어 파일명 | `mykey.pem` |
| `<APP_NAME>` | 앱 컨테이너/이미지 이름 | `auth-server` |
| `<DB_NAME>` | 운영 DB 이름 | `authdb` |
| `<DB_USER>` | 앱용 DB 계정 | `appuser` |
| `<DB_PASSWORD>` | 앱용 DB 비밀번호 | (비공개) |
| `<RDS_ENDPOINT>` | RDS 엔드포인트 | `xxx.rds.amazonaws.com` |
| `<EMAIL>` | Let's Encrypt 등록 이메일 | `me@example.com` |
| `<REPO_URL>` | GitHub 레포 URL | `https://github.com/.../app.git` |

---

## 목차

- [최종 아키텍처](#최종-아키텍처)
- [Phase 0. AWS 기초 세팅](#phase-0-aws-기초-세팅)
- [Phase 1-2. EC2 생성 + Docker 설치](#phase-1-2-ec2-생성--docker-설치)
- [Phase 3. RDS MySQL 생성](#phase-3-rds-mysql-생성)
- [Phase 4. 배포 파일 작성](#phase-4-배포-파일-작성)
- [Phase 5. EC2 배포 실행](#phase-5-ec2-배포-실행)
- [Phase 6. HTTPS + 도메인 연결 (nginx + certbot)](#phase-6-https--도메인-연결)
- [Phase 7. 프론트 연결 + 운영 정합](#phase-7-프론트-연결--운영-정합)
- [부록 A. 트러블슈팅](#부록-a-트러블슈팅)
- [부록 B. 비용 점검](#부록-b-비용-점검)
- [부록 C. 자주 묻는 질문](#부록-c-자주-묻는-질문)

---

## 최종 아키텍처

```
[Vercel 프론트 / <DOMAIN>]
   │  HTTPS
   └──► [EC2 / <API_SUBDOMAIN> / <EC2_ELASTIC_IP>]
            └─ Docker compose
                 ├─ nginx (443, 80)        ← TLS 종료 + 리버스 프록시 + CORS
                 ├─ <APP_NAME> (9001)      ← Spring Boot, 내부 네트워크만 expose
                 └─ certbot                ← Let's Encrypt 발급/갱신 (필요 시 1회 실행)
                       │
                       └─ JDBC ─► [RDS MySQL <DB_NAME> (3306, EC2 SG에서만 허용)]

[DNS]
<DOMAIN>            A     <Vercel CNAME 또는 IP>
<API_SUBDOMAIN>     A     <EC2_ELASTIC_IP>

[SSL]
<DOMAIN>            → Vercel 자동 발급 (Let's Encrypt)
<API_SUBDOMAIN>     → EC2 nginx + certbot 직접 발급
```

**핵심 원칙:**
- CORS 처리는 **한 곳에서만** (이 매뉴얼에서는 nginx). 두 곳에서 처리하면 헤더 중복으로 브라우저가 거부.
- 앱 컨테이너는 외부에 직접 노출하지 않음 (`ports` 대신 `expose`).
- RDS는 EC2 Security Group에서만 3306 허용. 인터넷 직접 접근 차단.

---

## Phase 0. AWS 기초 세팅

### 0.1 루트 계정 MFA 설정

> 루트 계정은 AWS 전체 제어권을 가진 계정. 유출 시 요금 폭탄 및 데이터 삭제 위험.

1. AWS 콘솔 → 루트 계정으로 로그인
2. 우측 상단 계정명 → **보안 자격 증명**
3. **멀티 팩터 인증(MFA)** → **MFA 디바이스 할당** → **Authenticator app**
4. Google Authenticator 또는 Authy로 QR 스캔 후 6자리 코드 두 번 입력

### 0.2 IAM 사용자 생성

> 평소 작업은 루트 대신 IAM 사용자로. 실수/유출 시 피해 범위 제한.

1. **IAM** → **사용자** → **사용자 생성**
2. 사용자 이름 입력
3. **AWS Management Console에 대한 사용자 액세스 권한 제공** 체크
4. 권한: **직접 정책 연결** → `AdministratorAccess` 선택
5. **사용자 생성**

### 0.3 IAM 로그인 URL 북마크

1. IAM → **대시보드** → **로그인 URL** 복사
2. 로그아웃 후 IAM 사용자로 재로그인

> 루트 로그인: 이메일 / IAM 로그인: 위 URL + 사용자명 + 비밀번호

### 0.4 IAM 사용자 결제 액세스 허용

> IAM 사용자는 기본적으로 결제 정보 접근 불가.

1. 루트 계정 로그인 → 우측 상단 → **계정**
2. **결제 정보에 대한 IAM 사용자 및 역할 액세스** → **활성화**

### 0.5 예산 알림 설정

1. **Cost Management** → **Budgets** → **예산 생성**
2. **월별 비용 예산** → 임계 금액 설정 (예: $20)
3. 알림 이메일 등록

### 체크리스트

- [ ] 루트 MFA 활성화
- [ ] IAM 사용자 생성 (AdministratorAccess)
- [ ] IAM 로그인 URL 북마크
- [ ] 결제 IAM 액세스 허용
- [ ] 예산 알림 설정

---

## Phase 1-2. EC2 생성 + Docker 설치

### 1.1 EC2 인스턴스 생성

1. 콘솔 → **EC2** → 우측 상단 리전 선택 (예: 서울 `ap-northeast-2`)
2. **인스턴스 시작**

| 항목 | 값 |
|------|-----|
| 이름 | `<APP_NAME>` |
| AMI | Ubuntu Server LTS |
| 인스턴스 유형 | `t3.micro` (프리티어) |
| 스토리지 | 8GB로 시작 → 필요 시 20GB로 확장 (프리티어 30GB까지 무료) |

> 처음부터 20GB로 만들어도 됨. Gradle 빌드 캐시가 누적되면 8GB는 빠르게 부족해진다.

### 1.2 키 페어 생성

1. **새 키 페어 생성**
2. 이름: `<KEY_FILE>` (`.pem` 형식, RSA)
3. 다운로드 → 안전한 폴더 보관 (git 커밋 금지, 분실 시 재발급 불가)

### 1.3 Security Group

| 포트 | 프로토콜 | 소스 | 용도 |
|------|---------|------|------|
| 22 | SSH | 내 IP | 관리자 접속 |
| 80 | HTTP | 0.0.0.0/0 | certbot http-01 challenge / nginx 리다이렉트 |
| 443 | HTTPS | 0.0.0.0/0 | API 서비스 |
| 9001 | TCP | 0.0.0.0/0 | (배포 검증 한정) — 운영 안정화 후 **제거** |

> SSH는 유동 IP 환경(카페/공유기 변경)에서는 자주 막힌다. 막히면 인바운드 규칙의 SSH 소스를 현재 IP로 갱신.

### 1.4 Elastic IP 할당

1. EC2 좌측 메뉴 **탄력적 IP** → **할당** → 인스턴스에 **연결**

> Elastic IP가 없으면 EC2 재시작 시 IP가 바뀌어 DNS와 어긋난다.

### 1.5 SSH 접속

```bash
# 키 파일을 WSL 홈 .ssh로 복사 (chmod 적용 위해)
cp "/mnt/c/path/to/<KEY_FILE>" ~/.ssh/
chmod 400 ~/.ssh/<KEY_FILE>

# 접속
ssh -i ~/.ssh/<KEY_FILE> ubuntu@<EC2_ELASTIC_IP>
```

> `/mnt/c|d/...` 경로의 키 파일은 chmod가 적용 안 됨 → 반드시 WSL 홈으로 복사.

### 1.6 Docker 설치 (EC2 내부)

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker ubuntu
exit                  # 그룹 적용을 위해 재접속
```

재접속 후 확인:

```bash
docker ps
docker compose version
```

### 체크리스트

- [ ] EC2 t3.micro 생성 (Ubuntu LTS)
- [ ] 키 페어 안전 보관 + chmod 400
- [ ] Security Group 22/80/443/9001 인바운드 설정
- [ ] Elastic IP 할당 및 연결
- [ ] SSH 접속 성공
- [ ] Docker + Compose v2 동작 확인

---

## Phase 3. RDS MySQL 생성

### 3.1 DB 서브넷 그룹 생성

1. **RDS** → **서브넷 그룹** → **DB 서브넷 그룹 생성**
2. 기본 VPC 선택, **가용 영역 2개 이상** 포함 (예: `ap-northeast-2a`, `ap-northeast-2c`)

### 3.2 RDS 전용 Security Group

EC2 콘솔 → **보안 그룹** → **생성**

| 항목 | 값 |
|------|-----|
| 이름 | `rds-sg` (관례) |
| 인바운드 | MySQL/Aurora 3306, 소스: **EC2의 SG ID** |

> **소스에 IP 대신 EC2 SG ID를 입력**하면 그 SG가 붙은 리소스에서 오는 트래픽만 허용 → EC2만 RDS에 접근 가능.

### 3.3 RDS 인스턴스 생성

1. RDS → **데이터베이스 생성**

| 항목 | 값 |
|------|-----|
| 엔진 | MySQL 8.0.x |
| 템플릿 | **프리 티어** |
| DB 인스턴스 식별자 | `<APP_NAME>-db` |
| 마스터 사용자 | `admin` |
| 마스터 암호 | 강력 (기록 필수) |
| 인스턴스 유형 | `db.t3.micro` |
| 스토리지 | 20GB / gp2 / 자동 조정 비활성화 |
| VPC | 기본 VPC |
| 서브넷 그룹 | 위에서 만든 그룹 |
| 퍼블릭 액세스 | **아니요** |
| VPC 보안 그룹 | **기존 항목 선택** → `rds-sg` |
| 초기 DB 이름 | `<DB_NAME>` |
| 자동 백업 | 비활성화 (학습 목적) |
| 마이너 자동 업그레이드 | 비활성화 |

생성에 5~10분 소요.

### 3.4 엔드포인트 확인

RDS → 데이터베이스 → 인스턴스 클릭 → **연결 & 보안** → **엔드포인트** 복사 → `<RDS_ENDPOINT>`.

### 3.5 EC2에서 RDS 연결 테스트

```bash
sudo apt install -y mysql-client
mysql -h <RDS_ENDPOINT> -u admin -p
```

```sql
SHOW DATABASES;
USE <DB_NAME>;
```

### 3.6 앱 전용 DB 유저 생성

```sql
CREATE USER '<DB_USER>'@'%' IDENTIFIED BY '<DB_PASSWORD>';
GRANT ALL PRIVILEGES ON <DB_NAME>.* TO '<DB_USER>'@'%';
FLUSH PRIVILEGES;
```

> admin 계정을 앱에 직접 물리는 것보다 권한 분리가 안전.

### 체크리스트

- [ ] DB 서브넷 그룹 생성 (AZ 2개 이상)
- [ ] `rds-sg` 생성 (EC2 SG에서만 3306 허용)
- [ ] RDS 인스턴스 생성 + 엔드포인트 확인
- [ ] EC2에서 mysql 접속 성공
- [ ] 앱 전용 DB 유저 생성 + 권한 부여

---

## Phase 4. 배포 파일 작성

### 4.1 핵심 개념

| 파일 | 역할 |
|------|------|
| `Dockerfile` | 이미지 빌드 레시피 |
| `docker-compose.yml` | 컨테이너 실행 정의 (서비스, 네트워크, 볼륨) |
| `.env` | 실제 비밀값 (git 제외) |
| `.env.example` | 필요한 변수 목록 (git 포함) |
| `.dockerignore` | 빌드 컨텍스트 제외 (캐시·`.env` 누출 방지) |

### 4.2 Dockerfile (멀티스테이지)

```dockerfile
# Stage 1: 빌드 (JDK 필요)
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew
RUN ./gradlew dependencies --no-daemon
COPY src src
RUN ./gradlew bootJar --no-daemon

# Stage 2: 실행 (JRE만)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 9001
CMD ["java", "-jar", "app.jar"]
```

> 빌드 도구는 실행에 불필요. JRE 이미지로 분리해 최종 이미지 크기 축소.

### 4.3 docker-compose.yml (단일 서비스 시)

```yaml
services:
  <APP_NAME>:
    build: .
    ports:
      - "9001:9001"        # 운영 안정화 후 expose로 교체
    env_file:
      - .env
    restart: unless-stopped
```

| restart 옵션 | 동작 |
|--------------|------|
| `no` | 재시작 안 함 |
| `always` | 항상 |
| `unless-stopped` | 수동 중지 외 항상 (권장) |
| `on-failure` | 실패 시만 |

### 4.4 .env.example

```env
SPRING_PROFILES_ACTIVE=prod

DB_URL=jdbc:mysql://<RDS_ENDPOINT>:3306/<DB_NAME>?useSSL=false&serverTimezone=Asia/Seoul
DB_USERNAME=<DB_USER>
DB_PASSWORD=<DB_PASSWORD>

JWT_SECRET=<random_secret>
```

### 4.5 .dockerignore

```
.gradle
build
.env
*.md
.gitignore
```

### 4.6 .env / .gitignore / .dockerignore 관계

```
.env                ← 실제 비밀값 파일
 ├── .gitignore     ← git 추적 제외
 └── .dockerignore  ← Docker 이미지 빌드 시 제외
```

`.env`를 실수로 커밋했다면:
1. `git rm --cached .env`
2. GitHub에 이미 푸시됐다면 **비밀값 자체를 모두 교체** (노출된 것으로 간주)

### 4.7 Spring 프로파일 분리

```
application.yml          ← 항상 로드 (공통)
application-local.yml    ← SPRING_PROFILES_ACTIVE=local
application-dev.yml      ← SPRING_PROFILES_ACTIVE=dev
application-prod.yml     ← SPRING_PROFILES_ACTIVE=prod
```

활성화 우선순위 (낮 → 높):
1. `application.yml` 내 `spring.profiles.active`
2. JVM 옵션 `-Dspring.profiles.active=...`
3. 환경변수 `SPRING_PROFILES_ACTIVE=...` ← Docker에서 주로 사용
4. IDE Run Configuration

| 환경 | 프로파일 | 활성화 방법 | DB |
|------|---------|------------|-----|
| 로컬 | `local` | IDE Run Config | 로컬 MySQL |
| 운영 | `prod` 또는 `dev` | `.env` → docker-compose 주입 | RDS |

### 체크리스트

- [ ] Dockerfile 멀티스테이지 작성
- [ ] docker-compose.yml 작성
- [ ] `.env.example` 작성
- [ ] `.dockerignore` 작성
- [ ] `.gitignore`에 `.env` 포함 확인
- [ ] GitHub push

---

## Phase 5. EC2 배포 실행

### 5.1 SSH 접속 후 git clone

```bash
ssh -i ~/.ssh/<KEY_FILE> ubuntu@<EC2_ELASTIC_IP>

git --version || sudo apt install -y git
git clone <REPO_URL>
cd <APP_NAME>
```

### 5.2 .env 작성

```bash
cp .env.example .env
nano .env       # 실제 값 채워넣기
```

> `nano` 저장: `Ctrl+O` → Enter → `Ctrl+X`

### 5.3 빌드 + 실행

```bash
docker compose up -d --build
```

| 옵션 | 의미 |
|------|------|
| `up` | 컨테이너 시작 |
| `-d` | detached (백그라운드) |
| `--build` | 이미지 새로 빌드 |

> 첫 빌드는 Gradle 의존성 다운로드로 5~10분 소요.

### 5.4 로그 / 헬스체크

```bash
docker compose logs -f
docker compose ps

curl localhost:9001/actuator/health
# {"status":"UP"}
```

### 5.5 외부 접속 검증

```bash
curl http://<EC2_ELASTIC_IP>:9001/actuator/health
```

> 안 되면 Security Group 9001 인바운드 확인.

### 체크리스트

- [ ] git clone 완료
- [ ] `.env` 작성 완료
- [ ] `docker compose up -d --build` 성공
- [ ] `localhost:9001/actuator/health` 200
- [ ] 외부 IP 9001 헬스체크 200

---

## Phase 6. HTTPS + 도메인 연결

> 이 시점부터 nginx 리버스 프록시를 추가하고, certbot으로 Let's Encrypt 인증서를 발급받아 HTTPS를 끝낸다.

### 6.1 DNS A 레코드 추가

도메인을 산 곳이 아니라 **네임서버가 가리키는 곳**에서 DNS를 관리한다. (Vercel DNS / Cloudflare / Route53 등)

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `api` | `<EC2_ELASTIC_IP>` | 60 (검증) → 3600 |

검증:

```bash
dig +short <API_SUBDOMAIN>
# → <EC2_ELASTIC_IP>
nslookup <API_SUBDOMAIN> 8.8.8.8
```

> 전파 1~5분, 길게는 30분.

### 6.2 EC2 디렉터리 구조

```bash
mkdir -p ~/<APP_NAME>/nginx/conf.d
mkdir -p ~/<APP_NAME>/certbot/{conf,www,logs}
```

```
~/<APP_NAME>/
├── docker-compose.yml
├── .env
├── nginx/
│   └── conf.d/
│       └── <API_SUBDOMAIN>.conf
└── certbot/
    ├── conf/    ← /etc/letsencrypt 마운트
    ├── www/     ← ACME http-01 challenge 루트
    └── logs/
```

### 6.3 nginx 설정 — Step 1 (인증서 발급 전)

`~/<APP_NAME>/nginx/conf.d/<API_SUBDOMAIN>.conf`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name <API_SUBDOMAIN>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 'awaiting-certificate';
        add_header Content-Type text/plain;
    }
}
```

### 6.4 docker-compose.yml 수정 (nginx + certbot 추가)

> **핵심 변경: 앱 컨테이너의 `ports` → `expose`** (외부 직접 접근 차단)

```yaml
services:
  <APP_NAME>:
    image: <APP_NAME>:latest
    container_name: <APP_NAME>
    restart: unless-stopped
    expose:
      - "9001"
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
      <APP_NAME>:
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
    # 평소엔 안 뜸. 발급/갱신 시 `docker compose run --rm certbot ...`로만 실행

networks:
  app-net:
    driver: bridge
```

### 6.5 nginx 80번 먼저 띄우기

```bash
cd ~/<APP_NAME>
docker compose up -d nginx
docker compose logs -f nginx
```

외부 검증:

```bash
curl -I http://<API_SUBDOMAIN>/
# HTTP/1.1 200 awaiting-certificate
```

### 6.6 Let's Encrypt 발급

**Step A. dry-run** (발급 한도 보호 — 일 5회 제한, dry-run은 카운트 안 됨)

```bash
docker compose run --rm certbot \
  certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email <EMAIL> \
  --agree-tos --no-eff-email \
  --dry-run \
  -d <API_SUBDOMAIN>
```

`The dry run was successful` 확인.

**Step B. 정식 발급**

```bash
docker compose run --rm certbot \
  certonly --webroot \
  --webroot-path=/var/www/certbot \
  --email <EMAIL> \
  --agree-tos --no-eff-email \
  -d <API_SUBDOMAIN>
```

발급 성공 시 `~/<APP_NAME>/certbot/conf/live/<API_SUBDOMAIN>/`에 `fullchain.pem`, `privkey.pem` 생성.

### 6.7 nginx 설정 — Step 2 (HTTPS 본진)

```nginx
# HTTP → HTTPS 강제 리다이렉트
server {
    listen 80;
    listen [::]:80;
    server_name <API_SUBDOMAIN>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name <API_SUBDOMAIN>;

    ssl_certificate     /etc/letsencrypt/live/<API_SUBDOMAIN>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<API_SUBDOMAIN>/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 10m;

    # CORS — 단일 책임 (앱에서는 비활성화)
    set $cors_origin "";
    if ($http_origin ~* "^https://(www\.)?<DOMAIN>$") {
        set $cors_origin $http_origin;
    }
    if ($http_origin ~* "^http://localhost:3000$") {
        set $cors_origin $http_origin;
    }

    # 헬스체크
    location = /healthz {
        access_log off;
        proxy_pass http://<APP_NAME>:9001/actuator/health;
    }

    location / {
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type, X-Requested-With" always;
            add_header Access-Control-Allow-Credentials "true" always;
            add_header Access-Control-Max-Age 3600 always;
            return 204;
        }

        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Access-Control-Expose-Headers "Authorization" always;

        proxy_pass         http://<APP_NAME>:9001;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

리로드:

```bash
docker compose exec nginx nginx -t       # 문법 검사
docker compose exec nginx nginx -s reload
```

### 6.8 HTTPS 검증

```bash
curl -I https://<API_SUBDOMAIN>/healthz
# HTTP/2 200, Strict-Transport-Security 헤더 확인
```

### 6.9 Spring Security CORS는 끈다

nginx에서 단일 처리하므로 앱 쪽 CORS는 비활성화한다 (헤더 중복 방지).

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

`application.yml`:

```yaml
server:
  forward-headers-strategy: native
```

### 6.10 인증서 자동 갱신 cron

```bash
crontab -e
```

```cron
30 3 * * * cd /home/ubuntu/<APP_NAME> && /usr/bin/docker compose run --rm certbot renew --quiet && /usr/bin/docker compose exec nginx nginx -s reload >> /home/ubuntu/<APP_NAME>/certbot/logs/renew.log 2>&1
```

확인:

```bash
docker compose run --rm certbot certificates
```

### 6.11 Security Group 정리

| 규칙 | 액션 |
|------|------|
| 22 SSH | 유지 (가능하면 내 IP만) |
| 80 HTTP | 유지 (HTTPS 리다이렉트 + certbot) |
| 443 HTTPS | 유지 |
| 9001 | **제거** (nginx 경유만 허용) |

### 체크리스트

- [ ] DNS A 레코드 추가 + 전파 확인
- [ ] nginx Step 1 conf로 80 띄우기
- [ ] dry-run 통과
- [ ] 정식 발급 + `fullchain.pem` 생성
- [ ] nginx Step 2 conf로 교체 + reload
- [ ] HTTPS 200 + 자물쇠
- [ ] crontab 자동 갱신 등록
- [ ] Spring CORS 비활성화 후 재배포
- [ ] Security Group 9001 제거

---

## Phase 7. 프론트 연결 + 운영 정합

### 7.1 프론트 환경변수

Vercel Dashboard → 프로젝트 → **Settings → Environment Variables**

| Key | Value | Scope |
|-----|-------|-------|
| `NEXT_PUBLIC_API_BASE_URL` | `https://<API_SUBDOMAIN>` | Production, Preview |
| `NEXT_PUBLIC_API_BASE_URL` | `http://localhost:9001` | Development |

변경 후 재배포 (Deployments → Redeploy 또는 git push).

### 7.2 RDS 스키마 적용

운영 RDS에 로컬 DB 스키마를 그대로 옮긴다.

1. **로컬 MySQL에 접속** 후 대상 DB 선택 → 각 테이블 DDL 추출
   ```sql
   USE <local_db>;
   SHOW TABLES;
   SHOW CREATE TABLE <table_name>;
   ```
   또는 mysqldump (스키마만):
   ```bash
   mysqldump -u root -p --no-data <local_db> > schema.sql
   ```

2. **EC2에서 RDS mysql 접속** (RDS는 EC2 SG에서만 3306 허용)
   ```bash
   mysql -h <RDS_ENDPOINT> -u <DB_USER> -p <DB_NAME>
   ```

3. **DDL 실행** → `SHOW TABLES;`로 검증

> AUTO_INCREMENT 초깃값, COLLATION, ENGINE 등은 그대로 유지.

### 7.3 RDS GUI 접속 (DBeaver SSH 터널)

운영 데이터 확인 용도. RDS 보안 그룹은 EC2에서만 3306을 허용하므로 SSH 터널이 정석.

| 항목 | 값 |
|------|-----|
| Connection Type | MySQL |
| Host | `<RDS_ENDPOINT>` |
| Port | `3306` |
| Database | `<DB_NAME>` |
| SSH Tunnel | 사용 |
| SSH Host | `<EC2_ELASTIC_IP>` |
| SSH User | `ubuntu` |
| SSH Auth | Public Key (`~/.ssh/<KEY_FILE>`) |

### 7.4 재배포 표준 절차

```bash
# 로컬
git push origin master

# EC2
cd ~/<APP_NAME>
git pull origin master
df -h /                          # 여유 1GB 미만이면 prune
docker compose up -d --build <APP_NAME>
```

### 7.5 End-to-End 검증

**CORS preflight:**

```bash
curl -i -X OPTIONS https://<API_SUBDOMAIN>/auth/sign-up \
  -H "Origin: https://www.<DOMAIN>" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type"
```

기대: `204`, `Access-Control-Allow-Origin` 정확히 1개.

**회원가입:** 프론트에서 가입 → Network 200/201 → RDS 테이블 INSERT 확인.
**로그인:** JWT 발급 → refresh_token row 추가 확인.

### 체크리스트

- [ ] Vercel 환경변수 등록 + 재배포
- [ ] RDS 스키마 적용 + 검증
- [ ] DBeaver SSH 터널 접속 가능
- [ ] 회원가입/로그인 E2E 통과
- [ ] CORS 헤더 단일성 검증

---

## 부록 A. 트러블슈팅

### A.1 SSH 접속 timeout
유동 IP 환경에서 자주 발생. EC2 Security Group의 SSH(22) 인바운드 소스를 **현재 내 IP**로 갱신.

### A.2 RDS 연결 실패 (ERROR 2003 / 110)
RDS에 붙은 SG가 `default`인 경우. RDS → 수정 → 보안 그룹에서 `default` 제거하고 `rds-sg`로 교체.

> Security Group은 이름이 아닌 **인바운드/아웃바운드 정책 묶음**이다. 소스에 IP 대신 다른 SG를 지정하면 "그 SG가 붙은 리소스에서 오는 트래픽만 허용"된다.

### A.3 GRANT 실패 (ERROR 1046 No database selected)
`<DB_NAME>`이 미존재 또는 GRANT 문법에서 `.*` 누락.

```sql
CREATE DATABASE <DB_NAME>;
GRANT ALL PRIVILEGES ON <DB_NAME>.* TO '<DB_USER>'@'%';
FLUSH PRIVILEGES;
```

### A.4 docker compose 명령어 없음
```bash
docker compose version || sudo apt install -y docker-compose-plugin
```

### A.5 t3.micro Gradle OOM
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### A.6 컨테이너 즉시 종료
```bash
docker compose logs <APP_NAME>
```
대부분 `.env`의 DB 연결 정보 오타.

### A.7 외부 9001 접속 안 됨
Security Group 9001 인바운드 누락. 운영 정합 후엔 의도적으로 제거.

### A.8 재배포 시 Gradle 빌드 실패 (디스크 부족)

```
Could not update /app/.gradle/.../last-build.bin
Could not receive a message from the daemon.
BUILD FAILED
```

소스 수정 전엔 Docker 레이어 캐시로 bootJar를 건너뛰다가, 변경 시 첫 빌드에서 디스크 부족이 드러난다.

**응급:**
```bash
df -h /
docker system prune -a -f       # -a 필수 (이미지까지 삭제)
docker compose up -d --build <APP_NAME>
```

**근본:** EBS 8GB → 20GB 확장 (프리티어 30GB 이내 무료) + 아래 A.9 작업 필수.

### A.9 EBS 볼륨 확장 후 디스크가 늘어나지 않음

AWS 콘솔에서 볼륨을 늘려도 EC2 OS는 여전히 옛 크기로 인식. EBS 확장은 2단계 작업이다.

| 단계 | 설명 | 비유 |
|------|------|------|
| 1 (콘솔) | EBS 볼륨 자체 크기 변경 | 하드디스크 교체 |
| 2 (OS) | 파티션 + 파일시스템 확장 | 늘어난 공간을 OS가 인식 |

```bash
lsblk                                # 디바이스명 확인 (Nitro: nvme0n1, 구형: xvda)
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
```

### A.10 CORS preflight 403 (Spring Security 차단)
Spring CORS allowedOrigins에 운영 도메인이 빠진 경우. 단, 이 매뉴얼에서는 **CORS를 nginx에 위임하고 Spring CORS는 비활성화**하는 것이 표준.

### A.11 CORS 헤더 중복

```
The 'Access-Control-Allow-Origin' header contains multiple values
'https://www.<DOMAIN>, https://www.<DOMAIN>', but only one is allowed.
```

nginx와 Spring 양쪽에서 헤더를 추가 중. 한 곳만 처리:

```java
return http.cors(AbstractHttpConfigurer::disable)
           .csrf(AbstractHttpConfigurer::disable)
           // ...
           .build();
```

`corsConfigurationSource()` 빈은 통째로 제거. 재배포 후 응답 헤더가 정확히 1개인지 curl로 확인.

### A.12 502 Bad Gateway
앱 컨테이너 다운 / `expose` 누락 / 다른 네트워크 → `docker compose ps`, `logs`, 네트워크 정합 확인.

### A.13 Mixed Content
프론트에서 `http://`로 호출 중. `https://<API_SUBDOMAIN>`로 변경.

### A.14 인증서 갱신 실패
cron PATH 문제. `docker`/`docker compose`를 절대경로로 지정.

### A.15 unhealthy 표시 (실제로는 정상)
컨테이너 이미지에 `curl`이 없어 healthcheck 실패. `wget` 또는 명시적 `apk add curl`로 교체.

### A.16 WSL Ubuntu에서 Windows MySQL 접속 불가
WSL2는 Windows와 별도 네트워크. `127.0.0.1`은 Ubuntu 자기 자신을 가리킴.

1. Windows 호스트 IP 확인 (WSL):
   ```bash
   cat /etc/resolv.conf | grep nameserver
   ```
2. Windows MySQL `my.ini`:
   ```ini
   [mysqld]
   bind-address=0.0.0.0
   ```
3. 서비스 재시작 (관리자 CMD):
   ```cmd
   net stop mysql80
   net start mysql80
   ```
4. 유저 권한 확장:
   ```sql
   UPDATE mysql.user SET host='%' WHERE user='root';
   FLUSH PRIVILEGES;
   ```

> 공용 와이파이에서는 `0.0.0.0` + `host='%'` 조합이 위험. 방화벽으로 3306 차단 또는 host를 WSL IP로 제한.

---

## 부록 B. 비용 점검

| 항목 | 무료 한도 |
|------|-----------|
| EC2 t3.micro | 프리티어 750h/월 |
| RDS t4g.micro | 프리티어 750h/월 |
| EBS gp3 | 프리티어 30GB |
| Elastic IP | EC2 연결 시 무료 (해제 후 보유 시 유료) |
| Route 53 | 사용 안 하면 $0 (Hosted Zone 보유 시 $0.5/월) |
| API Gateway | 미사용 |
| Let's Encrypt | 영구 무료 |
| Vercel Hobby | 무료 |

> EC2 프리티어 만료(12개월) 후 ~$7-10/월. Lightsail $3.5 또는 Oracle Cloud Free Tier(ARM)로 이전 검토.

**RDS 주의:**
- 중지해도 7일 후 AWS가 자동 재시작 (정책)
- 중지 중에도 스토리지 비용은 발생
- 안 쓸 땐 EC2만 중지 가능 (Elastic IP는 인스턴스에 연결돼 있어야 무료)

---

## 부록 C. 자주 묻는 질문

**Q. A 레코드가 뭔가요?**
도메인 이름을 IP에 연결하는 DNS 레코드. `<API_SUBDOMAIN>` → `<EC2_ELASTIC_IP>`로 등록.

**Q. DNS는 도메인 산 곳에서 설정하나요?**
"네임서버가 어디냐"에 따라 다름. Vercel/Cloudflare로 위임돼 있으면 거기서, 등록기관 그대로면 등록기관에서. 확인:
```bash
nslookup -type=NS <DOMAIN> 8.8.8.8
```

**Q. 서브도메인을 따로 만드는 이유?**
프론트(`<DOMAIN>`, Vercel)와 API(`<API_SUBDOMAIN>`, EC2)를 분리해 운영하기 위함. 보안/배포/스케일링 측면에서 깨끗.

**Q. Vercel 쪽 HTTPS는 따로 해야 하나요?**
Vercel은 도메인 연결 시 Let's Encrypt를 자동 발급/갱신. EC2 쪽만 직접 발급.

**Q. Let's Encrypt 비용?**
완전 무료. 90일 만료지만 cron 자동 갱신.

**Q. dry-run이 뭐죠?**
실제 발급 없이 가능 여부만 검증. Let's Encrypt는 일 5회 발급 한도가 있어 설정 실수로 한도가 소진될 수 있음. dry-run은 카운트되지 않으므로 정식 발급 전 필수.

**Q. healthcheck unhealthy로 뜨는데 동작은 정상?**
컨테이너에 `curl`이 없거나 startup 시간이 healthcheck보다 길어서. 알려진 패턴이며 동작엔 영향 없음.

**Q. WSL2에서 Windows MySQL이 왜 안 보이나요?**
WSL2는 별도 네트워크 인터페이스를 가진 가상머신처럼 동작. `127.0.0.1`은 Ubuntu 자기 자신, Windows는 `cat /etc/resolv.conf` 의 nameserver IP로 접근.

**Q. EBS 늘렸는데 디스크 용량이 그대로?**
EBS 확장은 2단계. 콘솔에서 볼륨 크기를 늘려도 OS 안에서 `growpart` + `resize2fs`로 파티션과 파일시스템을 확장해야 인식.

**Q. CORS는 nginx와 Spring 중 어디서?**
**한 곳에서만**. 양쪽에서 추가하면 헤더 중복으로 브라우저가 거부. 이 매뉴얼은 nginx 단일 책임을 표준으로 한다.
