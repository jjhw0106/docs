# Gateway Domain Skill

gateway 작업 시 라우팅 원칙을 강제하는 스킬.

## 라우팅 추가 체크리스트

1. `application.yml`의 `spring.cloud.gateway.routes`에 라우트 추가
2. `id`: 고유한 라우트 식별자
3. `uri`: 대상 서비스 주소 (환경별로 분리 필요 시 profile 사용)
4. `predicates`: Path 패턴
5. CORS 설정에 새 origin이 필요하면 추가

## 금지 패턴
- 게이트웨이에 비즈니스 로직 작성 (❌)
- 블로킹 코드 사용 (WebFlux 환경) (❌)
- 게이트웨이에서 DB 직접 접근 (❌)

## 새 서비스 연결 시 확인 사항
- 대상 서비스 포트 확인
- 경로 충돌 여부 확인
- StripPrefix 필요 여부 결정
- 인증 필터 적용 여부 결정
