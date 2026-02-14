# 🔐 auth-server 세팅 가이드

회원 인증 및 권한 관리를 담당하는 서비스입니다.

## 1. 프레임워크 및 버전
- **Framework**: Spring Boot 3.5.0
- **Language**: Java 17 / Kotlin
- **Build Tool**: Gradle

## 2. 사전 준비 사항
1. **MySQL 설치 및 실행**: 3306 포트 확인
2. **데이터베이스 생성**:
   ```sql
   CREATE DATABASE member;
   ```
3. **환경 변수(.env) 설정**: 프로젝트 루트 폴더에 `.env` 파일을 생성하고 보안 정보를 설정해야 합니다. (Git에 포함되지 않으므로 직접 생성 필요)
   - `.env` 예시:
     ```env
     SPRING_PROFILES_ACTIVE=local
     DB_URL=jdbc:mysql://localhost:3306/member?useSSL=false&serverTimezone=Asia/Seoul
     DB_USERNAME=root
     DB_PASSWORD=자신의_DB_비밀번호
     JWT_SECRET=자신의_JWT_비밀키
     ```

## 3. 주요 변경 및 관리 사항

### 🏢 프로파일 관리 (Profiles)
- **local**: 로컬 개발용 (`application-local.yml`). `.env`의 설정을 기반으로 동작합니다.
- **dev**: 개발 서버용 (`application-dev.yml`).
- 실행 시 환경 변수 `SPRING_PROFILES_ACTIVE`를 통해 변경 가능합니다.

### 🛡 보안 업데이트 (HTTP-Only Cookie)
- 기존의 토큰 반환 방식에서 **HTTP-Only 및 Secure 쿠키**를 사용하는 방식으로 변경되었습니다.
- 클라이언트(Frontend)에서는 별도의 토큰 저장 없이 쿠키를 통해 인증을 수행합니다.

## 4. 설치 및 실행 방법

### IDE (IntelliJ) 사용 시
1. `auth-server` 폴더를 프로젝트로 오픈합니다.
2. Gradle 싱크가 완료될 때까지 기다립니다.
3. `src/main/java/.../AuthServerApplication` (또는 해당 메인 클래스)를 찾아 실행합니다.

### CLI 사용 시
```bash
# 프로젝트 폴더 이동
cd auth-server

# 빌드 및 실행 (Windows)
./gradlew bootRun
```

## 4. 참고 사항
- 실행 후 `http://localhost:9001`에서 서버가 동작합니다.
- JWT 보안 설정이 포함되어 있으므로, `jwt.secret-key`를 확인하세요.
