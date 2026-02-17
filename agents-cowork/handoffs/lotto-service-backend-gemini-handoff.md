# Lotto Service - Backend Gemini Handoff Document

> 작성일: 2026-02-17 | 대상: Gemini Backend Developer | 최종 수정: 2026-02-17

---

## 1. 서비스 개요

### 1.1 목적
사용자가 자신의 로또 구매 내역을 기록하고, 최신 당첨 번호를 조회하여 당첨 여부를 자동으로 확인할 수 있는 개인 로또 관리 서비스.

### 1.2 기술 스택
| 항목 | 내용 |
|---|---|
| 프레임워크 | Spring Boot 3.5.0 |
| 자바 버전 | Java 17 |
| 서버 포트 | 8082 |
| 데이터베이스 | MySQL (auth-server와 동일) |
| 아키텍처 | 헥사고날 아키텍처 (포트-어댑터) |
| 빌드 도구 | Gradle |
| API 통신 | REST (JSON) |

### 1.3 서비스 위치
- **API Gateway 라우팅**: `http://localhost:8080/lotto/**` → `http://localhost:8082`
- **직접 접근**: `http://localhost:8082`

---

## 2. 프로젝트 구조 (헥사고날 아키텍처)

```
com.my_back_office.lotto_server/
├── application/                          # 애플리케이션 계층 (유스케이스)
│   ├── port/
│   │   ├── in/                          # 입력 포트 (유스케이스 인터페이스)
│   │   │   ├── FetchWinningNumbersUseCaseITF.java
│   │   │   ├── RegisterPurchaseUseCaseITF.java
│   │   │   ├── FetchPurchaseHistoryUseCaseITF.java
│   │   │   ├── FetchPurchaseSummaryUseCaseITF.java
│   │   │   └── DeletePurchaseUseCaseITF.java
│   │   └── out/                         # 출력 포트 (리포지토리 인터페이스)
│   │       ├── WinningNumbersRepositoryITF.java
│   │       ├── PurchaseGameRepositoryITF.java
│   │       └── ExternalApiClientITF.java
│   └── service/                          # 유스케이스 구현 서비스
│       ├── LottoWinningService.java     # 당첨 번호 조회/수집
│       ├── PurchaseService.java         # 구매 등록/조회/삭제
│       ├── RankCalculationService.java  # 당첨 등수 계산
│       └── DataCleanupService.java      # 배치 데이터 삭제
├── domain/                               # 도메인 계층 (비즈니스 규칙)
│   ├── lotto/
│   │   ├── WinningNumbers.java          # 당첨 번호 도메인 엔티티
│   │   ├── PurchaseGame.java            # 구매 게임 도메인 엔티티
│   │   └── vo/
│   │       ├── LottoNumber.java         # 로또 번호 값 객체 (1~45)
│   │       ├── Round.java               # 회차 값 객체
│   │       └── Rank.java                # 등수 열거형
│   └── RankCalculator.java              # 순수 도메인 로직 (등수 판정)
├── infra/                                # 인프라 계층 (외부 기술)
│   ├── jpa/
│   │   ├── WinningNumbersJpaRepository.java
│   │   └── PurchaseGameJpaRepository.java
│   ├── entity/
│   │   ├── WinningNumbersEntity.java
│   │   └── PurchaseGameEntity.java
│   ├── repository/
│   │   ├── WinningNumbersRepository.java     # 리포지토리 ITF 구현
│   │   └── PurchaseGameRepository.java       # 리포지토리 ITF 구현
│   ├── external/
│   │   └── DhlotteryApiClient.java          # 동행복권 API 클라이언트
│   └── batch/
│       ├── WinningNumberFetchJob.java       # 매주 토요일 21:00 배치
│       └── DataCleanupJob.java              # 매일 03:00 배치
├── web/                                  # 웹 어댑터 (HTTP 컨트롤러)
│   ├── LottoController.java
│   └── dto/
│       ├── WinningNumbersResponseDTO.java
│       ├── PurchaseRequestDTO.java
│       ├── PurchaseResponseDTO.java
│       ├── PurchaseGameDetailDTO.java
│       ├── PurchaseHistoryResponseDTO.java
│       └── PurchaseSummaryResponseDTO.java
├── global/
│   ├── config/
│   │   ├── SecurityConfig.java          # JWT 인증/인가 설정
│   │   ├── SchedulingConfig.java        # 배치 스케줄 설정
│   │   └── RestTemplateConfig.java      # HTTP 클라이언트 설정
│   ├── exception/
│   │   ├── LottoException.java
│   │   ├── LottoExceptionHandler.java
│   │   └── ErrorCode.java               # 에러 코드 정의
│   └── util/
│       └── RoundCalculator.java         # 회차 계산 유틸리티
└── LottoServerApplication.java          # Spring Boot 메인 클래스
```

---

## 3. 데이터베이스 스키마

### 3.1 당첨 번호 테이블 (winning_numbers)

