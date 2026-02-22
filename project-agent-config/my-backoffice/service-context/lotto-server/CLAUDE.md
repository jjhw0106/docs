# Lotto Server — Domain Context

Spring Boot 3.5.0 + Java 17 기반 로또 관리 서비스.

## 핵심 아키텍처: 헥사고날 (auth-server와 동일 패턴)

```
lotto-server/src/main/java/
└── {base-package}/
    ├── application/
    │   ├── port/
    │   │   ├── in/      ← 유스케이스 인터페이스
    │   │   └── out/     ← 리포지토리 인터페이스
    │   └── service/     ← 유스케이스 구현체
    ├── domain/
    │   ├── entity/      ← WinningNumbers, PurchaseGame
    │   └── vo/          ← LottoNumber(1-45), Round, Rank
    ├── infra/           ← JPA 구현, WebClient
    └── web/             ← 컨트롤러, DTO
```

## 도메인 핵심 개념

### Value Objects
- `LottoNumber`: 1-45 사이 값 검증
- `Round`: 회차 번호
- `Rank`: FIRST, SECOND, THIRD, FOURTH, FIFTH, NONE

### 순수 도메인 로직
- `RankCalculator`: 당첨 번호와 구매 번호 비교 → Rank 반환

### 배치 잡
- `WinningNumberFetchJob`: 매주 토요일 21:00 (동행복권 API 조회)
- `DataCleanupJob`: 매일 03:00 (정리 작업)

## 외부 API
- 동행복권 dhlottery.co.kr
- 캐시 우선: DB → 없으면 외부 API → DB 저장

## 주요 제약
- 현재 `DEFAULT_MEMBER_ID = 1L` 사용 (멀티유저 미지원)
- JWT 인증 필수 (auth-server와 시크릿 공유)

## API 엔드포인트 (via Gateway /lotto/**)
- `GET /lotto/winning-numbers?round={round}` — 당첨 번호 (공개)
- `POST /lotto/purchases` — 구매 등록 (JWT 필요)
- `GET /lotto/purchases?page&size&result&roundFrom&roundTo` — 구매 이력 (JWT 필요)
- `GET /lotto/purchases/summary` — 통계 (JWT 필요)
- `DELETE /lotto/purchases/{gameId}` — 삭제 (JWT 필요)

## 도메인 스킬
`/lotto-domain` 스킬 사용 시 이 컨텍스트 자동 로드.
