# 🚪 gateway 세팅 가이드

모든 마이크로서비스의 요청을 라우팅하고 CORS 설정을 관리하는 게이트웨이입니다.

## 1. 프레임워크 및 버전
- **Framework**: Spring Boot 3.4.4 / Spring Cloud Gateway
- **Language**: Java 17
- **Build Tool**: Gradle

## 2. 주요 설정 (Routing)
현재 게이트웨이는 다음과 같은 라우팅 규칙을 가지고 있습니다:
- `/auth/**` -> `http://localhost:9001` (auth-server)
- `/resume/**` -> `http://localhost:8081` (예정 혹은 다른 서버)

## 3. 설치 및 실행 방법

### IDE (IntelliJ) 사용 시
1. `gateway` 폴더를 프로젝트로 오픈합니다.
2. `src/main/java/.../GatewayApplication`을 찾아 실행합니다.

### CLI 사용 시
```bash
cd gateway
./gradlew bootRun
```

## 4. 참고 사항
- 실행 후 `http://localhost:8080`에서 서버가 동작합니다.
- 프론트엔드(`localhost:3000`)에서의 모든 요청은 이 8080 포트를 통해 전달되어야 합니다.
