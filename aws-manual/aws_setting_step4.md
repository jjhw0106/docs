# AWS 배포 매뉴얼 - 4단계: 배포 파일 작성

## 개요

EC2에서 Docker로 auth-server를 실행하기 위한 파일 3개를 작성한다.

---

## 핵심 개념

### Docker란?

"내 컴퓨터에서 되는데 서버에서 안 돼요" 문제를 없애는 도구.

앱 실행에 필요한 것들(Java, 설정, 코드)을 **컨테이너**라는 박스에 통째로 묶어서
어디서든 똑같이 실행되게 한다.

```
[로컬 PC]          [EC2 서버]
Docker 컨테이너 == Docker 컨테이너
완전히 동일한 환경
```

### 파일별 역할

| 파일 | 역할 | 비유 |
|------|------|------|
| `Dockerfile` | 이미지를 어떻게 만들지 정의 | 레시피 |
| `docker-compose.yml` | 컨테이너를 어떻게 실행할지 정의 | 주문서 |
| `.env` | 실제 비밀값 (git 미포함) | 금고 |
| `.env.example` | 어떤 변수가 필요한지 알려주는 템플릿 | 금고 목록 |
| `.dockerignore` | Docker 이미지 빌드 시 제외할 파일 목록 | Docker용 gitignore |

---

## 파일 설명

### 1. Dockerfile (기존 파일)

```dockerfile
# stage 1: 빌드 환경 (JDK 필요)
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew
RUN ./gradlew dependencies --no-daemon  # 의존성 먼저 받아서 캐싱
COPY src src
RUN ./gradlew bootJar --no-daemon       # jar 파일 빌드

# stage 2: 실행 환경 (JRE만 — JDK보다 훨씬 가벼움)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 9001
CMD ["java", "-jar", "app.jar"]
```

**멀티스테이지 빌드를 쓰는 이유:**
빌드 도구(JDK, Gradle)는 실행 시엔 필요 없다.
stage 1에서 빌드 후 jar만 stage 2로 복사 → 이미지 크기 대폭 감소.

---

### 2. docker-compose.yml

```yaml
services:
  auth-server:
    build: .              # 이 폴더의 Dockerfile로 이미지 빌드
    ports:
      - "9001:9001"       # EC2포트:컨테이너포트
    env_file:
      - .env              # 환경변수를 .env 파일에서 읽기
    restart: unless-stopped  # EC2 재시작 시 컨테이너도 자동 재시작
```

**ports 설명:**
`"9001:9001"` = EC2의 9001번 포트로 들어오면 컨테이너 9001번으로 연결.
왼쪽이 EC2(호스트), 오른쪽이 컨테이너.

**restart 옵션:**

| 값 | 동작 |
|----|------|
| `no` | 재시작 안 함 |
| `always` | 항상 재시작 (수동 중지해도 재시작) |
| `unless-stopped` | 수동으로 중지하지 않는 한 항상 재시작 ← 권장 |
| `on-failure` | 오류로 종료됐을 때만 재시작 |

---

### 3. .env.example

```
SPRING_PROFILES_ACTIVE=prod

DB_URL=jdbc:mysql://<RDS_ENDPOINT>:3306/authdb?useSSL=false&serverTimezone=Asia/Seoul
DB_USERNAME=authuser
DB_PASSWORD=your_db_password

JWT_SECRET=your_jwt_secret_key
```

- git에 올라가는 파일 (비밀값 없음)
- EC2에서 `.env` 파일 만들 때 이걸 복사해서 실제 값 채워 넣음

---

### 4. .dockerignore

```
.gradle
build
.env
*.md
.gitignore
```

`docker build` 실행 시 현재 폴더를 Docker 엔진에 통째로 전송하는데,
불필요한 파일 제외 → 빌드 속도 향상 + `.env` 비밀값 이미지 포함 방지.

| 파일 | 제외 이유 |
|------|----------|
| `.gradle` | Gradle 캐시, 빌드에 불필요 |
| `build` | 로컬 빌드 결과물, Docker가 직접 빌드함 |
| `.env` | 비밀값이 Docker 이미지에 포함되는 것 방지 |

---

## .env vs .gitignore vs .dockerignore 관계

```
.env                ← 실제 비밀값 파일
 ├── .gitignore     ← git 커밋에서 제외 (GitHub에 안 올라감)
 └── .dockerignore  ← Docker 이미지 빌드에서 제외 (이미지에 안 포함됨)
```

**.env를 실수로 git 커밋했다면:**
1. `git rm --cached .env` 로 추적 제거
2. GitHub에 이미 올라갔다면 → **비밀값 자체를 변경** (노출된 것으로 간주)

---

## Spring 프로파일 분리 원리

### 파일 구조

```
application.yml          ← 항상 로드 (공통 설정)
application-local.yml    ← SPRING_PROFILES_ACTIVE=local 일 때 추가 로드
application-dev.yml      ← SPRING_PROFILES_ACTIVE=dev 일 때 추가 로드
application-prod.yml     ← SPRING_PROFILES_ACTIVE=prod 일 때 추가 로드
```

Spring Boot가 시작 시 `application-{프로파일명}.yml` 규칙으로 자동으로 파일을 찾는다.
개발자가 따로 연결 코드 작성 불필요.

### 로드 순서

```
application.yml (공통)
      +
application-prod.yml (환경별)
      ↓
겹치는 키는 환경별 파일이 덮어씀
```

### 프로파일 활성화 방법 (우선순위 낮 → 높)

```
1. application.yml에 명시
   spring:
     profiles:
       active: local

2. JVM 옵션으로 전달
   java -jar -Dspring.profiles.active=prod app.jar

3. 환경변수로 전달  ← Docker에서 주로 이 방법 사용
   SPRING_PROFILES_ACTIVE=prod

4. IntelliJ Run Configuration
   Active profiles 입력란에 local 입력
```

### 환경별 구성

| 환경 | 프로파일 | 활성화 방법 | DB |
|------|---------|------------|-----|
| 로컬 개발 | `local` | IntelliJ Run Configuration | 로컬 MySQL |
| EC2 운영 | `prod` | `.env` → docker-compose 주입 | RDS MySQL |

---

## 완료 체크리스트

- [x] `docker-compose.yml` 작성
- [x] `.env.example` 작성
- [x] `.dockerignore` 작성
- [x] `.gitignore`에 `.env` 포함 확인
- [ ] GitHub에 push
