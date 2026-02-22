# Auth Server — Domain Context

Spring Boot 3.5.0 + Java 17 기반 인증 서비스.

## 핵심 아키텍처: 헥사고날 (포트-어댑터)

```
auth-server/src/main/java/
└── {base-package}/
    ├── application/
    │   ├── port/
    │   │   ├── in/      ← 유스케이스 인터페이스
    │   │   └── out/     ← 리포지토리 인터페이스
    │   └── service/     ← 유스케이스 구현체
    ├── domain/
    │   ├── entity/      ← 도메인 엔티티
    │   └── vo/          ← 값 객체 (Email, HashedPassword, Provider, Role, Status)
    ├── infra/
    │   └── repository/  ← JPA 리포지토리 구현
    └── web/
        └── controller/  ← REST 컨트롤러
```

## 핵심 패턴

- **Value Objects (VOs)**: 도메인 기본 타입은 VO로 래핑 (Email, HashedPassword, Provider, Role, Status)
- **유스케이스 인터페이스**: `application.port.in`에 정의, `application.service`에서 구현
- **리포지토리 인터페이스**: `application.port.out`에 정의, `infra.repository`에서 구현
- **컨트롤러**: 얇게 유지 — 유스케이스 호출만, 비즈니스 로직 금지

## 기술 스택
- Spring Security + JWT (Auth0 java-jwt 4.5.0)
- MySQL with Spring Data JPA
- Spring Profile: local, dev
- JWT Secret: `JWT_SECRET` 환경변수

## API 엔드포인트
- `POST /auth/login` — 로그인, JWT를 HttpOnly 쿠키로 반환
- `POST /auth/sign-up` — 회원가입

## DB 연결
- `jdbc:mysql://localhost:3306/member`
- Profile별 설정: `application-{local/dev}.yml`

## 도메인 스킬
`/auth-domain` 스킬 사용 시 이 컨텍스트 자동 로드.