```sql
CREATE TABLE winning_numbers (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  round INT NOT NULL UNIQUE COMMENT '회차 번호',
  draw_date DATE NOT NULL COMMENT '추첨일자',
  number1 TINYINT NOT NULL COMMENT '당첨 번호 1번',
  number2 TINYINT NOT NULL COMMENT '당첨 번호 2번',
  number3 TINYINT NOT NULL COMMENT '당첨 번호 3번',
  number4 TINYINT NOT NULL COMMENT '당첨 번호 4번',
  number5 TINYINT NOT NULL COMMENT '당첨 번호 5번',
  number6 TINYINT NOT NULL COMMENT '당첨 번호 6번',
  bonus_number TINYINT NOT NULL COMMENT '보너스 번호',
  first_prize_amount BIGINT COMMENT '1등 1인당 당첨금 (원)',
  first_prize_winner_count INT COMMENT '1등 당첨자 수',
  total_sales_amount BIGINT COMMENT '총 판매액 (원)',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '레코드 생성 시각',
  INDEX idx_round (round),
  INDEX idx_draw_date (draw_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.2 구매 게임 테이블 (purchase_games)

```sql
CREATE TABLE purchase_games (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT NOT NULL COMMENT '사용자 ID (auth-server member)',
  round INT NOT NULL COMMENT '로또 회차 번호',
  number1 TINYINT NOT NULL COMMENT '구매 번호 1번',
  number2 TINYINT NOT NULL COMMENT '구매 번호 2번',
  number3 TINYINT NOT NULL COMMENT '구매 번호 3번',
  number4 TINYINT NOT NULL COMMENT '구매 번호 4번',
  number5 TINYINT NOT NULL COMMENT '구매 번호 5번',
  number6 TINYINT NOT NULL COMMENT '구매 번호 6번',
  rank VARCHAR(20) COMMENT '당첨 등수 (FIRST, SECOND, THIRD, FOURTH, FIFTH, NONE, NULL)',
  prize_amount INT DEFAULT 0 COMMENT '당첨금 (원)',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '등록 시각',
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '최종 수정 시각',
  INDEX idx_member_round (member_id, round) COMMENT '회차별 사용자 게임 조회',
  INDEX idx_round (round) COMMENT '배치 일괄 업데이트',
  INDEX idx_created_at (created_at) COMMENT '배치 삭제 대상 조회',
  FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 4. 외부 API 연동

### 4.1 동행복권 API 명세

**엔드포인트:**
```
GET https://www.dhlottery.co.kr/common.do?method=getLottoNumber&drwNo={drawNumber}
```

**요청 파라미터:**
| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| drawNumber | INT | Y | 회차 번호 (예: 1155) |

**응답 예시:**
```json
{
  "returnValue": "success",
  "drwNo": 1155,
  "drwNoDate": "2025-01-25",
  "drwtNo1": 1,
  "drwtNo2": 9,
  "drwtNo3": 12,
  "drwtNo4": 13,
  "drwtNo5": 20,
  "drwtNo6": 45,
  "bnusNo": 3,
  "firstWinamnt": 1396028764,
  "firstPrzwnerCo": 12,
  "totSellamnt": 111840714000
}
```

### 4.2 응답 필드 매핑

| API 필드 | 내부 필드 | 타입 | 설명 |
|---|---|---|---|
| returnValue | - | STRING | "success" 또는 "fail" |
| drwNo | round | INT | 회차 번호 |
| drwNoDate | drawDate | DATE | 추첨일자 (YYYY-MM-DD 형식) |
| drwtNo1~6 | number1~6 | TINYINT | 당첨 번호 (오름차순 정렬 저장) |
| bnusNo | bonusNumber | TINYINT | 보너스 번호 |
| firstWinamnt | firstPrizeAmount | BIGINT | 1등 1인당 당첨금 (원) |
| firstPrzwnerCo | firstPrizeWinnerCount | INT | 1등 당첨자 수 |
| totSellamnt | totalSalesAmount | BIGINT | 총 판매액 (원) |

### 4.3 API 클라이언트 구현 가이드

```java
// DhlotteryApiClient.java
@Component
public class DhlotteryApiClient implements ExternalApiClientITF {

  private static final String DHLOTTERY_BASE_URL = "https://www.dhlottery.co.kr";
  private static final String METHOD_NAME = "getLottoNumber";

  @Autowired
  private RestTemplate restTemplate;

  /**
   * 동행복권 API에서 당첨 번호 조회
   * @param drawNumber 회차 번호
   * @return DhlotteryResponse (원본 응답)
   * @throws LottoException API 호출 실패 시
   */
  public DhlotteryResponse fetchWinningNumbers(int drawNumber) {
    String url = String.format("%s/common.do?method=%s&drwNo=%d",
      DHLOTTERY_BASE_URL, METHOD_NAME, drawNumber);

    try {
      // Rate limiting: 1초 간격 유지
      Thread.sleep(1000);

      ResponseEntity<DhlotteryResponse> response = restTemplate.getForEntity(
        url,
        DhlotteryResponse.class
      );

      if (response.getStatusCode() != HttpStatus.OK) {
        throw new LottoException(ErrorCode.EXTERNAL_API_FAILURE);
      }

      DhlotteryResponse body = response.getBody();
      if (body == null || !"success".equals(body.getReturnValue())) {
        throw new LottoException(ErrorCode.DRAWING_NOT_FOUND);
      }

      return body;

    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new LottoException(ErrorCode.EXTERNAL_API_FAILURE);
    } catch (Exception e) {
      throw new LottoException(ErrorCode.EXTERNAL_API_FAILURE);
    }
  }
}
```

---

## 5. REST API 명세

### 5.1 GET /lotto/winning-numbers - 당첨 번호 조회

**설명:** 동행복권 API를 호출하여 해당 회차(또는 최신 회차) 당첨 번호를 반환합니다.

**요청:**
```
GET /lotto/winning-numbers?round=1155
```

**요청 파라미터:**
| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| round | INT | N | null | 회차 번호. 미입력 시 현재 판매 중인 회차 |

**응답 (200 OK):**
```json
{
  "round": 1155,
  "drawDate": "2025-01-25",
  "numbers": [1, 9, 12, 13, 20, 45],
  "bonusNumber": 3,
  "firstPrizeAmount": 1396028764,
  "firstPrizeWinnerCount": 12,
  "totalSalesAmount": 111840714000
}
```

**에러 응답 (502):**
```json
{
  "code": "LOTTO_001",
  "message": "당첨 번호를 불러올 수 없습니다. 잠시 후 다시 시도해주세요.",
  "timestamp": "2026-02-17T10:30:00"
}
```

**구현 로직:**
1. round 파라미터 미입력 시 현재 회차 계산 (2002-12-07 기준, 매주 토요일 추첨)
2. DB 캐시 우선 조회 (winning_numbers 테이블)
3. 캐시 미스 시 동행복권 API 호출
4. API 응답 파싱 후 내부 DTO로 변환하여 반환
5. 실패 시 예외 발생

---

### 5.2 POST /lotto/purchases - 구매 번호 등록

**설명:** 사용자의 로또 구매 번호를 등록합니다. 해당 회차 추첨이 완료되었으면 즉시 당첨 결과를 계산합니다.

**인증:** 필수 (JWT 토큰 - Authorization 헤더)

**요청:**
```
POST /lotto/purchases
Content-Type: application/json
Authorization: Bearer {accessToken}

{
  "round": 1156,
  "games": [
    { "numbers": [3, 12, 17, 28, 33, 41] },
    { "numbers": [5, 11, 22, 30, 38, 44] }
  ]
}
```

**요청 DTO 상세:**
```typescript
// PurchaseRequestDTO
{
  round: number;           // 회차 번호 (1 이상, 현재 회차 이하)
  games: Array<{
    numbers: number[];     // 6개 번호 배열 (1~45, 오름차순 정렬됨)
  }>;
}
```

**응답 (201 Created):**
```json
{
  "purchaseId": 42,
  "round": 1156,
  "games": [
    {
      "gameId": 101,
      "numbers": [3, 12, 17, 28, 33, 41],
      "rank": null,
      "prizeAmount": 0
    },
    {
      "gameId": 102,
      "numbers": [5, 11, 22, 30, 38, 44],
      "rank": null,
      "prizeAmount": 0
    }
  ],
  "createdAt": "2026-02-15T14:30:00"
}
```

**검증 규칙:**

| 규칙 | 조건 | 에러 코드 | 상태 |
|---|---|---|---|
| 회차 유효성 | 1 이상 정수, 현재 회차 이하 | LOTTO_003 | 400 |
| 게임 수 제한 | 1~5개 | LOTTO_004 | 400 |
| 번호 개수 | 각 게임 정확히 6개 | LOTTO_004 | 400 |
| 번호 범위 | 각 번호 1~45 | LOTTO_004 | 400 |
| 번호 중복 | 같은 게임 내 중복 불가 | LOTTO_004 | 400 |
| 회차당 최대 게임 | 누적 5게임 초과 불가 | LOTTO_005 | 400 |

**처리 로직:**
1. JWT에서 사용자 ID (memberId) 추출
2. 입력값 유효성 검증 (위의 규칙들)
3. 해당 회차에 기존 등록 게임 수 확인 (누적 5게임 제한)
4. purchase_games 테이블에 저장
5. 해당 회차 추첨이 완료되었으면:
   - winning_numbers 테이블에서 당첨 번호 조회
   - RankCalculationService로 등수 계산
   - rank, prize_amount 업데이트
6. 저장된 결과 반환

---

### 5.3 GET /lotto/purchases - 구매 이력 조회

**설명:** 사용자의 로또 구매 이력을 페이지네이션으로 조회합니다.

**인증:** 필수 (JWT 토큰)

**요청:**
```
GET /lotto/purchases?page=0&size=20&result=ALL&roundFrom=1150&roundTo=1156
```

**요청 파라미터:**
| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| page | INT | N | 0 | 페이지 번호 (0부터 시작) |
| size | INT | N | 20 | 페이지 크기 |
| result | STRING | N | ALL | 필터: ALL(전체), WON(당첨만), LOST(미당첨만) |
| roundFrom | INT | N | null | 회차 범위 시작 |
| roundTo | INT | N | null | 회차 범위 종료 |

**응답 (200 OK):**
```json
{
  "content": [
    {
      "gameId": 101,
      "round": 1155,
      "drawDate": "2025-01-25",
      "numbers": [5, 11, 22, 30, 38, 44],
      "winningNumbers": [1, 9, 12, 22, 38, 45],
      "bonusNumber": 3,
      "matchedNumbers": [22, 38],
      "matchedBonusNumber": false,
      "rank": "FIFTH",
      "prizeAmount": 5000,
      "createdAt": "2025-01-24T18:00:00"
    }
  ],
  "totalElements": 48,
  "totalPages": 3,
  "currentPage": 0,
  "pageSize": 20
}
```

**필터 로직:**
- `result=WON`: rank가 FIRST, SECOND, THIRD, FOURTH, FIFTH인 게임만
- `result=LOST`: rank가 NONE 또는 NULL인 게임
- `roundFrom/roundTo`: 회차 범위 필터 (포함 조건)

---

### 5.4 GET /lotto/purchases/summary - 통계 요약 조회

**설명:** 사용자의 구매/당첨 통계를 요약으로 반환합니다.

**인증:** 필수 (JWT 토큰)

**요청:**
```
GET /lotto/purchases/summary
```

**응답 (200 OK):**
```json
{
  "totalGames": 24,
  "totalWins": 3,
  "totalInvestment": 120000,
  "totalPrize": 15000,
  "winRate": 12.5,
  "rankDistribution": {
    "FIRST": 0,
    "SECOND": 0,
    "THIRD": 0,
    "FOURTH": 1,
    "FIFTH": 2,
    "NONE": 21
  }
}
```

**계산 로직:**
- `totalGames`: 사용자의 모든 구매 게임 수
- `totalWins`: rank가 FIRST~FIFTH인 게임 수
- `totalInvestment`: 총 구매 게임 수 × 2,500원 (1게임 가격)
- `totalPrize`: 모든 prizeAmount의 합
- `winRate`: (totalWins / totalGames) × 100 (소수점 1자리)
- `rankDistribution`: 등수별 게임 수 집계

---

### 5.5 DELETE /lotto/purchases/{gameId} - 구매 내역 삭제

**설명:** 특정 구매 게임을 삭제합니다.

**인증:** 필수 (JWT 토큰)

**요청:**
```
DELETE /lotto/purchases/101
Authorization: Bearer {accessToken}
```

**응답 (204 No Content)**

**에러 케이스:**
| 상황 | 에러 코드 | 상태 | 메시지 |
|---|---|---|---|
| 게임이 다른 사용자 소유 | LOTTO_007 | 403 | "삭제 권한이 없습니다." |
| 당첨 게임 삭제 시도 | LOTTO_008 | 400 | "당첨된 게임은 삭제할 수 없습니다." |
| 게임을 찾을 수 없음 | LOTTO_002 | 404 | "해당 게임을 찾을 수 없습니다." |

**처리 로직:**
1. JWT에서 사용자 ID 추출
2. gameId로 게임 조회
3. 권한 검증 (memberId 일치 확인)
4. 게임의 rank 확인:
   - rank가 FIRST~FIFTH이면 삭제 불가 (에러)
   - rank가 NONE 또는 NULL이면 삭제 진행
5. 데이터베이스에서 삭제

---

## 6. 당첨 등수 판정 로직

### 6.1 등수 정의 및 당첨금

| 등수 | 조건 | 당첨금 |
|---|---|---|
| 1등 | 6개 번호 모두 일치 | 가변 (총 당첨금의 약 75% / 당첨자 수) |
| 2등 | 5개 번호 일치 + 보너스 번호 일치 | 가변 (총 당첨금의 약 12.5% / 당첨자 수) |
| 3등 | 5개 번호 일치 (보너스 미일치) | 가변 (총 당첨금의 약 12.5% / 당첨자 수) |
| 4등 | 4개 번호 일치 | 고정 50,000원 |
| 5등 | 3개 번호 일치 | 고정 5,000원 |
| 미당첨 (NONE) | 2개 이하 일치 | 0원 |

### 6.2 판정 알고리즘

```java
// RankCalculator.java - 순수 도메인 로직

public class RankCalculator {

  /**
   * 사용자의 번호와 당첨 번호를 비교하여 등수를 판정
   * @param myNumbers 사용자 번호 6개 (정렬됨)
   * @param winningNumbers 당첨 번호 6개 (정렬됨)
   * @param bonusNumber 보너스 번호 1개
   * @return Rank 등수 (FIRST ~ NONE)
   */
  public static Rank calculateRank(
      List<Integer> myNumbers,
      List<Integer> winningNumbers,
      Integer bonusNumber) {

    // 일치하는 번호 개수 계산
    int matchCount = (int) myNumbers.stream()
      .filter(winningNumbers::contains)
      .count();

    // 보너스 번호 일치 여부
    boolean hasBonusMatch = myNumbers.contains(bonusNumber);

    // 등수 판정
    if (matchCount == 6) {
      return Rank.FIRST;
    }
    if (matchCount == 5 && hasBonusMatch) {
      return Rank.SECOND;
    }
    if (matchCount == 5) {
      return Rank.THIRD;
    }
    if (matchCount == 4) {
      return Rank.FOURTH;
    }
    if (matchCount == 3) {
      return Rank.FIFTH;
    }

    return Rank.NONE;
  }

  /**
   * 당첨 등수별 고정 당첨금 반환
   * 1등, 2등, 3등은 동행복권 API 응답 데이터 사용
   * @param rank 등수
   * @param firstPrizeAmount 1등 당첨금 (API 응답)
   * @param firstPrizeWinnerCount 1등 당첨자 수 (API 응답)
   * @param totalSalesAmount 총 판매액 (API 응답)
   * @return 당첨금 (원)
   */
  public static long calculatePrizeAmount(
      Rank rank,
      Long firstPrizeAmount,
      Integer firstPrizeWinnerCount,
      Long totalSalesAmount) {

    switch (rank) {
      case FOURTH:
        return 50000L;
      case FIFTH:
        return 5000L;
      default:
        // 1등, 2등, 3등은 API 응답값 활용하여 계산
        // 현재 단순화: 고정값 사용
        // TODO: 정확한 당첨금 계산 로직 추가
        return 0L;
    }
  }
}
```

### 6.3 도메인 엔티티 예시

```java
// PurchaseGame.java - 도메인 엔티티

public class PurchaseGame {

  private Long gameId;
  private Long memberId;
  private Round round;
  private List<LottoNumber> numbers;  // 6개
  private Rank rank;
  private Long prizeAmount;

  /**
   * 당첨 결과 계산 (추첨 완료 후 호출)
   */
  public void calculateAndSetRank(
      WinningNumbers winningNumbers,
      RankCalculator calculator) {

    Rank calculatedRank = calculator.calculateRank(
      this.numbers.stream().map(LottoNumber::getValue).collect(Collectors.toList()),
      winningNumbers.getNumbers(),
      winningNumbers.getBonusNumber()
    );

    this.rank = calculatedRank;

    if (calculatedRank != Rank.NONE) {
      this.prizeAmount = calculator.calculatePrizeAmount(
        calculatedRank,
        winningNumbers.getFirstPrizeAmount(),
        winningNumbers.getFirstPrizeWinnerCount(),
        winningNumbers.getTotalSalesAmount()
      );
    }
  }
}
```

---

## 7. 배치 처리 명세

### 7.1 당첨 번호 자동 수집 배치

**스케줄:** 매주 토요일 21:00 (추첨 직후)

**목적:** 최신 회차 당첨 번호를 동행복권 API에서 수집하여 DB에 저장하고, 모든 사용자의 구매 내역에 대해 당첨 결과를 자동 계산합니다.

**구현:**
```java
// WinningNumberFetchJob.java

@Component
public class WinningNumberFetchJob {

  @Autowired
  private LottoWinningService winningService;

  @Autowired
  private PurchaseService purchaseService;

  @Scheduled(cron = "0 0 21 * * SAT")  // 매주 토요일 21:00
  public void fetchAndCalculateWinningNumbers() {

    try {
      // 1. 현재 회차 계산
      int currentRound = RoundCalculator.getCurrentRound();

      // 2. 동행복권 API 호출 (재시도 로직 포함)
      WinningNumbers winningNumbers = winningService.fetchWinningNumbersWithRetry(
        currentRound,
        3,  // 최대 3회 재시도
        30  // 30분 간격
      );

      // 3. DB 저장
      winningService.saveWinningNumbers(winningNumbers);

      // 4. 해당 회차의 모든 사용자 구매 내역 조회
      List<PurchaseGame> allGames = purchaseService.findGamesByRound(currentRound);

      // 5. 각 게임별 당첨 등수 계산
      for (PurchaseGame game : allGames) {
        game.calculateAndSetRank(winningNumbers, new RankCalculator());
      }

      // 6. 업데이트 저장
      purchaseService.updateAll(allGames);

      // 7. 완료 로그 기록
      logger.info("Winning number fetch completed. Round: {}, Total updated games: {}",
        currentRound, allGames.size());

    } catch (Exception e) {
      logger.error("Winning number fetch job failed", e);
      // 에러 알림 로직 추가
    }
  }
}
```

**재시도 로직:**
- 초기 실패 시 30분 간격으로 최대 3회 재시도
- 3회 모두 실패 시 에러 로그 기록 및 관리자 알림

---

### 7.2 오래된 데이터 자동 삭제 배치

**스케줄:** 매일 03:00

**목적:** 보관 기간이 지난 미당첨 구매 내역을 자동으로 삭제합니다. 당첨 내역은 영구 보관합니다.

**구현:**
```java
// DataCleanupJob.java

@Component
public class DataCleanupJob {

  @Autowired
  private DataCleanupService cleanupService;

  @Value("${lotto.data.retention-days:180}")  // 기본 180일 (6개월)
  private Integer retentionDays;

  @Scheduled(cron = "0 0 3 * * *")  // 매일 03:00
  public void deleteOldPurchaseData() {

    try {
      // 1. 기준일 계산: 현재 - 6개월
      LocalDateTime cutoffDate = LocalDateTime.now()
        .minusDays(retentionDays);

      // 2. 삭제 대상 조회
      // 조건:
      // - created_at < cutoffDate
      // - (rank IS NULL OR rank = 'NONE' OR rank = 'FIFTH')
      List<PurchaseGame> targetGames = purchaseRepository.findOldUnwonGames(cutoffDate);

      if (targetGames.isEmpty()) {
        logger.info("No old data to delete");
        return;
      }

      // 3. 대상 건수 로그 기록
      logger.info("Starting deletion of {} old game records", targetGames.size());

      // 4. 1000건 단위 배치 삭제 (DB 부하 방지)
      int batchSize = 1000;
      int totalDeleted = 0;

      for (int i = 0; i < targetGames.size(); i += batchSize) {
        int end = Math.min(i + batchSize, targetGames.size());
        List<PurchaseGame> batch = targetGames.subList(i, end);
        List<Long> gameIds = batch.stream()
          .map(PurchaseGame::getId)
          .collect(Collectors.toList());

        purchaseRepository.deleteAllByIdInBatch(gameIds);
        totalDeleted += batch.size();

        logger.debug("Deleted {} games (progress: {}/{})",
          batch.size(), totalDeleted, targetGames.size());
      }

      // 5. 완료 로그 기록
      Optional<PurchaseGame> first = targetGames.stream().findFirst();
      Optional<PurchaseGame> last = targetGames.stream().reduce((a, b) -> b);

      if (first.isPresent() && last.isPresent()) {
        int roundFrom = first.get().getRound();
        int roundTo = last.get().getRound();
        logger.info("Deletion completed. {} records deleted. Round range: {} ~ {}",
          totalDeleted, roundFrom, roundTo);
      }

    } catch (Exception e) {
      logger.error("Data cleanup job failed", e);
      // 에러 알림 로직 추가
    }
  }
}
```

**삭제 제외 조건:**
- rank = FIRST, SECOND, THIRD, FOURTH (4등 이상 당첨 내역)
- 모든 당첨 내역은 보관 기간 제약 없이 영구 보관

**트랜잭션:**
- 1000건 단위 배치 삭제로 DB 부하 최소화
- 트랜잭션 타임아웃 설정 (예: 30초)

---

## 8. 에러 처리 및 예외 케이스

### 8.1 에러 응답 규격

```json
{
  "code": "LOTTO_001",
  "message": "사용자에게 보여줄 친화적인 메시지",
  "timestamp": "2026-02-17T10:30:00"
}
```

### 8.2 에러 코드 및 처리

| 코드 | 상황 | HTTP Status | 메시지 |
|---|---|---|---|
| LOTTO_001 | 동행복권 API 응답 실패 (네트워크, 타임아웃 등) | 502 | "당첨 번호를 불러올 수 없습니다. 잠시 후 다시 시도해주세요." |
| LOTTO_002 | 존재하지 않는 회차 조회 | 404 | "해당 회차의 추첨 결과가 없습니다." |
| LOTTO_003 | 미래 회차 번호 등록 시도 | 400 | "아직 판매 중이지 않은 회차입니다." |
| LOTTO_004 | 번호 유효성 검증 실패 (범위, 중복, 개수) | 400 | "입력한 번호를 확인해주세요. [상세 사유]" |
| LOTTO_005 | 회차당 5게임 초과 | 400 | "한 회차에 최대 5게임까지 등록할 수 있습니다. (현재 {N}게임 등록됨)" |
| LOTTO_006 | 인증 실패 / 토큰 만료 | 401 | "로그인이 필요합니다." |
| LOTTO_007 | 타인의 게임 삭제 시도 | 403 | "삭제 권한이 없습니다." |
| LOTTO_008 | 당첨 게임 삭제 시도 | 400 | "당첨된 게임은 삭제할 수 없습니다." |
| LOTTO_009 | 서버 내부 오류 | 500 | "일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요." |

### 8.3 예외 처리 구현

```java
// ErrorCode.java - 에러 코드 열거형

public enum ErrorCode {

  EXTERNAL_API_FAILURE("LOTTO_001", "당첨 번호를 불러올 수 없습니다. 잠시 후 다시 시도해주세요.",
    HttpStatus.BAD_GATEWAY),
  DRAWING_NOT_FOUND("LOTTO_002", "해당 회차의 추첨 결과가 없습니다.",
    HttpStatus.NOT_FOUND),
  INVALID_ROUND("LOTTO_003", "아직 판매 중이지 않은 회차입니다.",
    HttpStatus.BAD_REQUEST),
  INVALID_NUMBERS("LOTTO_004", "입력한 번호를 확인해주세요.",
    HttpStatus.BAD_REQUEST),
  GAME_LIMIT_EXCEEDED("LOTTO_005", "한 회차에 최대 5게임까지 등록할 수 있습니다.",
    HttpStatus.BAD_REQUEST),
  UNAUTHORIZED("LOTTO_006", "로그인이 필요합니다.",
    HttpStatus.UNAUTHORIZED),
  FORBIDDEN("LOTTO_007", "삭제 권한이 없습니다.",
    HttpStatus.FORBIDDEN),
  CANNOT_DELETE_WON_GAME("LOTTO_008", "당첨된 게임은 삭제할 수 없습니다.",
    HttpStatus.BAD_REQUEST),
  INTERNAL_SERVER_ERROR("LOTTO_009", "일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.",
    HttpStatus.INTERNAL_SERVER_ERROR);

  private final String code;
  private final String message;
  private final HttpStatus status;

  ErrorCode(String code, String message, HttpStatus status) {
    this.code = code;
    this.message = message;
    this.status = status;
  }

  // getter methods...
}

// LottoException.java

public class LottoException extends RuntimeException {
  private final ErrorCode errorCode;

  public LottoException(ErrorCode errorCode) {
    super(errorCode.getMessage());
    this.errorCode = errorCode;
  }

  public LottoException(ErrorCode errorCode, String detail) {
    super(errorCode.getMessage() + " - " + detail);
    this.errorCode = errorCode;
  }

  public ErrorCode getErrorCode() {
    return errorCode;
  }
}

// LottoExceptionHandler.java - GlobalExceptionHandler

@RestControllerAdvice
public class LottoExceptionHandler {

  @ExceptionHandler(LottoException.class)
  public ResponseEntity<ErrorResponse> handleLottoException(LottoException e) {
    ErrorCode errorCode = e.getErrorCode();
    ErrorResponse response = ErrorResponse.builder()
      .code(errorCode.getCode())
      .message(errorCode.getMessage())
      .timestamp(LocalDateTime.now())
      .build();

    return ResponseEntity
      .status(errorCode.getStatus())
      .body(response);
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ErrorResponse> handleGeneralException(Exception e) {
    ErrorResponse response = ErrorResponse.builder()
      .code("LOTTO_009")
      .message("일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.")
      .timestamp(LocalDateTime.now())
      .build();

    return ResponseEntity
      .status(HttpStatus.INTERNAL_SERVER_ERROR)
      .body(response);
  }
}
```

---

## 9. 인증/인가 처리

### 9.1 JWT 인증

**구매 관련 API는 모두 JWT 인증 필수:**
- `POST /lotto/purchases`
- `GET /lotto/purchases`
- `GET /lotto/purchases/summary`
- `DELETE /lotto/purchases/{gameId}`

**인증 안 함:**
- `GET /lotto/winning-numbers` (공개 API)

### 9.2 권한 검증

- 구매 이력 조회 및 삭제: 본인의 데이터만 조회/삭제 가능
- 삭제 권한: memberId 비교로 검증

### 9.3 SecurityConfig 예시

```java
// SecurityConfig.java

@Configuration
@EnableWebSecurity
public class SecurityConfig {

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf().disable()
      .authorizeRequests()
        .antMatchers(HttpMethod.GET, "/lotto/winning-numbers").permitAll()
        .antMatchers("/lotto/**").authenticated()
        .anyRequest().permitAll()
      .and()
      .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);

    return http.build();
  }

  @Bean
  public JwtAuthenticationFilter jwtAuthenticationFilter() {
    return new JwtAuthenticationFilter();  // auth-server와 동일 필터 재사용
  }
}
```

---

## 10. 의존성 설정

### 10.1 build.gradle

```gradle
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.5.0'
  id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.my_back_office'
version = '1.0.0'
sourceCompatibility = '17'

repositories {
  mavenCentral()
}

dependencies {
  // Spring Boot
  implementation 'org.springframework.boot:spring-boot-starter-web:3.5.0'
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa:3.5.0'
  implementation 'org.springframework.boot:spring-boot-starter-security:3.5.0'

  // Database
  runtimeOnly 'mysql:mysql-connector-java:8.0.33'

  // JWT (auth-server와 동일)
  implementation 'com.auth0:java-jwt:4.4.0'

  // Lombok
  compileOnly 'org.projectlombok:lombok:1.18.30'
  annotationProcessor 'org.projectlombok:lombok:1.18.30'

  // RestTemplate (HTTP 클라이언트)
  implementation 'org.springframework.boot:spring-boot-starter-webflux:3.5.0'

  // 테스트
  testImplementation 'org.springframework.boot:spring-boot-starter-test:3.5.0'
  testImplementation 'org.springframework.security:spring-security-test'

  // 로깅
  implementation 'org.springframework.boot:spring-boot-starter-logging:3.5.0'
}

tasks.named('test') {
  useJUnitPlatform()
}
```

---

## 11. 회차 계산 유틸리티

### 11.1 현재 회차 계산 공식

**기준:**
- 첫 회차 추첨일: 2002-12-07 (토요일)
- 추첨 주기: 매주 토요일 20:45

**계산식:**
```
현재 회차 = ((현재 날짜 - 기준일) / 7) + 1

예시:
2026-02-14 기준
= ((2026-02-14 - 2002-12-07) / 7) + 1
= (8470일 / 7) + 1
= 1210 + 1
= 제 1211회
```

### 11.2 구현 코드

```java
// RoundCalculator.java

public class RoundCalculator {

  private static final LocalDate FIRST_DRAWING_DATE = LocalDate.of(2002, 12, 7);

  /**
   * 현재 회차 계산
   * @return 현재 회차 번호
   */
  public static int getCurrentRound() {
    LocalDate today = LocalDate.now();
    long daysDiff = ChronoUnit.DAYS.between(FIRST_DRAWING_DATE, today);
    return (int) (daysDiff / 7) + 1;
  }

  /**
   * 지정된 회차의 추첨 예정일 계산
   * @param round 회차 번호
   * @return 추첨 예정일 (토요일)
   */
  public static LocalDate getDrawingDate(int round) {
    return FIRST_DRAWING_DATE.plusWeeks(round - 1);
  }

  /**
   * 해당 회차가 이미 추첨되었는지 확인
   * @param round 회차 번호
   * @return true if drawn
   */
  public static boolean isDrawn(int round) {
    LocalDateTime drawingDateTime = getDrawingDate(round)
      .atTime(20, 45);  // 추첨 시간: 20:45

    return LocalDateTime.now().isAfter(drawingDateTime);
  }
}
```

---

## 12. 구현 순서 및 체크리스트

### Phase 1: 기초 설정
- [ ] Spring Boot 프로젝트 생성
- [ ] 헥사고날 아키텍처 폴더 구조 생성
- [ ] DB 연결 설정 (application.yml)
- [ ] build.gradle 의존성 추가

### Phase 2: 도메인 계층
- [ ] LottoNumber, Round, Rank 값 객체 구현
- [ ] WinningNumbers, PurchaseGame 도메인 엔티티 구현
- [ ] RankCalculator 도메인 로직 구현
- [ ] RoundCalculator 유틸리티 구현

### Phase 3: 인프라 계층
- [ ] WinningNumbersEntity, PurchaseGameEntity JPA 엔티티 생성
- [ ] 데이터베이스 테이블 생성 (DDL)
- [ ] JPA Repository 인터페이스 생성
- [ ] DhlotteryApiClient 외부 API 클라이언트 구현
- [ ] 리포지토리 구현체 작성

### Phase 4: 애플리케이션 계층
- [ ] 포트(ITF) 인터페이스 정의
- [ ] LottoWinningService 구현
- [ ] PurchaseService 구현
- [ ] DataCleanupService 구현

### Phase 5: 웹 어댑터 계층
- [ ] DTO 클래스 작성
- [ ] LottoController 구현
- [ ] SecurityConfig 설정

### Phase 6: 배치 및 설정
- [ ] WinningNumberFetchJob 배치 구현
- [ ] DataCleanupJob 배치 구현
- [ ] SchedulingConfig 설정
- [ ] 에러 핸들링 (Exception Handler)

### Phase 7: 테스트
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성
- [ ] API 문서 생성 (Swagger/OpenAPI)

---

## 13. 성능 및 운영 가이드

### 13.1 응답 시간 목표
- 당첨 번호 조회: 500ms 이내 (캐시 적중 시)
- 구매 등록: 1초 이내
- 이력 조회: 2초 이내 (페이지 크기: 20)

### 13.2 인덱싱 전략
- `winning_numbers`: round (UNIQUE), draw_date
- `purchase_games`: (member_id, round), round, created_at

### 13.3 캐싱 전략
- 당첨 번호 (winning_numbers 테이블): 주 1회 변경 → DB 캐시 충분
- 구매 이력: 사용자별로 자주 변경 → 캐시 미사용

### 13.4 모니터링 지표
- 배치 실행 시간 및 성공/실패율
- API 응답 시간 (p50, p95, p99)
- DB 쿼리 실행 시간
- 에러 발생 빈도

---

