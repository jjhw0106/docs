# Lotto Domain Skill

lotto-server 작업 시 헥사고날 아키텍처와 도메인 패턴을 강제하는 스킬.

## 새 기능 추가 체크리스트

1. **도메인 레이어** (필요 시)
   - 새 VO: `domain/vo/`에 추가, 값 검증 포함
   - 새 엔티티: `domain/entity/`
   - 순수 로직: `RankCalculator` 패턴 참조

2. **애플리케이션 레이어**
   - 유스케이스 인터페이스: `application/port/in/`
   - 리포지토리 인터페이스: `application/port/out/`
   - 구현: `application/service/`

3. **인프라 레이어**
   - JPA 구현: `infra/repository/`
   - 외부 API 통신: WebClient 사용 (블로킹 클라이언트 금지)

4. **배치 잡 수정 시**
   - Cron 표현식 확인: `WinningNumberFetchJob` (토 21:00), `DataCleanupJob` (매일 03:00)
   - @Scheduled 어노테이션 사용

5. **JWT 검증**
   - 보안 엔드포인트에 JWT 필터 적용 확인
   - auth-server와 동일한 JWT_SECRET 사용

## 금지 패턴
- 컨트롤러에 랭크 계산 로직 직접 작성 (❌ — RankCalculator 사용)
- WebFlux 없이 RestTemplate 사용 (❌ — WebClient 사용)
- 하드코딩된 멤버 ID (DEFAULT_MEMBER_ID 상수 사용)
