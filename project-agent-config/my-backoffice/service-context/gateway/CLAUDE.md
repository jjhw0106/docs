# Gateway — Domain Context

Spring Cloud Gateway + WebFlux (Reactive) 기반 API 게이트웨이.

## 핵심 원칙

**게이트웨이는 라우팅만 담당. 비즈니스 로직 금지.**

## 현재 라우팅 구성

| 경로 패턴 | 대상 서비스 | 포트 |
|-----------|-------------|------|
| `/auth/**` | auth-server | 9001 |
| `/resume/**` | resume-service | 8081 |
| `/lotto/**` | lotto-server | 8082 |

## 기술 스택
- Spring Boot 3.4.4 + Java 17
- Spring Cloud 2024.0.1
- WebFlux (리액티브) — 블로킹 코드 금지
- CORS: localhost:3000 허용

## 라우팅 추가 패턴 (application.yml)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: new-service
          uri: http://localhost:{port}
          predicates:
            - Path=/{service}/**
          filters:
            - StripPrefix=0
```

## CORS 설정
- Allowed Origins: `http://localhost:3000`
- Allowed Methods: GET, POST, PUT, DELETE, OPTIONS
- Allow Credentials: true

## 도메인 스킬
`/gateway-domain` 스킬 사용 시 이 컨텍스트 자동 로드.
