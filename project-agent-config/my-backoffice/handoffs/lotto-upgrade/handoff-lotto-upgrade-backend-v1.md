# Lotto Service Upgrade - Backend Gemini Handoff Document

> 작성일: 2026-02-18 | 대상: Gemini Backend Developer | 최종 수정: 2026-02-18

---

## 1. 서비스 개요

### 1.1 목적
기존 프론트엔드 전용 로또 번호 생성기(머니뚝)에 백엔드를 추가하여, **스케줄 기반 자동 번호 생성 + 메신저 전송** 기능을 구현한다.

### 1.2 기술 스택
| 항목 | 내용 |
|---|---|
| 프레임워크 | Spring Boot 3.5.0 |
| 자바 버전 | Java 17 |
| 서버 포트 | 8082 |
| 데이터베이스 | MySQL (기존 auth-server의 `member` DB 공유) |
| 배치 | Spring Batch |
| 아키텍처 | 헥사고날 아키텍처 (포트-어댑터) |
| 빌드 도구 | Gradle (Kotlin DSL) |
| API 통신 | REST (JSON) |
| HTTP 클라이언트 | WebFlux WebClient (Telegram API 호출용) |

### 1.3 서비스 위치
- **API Gateway 라우팅**: `http://localhost:8080/lotto/**` → `http://localhost:8082`
- **직접 접근**: `http://localhost:8082`

### 1.4 핵심 제약
- **서버 교체 가능성**: 추후 Node.js로 교체 예정. 프론트엔드 수정 없이 서버만 교체 가능하도록:
  - 응답은 순수 JSON (Spring HATEOAS 등 프레임워크 의존 구조 금지)
  - 에러 응답 `{ "code", "message" }` 통일
  - JWT는 Authorization 헤더
  - API 버전 프리픽스 `/api/v1/`
  - 페이지네이션 `{ content, page, size, totalElements }` 통일

### 1.5 참조 프로젝트
- **auth-server**: `D:\workspace\02. my-backoffice\auth-server\` — 헥사고날 아키텍처 레퍼런스
- **기존 lotto-server 핸드오프**: `handoffs/lotto-service-backend-gemini-handoff.md` — 구매/당첨 기능 설계 (이번과 별도)

---

## 2. 프로젝트 구조 (헥사고날 아키텍처)

```
com.my_back_office.lotto_server/
├── domain/                                    # 순수 도메인 (Spring 의존 없음)
│   ├── generation/
│   │   ├── GeneratedNumber.java               # 생성된 번호 도메인 엔티티
│   │   ├── LotteryType.java                   # Enum: LOTTO_645, PENSION_720
│   │   └── vo/
│   │       ├── Strategy.java                  # Enum: RANDOM, BALANCED, ODD_EVEN
│   │       └── NumberSets.java                # List<List<Integer>> 래핑 VO
│   ├── schedule/
│   │   └── Schedule.java                      # 스케줄 도메인 엔티티
│   ├── messaging/
│   │   ├── MessagingAccount.java              # 메신저 계정 도메인 엔티티
│   │   ├── MessageHistory.java                # 전송 이력 도메인 엔티티
│   │   └── vo/
│   │       ├── Provider.java                  # Enum: TELEGRAM, EMAIL, KAKAO
│   │       └── MessageStatus.java             # Enum: PENDING, SENT, FAILED
│   └── generator/                             # 번호 생성 순수 도메인 로직
│       ├── NumberGenerator.java               # Interface
│       ├── RandomGenerator.java               # 완전 랜덤
│       ├── BalancedGenerator.java             # 균형 배분
│       ├── OddEvenGenerator.java              # 홀짝 균형
│       └── NumberGeneratorFactory.java        # Strategy → Generator 매핑
│
├── application/                               # 유스케이스 계층
│   ├── port/
│   │   ├── in/                                # 입력 포트 (유스케이스 인터페이스)
│   │   │   ├── GenerateNumbersUseCaseITF.java
│   │   │   ├── ManageScheduleUseCaseITF.java
│   │   │   ├── ManageMessagingAccountUseCaseITF.java
│   │   │   └── SendMessageUseCaseITF.java
│   │   └── out/                               # 출력 포트 (리포지토리/외부 인터페이스)
│   │       ├── GeneratedNumberRepositoryITF.java
│   │       ├── ScheduleRepositoryITF.java
│   │       ├── MessagingAccountRepositoryITF.java
│   │       ├── MessageHistoryRepositoryITF.java
│   │       └── MessagingProviderITF.java      # 메신저 전송 추상화
│   └── service/
│       ├── GenerateNumbersService.java
│       ├── ScheduleService.java
│       ├── MessagingAccountService.java
│       └── SendMessageService.java
│
├── infra/                                     # 인프라 계층 (외부 기술)
│   ├── jpa/                                   # Spring Data JPA 인터페이스
│   │   ├── GeneratedNumberJpaRepository.java
│   │   ├── ScheduleJpaRepository.java
│   │   ├── MessagingAccountJpaRepository.java
│   │   └── MessageHistoryJpaRepository.java
│   ├── persistence/                           # JPA Entity + 리포지토리 구현
│   │   ├── GeneratedNumberEntity.java
│   │   ├── GeneratedNumberRepository.java     # implements GeneratedNumberRepositoryITF
│   │   ├── ScheduleEntity.java
│   │   ├── ScheduleRepository.java
│   │   ├── MessagingAccountEntity.java
│   │   ├── MessagingAccountRepository.java
│   │   ├── MessageHistoryEntity.java
│   │   └── MessageHistoryRepository.java
│   └── messaging/                             # 메신저 프로바이더 어댑터
│       └── TelegramMessagingAdapter.java      # implements MessagingProviderITF
│
├── web/                                       # REST 컨트롤러
│   ├── ScheduleController.java
│   ├── MessagingAccountController.java
│   ├── GeneratedNumberController.java
│   └── dto/
│       ├── CreateScheduleRequest.java
│       ├── UpdateScheduleRequest.java
│       ├── ScheduleResponse.java
│       ├── CreateMessagingAccountRequest.java
│       ├── UpdateMessagingAccountRequest.java
│       ├── MessagingAccountResponse.java
│       ├── GeneratedNumberResponse.java
│       ├── MessageHistoryResponse.java
│       ├── PageResponse.java                  # 공통 페이지네이션 래퍼
│       └── ErrorResponse.java                 # 공통 에러 응답
│
├── batch/                                     # Spring Batch
│   ├── BatchConfig.java                       # JobRepository, DataSource 설정
│   ├── NumberGenerationJobConfig.java         # Job + Step 빈 정의
│   ├── GenerateAndSendTasklet.java            # 번호 생성 + 전송 Tasklet
│   └── BatchScheduler.java                    # @Scheduled로 Job 실행
│
├── global/
│   ├── config/
│   │   ├── SecurityConfig.java                # JWT 인증 설정
│   │   └── WebClientConfig.java               # WebClient 빈 설정
│   ├── jwt/
│   │   ├── JwtTokenProvider.java              # JWT 파싱/검증
│   │   └── JwtAuthenticationFilter.java       # Security 필터
│   └── exception/
│       ├── GlobalExceptionHandler.java        # @RestControllerAdvice
│       └── ErrorCode.java                     # 에러 코드 enum
│
└── LottoServerApplication.java
```

---

## 3. 데이터베이스 스키마

기존 `member` DB에 `lotto_` 접두사 테이블 4개 추가.

### 3.1 메신저 계정 테이블 (lotto_messaging_account)

```sql
CREATE TABLE lotto_messaging_account (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id         BIGINT NOT NULL COMMENT '사용자 ID',
    provider          VARCHAR(30) NOT NULL COMMENT '메신저 종류 (TELEGRAM, EMAIL, KAKAO)',
    display_name      VARCHAR(100) COMMENT '사용자가 지정한 별명',
    credentials_json  JSON NOT NULL COMMENT 'provider별 인증 정보 (예: {"chatId":"123"})',
    is_verified       BOOLEAN NOT NULL DEFAULT FALSE COMMENT '테스트 메시지 전송 성공 여부',
    created_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE,
    INDEX idx_member_provider (member_id, provider)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.2 스케줄 테이블 (lotto_schedule)

```sql
CREATE TABLE lotto_schedule (
    id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id             BIGINT NOT NULL COMMENT '사용자 ID',
    lottery_type          VARCHAR(20) NOT NULL COMMENT 'LOTTO_645 또는 PENSION_720',
    strategy              VARCHAR(20) NOT NULL COMMENT 'RANDOM, BALANCED, ODD_EVEN',
    set_count             INT NOT NULL DEFAULT 5 COMMENT '생성할 게임 수 (1~5)',
    cron_expression       VARCHAR(50) COMMENT 'null이면 기본값: 매주 토요일 17:00',
    messaging_account_id  BIGINT COMMENT '전송 대상 메신저 계정',
    enabled               BOOLEAN NOT NULL DEFAULT TRUE,
    last_run_at           DATETIME COMMENT '마지막 실행 시각',
    created_at            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE,
    FOREIGN KEY (messaging_account_id) REFERENCES lotto_messaging_account(id) ON DELETE SET NULL,
    INDEX idx_member (member_id),
    INDEX idx_enabled (enabled)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.3 생성된 번호 테이블 (lotto_generated_number)

```sql
CREATE TABLE lotto_generated_number (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id       BIGINT NOT NULL COMMENT '사용자 ID',
    lottery_type    VARCHAR(20) NOT NULL COMMENT 'LOTTO_645 또는 PENSION_720',
    strategy        VARCHAR(20) NOT NULL COMMENT '사용된 전략',
    numbers_json    JSON NOT NULL COMMENT '생성된 번호 세트 (예: [[3,12,25,31,38,44], ...])',
    generated_at    DATETIME NOT NULL COMMENT '생성 시각',
    schedule_id     BIGINT COMMENT '스케줄에 의한 생성 시 해당 스케줄 ID (수동 생성은 null)',
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE,
    INDEX idx_member_generated (member_id, generated_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.4 메시지 전송 이력 테이블 (lotto_message_history)

```sql
CREATE TABLE lotto_message_history (
    id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id             BIGINT NOT NULL COMMENT '사용자 ID',
    messaging_account_id  BIGINT NOT NULL COMMENT '전송 대상 계정',
    generated_number_id   BIGINT NOT NULL COMMENT '전송한 번호 ID',
    status                VARCHAR(20) NOT NULL COMMENT 'PENDING, SENT, FAILED',
    error_message         VARCHAR(500) COMMENT '실패 시 에러 메시지',
    sent_at               DATETIME COMMENT '실제 전송 시각',
    created_at            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (member_id) REFERENCES member(id) ON DELETE CASCADE,
    FOREIGN KEY (messaging_account_id) REFERENCES lotto_messaging_account(id),
    FOREIGN KEY (generated_number_id) REFERENCES lotto_generated_number(id),
    INDEX idx_member_status (member_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.5 Spring Batch 메타 테이블
- `spring.batch.jdbc.initialize-schema: always` 설정으로 자동 생성
- `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등

---

## 4. REST API 명세

Base Path: `/lotto/api/v1`
인증: 모든 엔드포인트에 JWT 필수 (`Authorization: Bearer {token}`)

---

### 4.1 POST /lotto/api/v1/schedules — 스케줄 생성

**요청:**
```json
{
  "lotteryType": "LOTTO_645",
  "strategy": "RANDOM",
  "setCount": 5,
  "cronExpression": null,
  "messagingAccountId": 1,
  "enabled": true
}
```

**검증 규칙:**
| 필드 | 규칙 | 에러 코드 |
|---|---|---|
| lotteryType | LOTTO_645 또는 PENSION_720 | INVALID_LOTTERY_TYPE |
| strategy | RANDOM, BALANCED, ODD_EVEN 중 하나 | INVALID_STRATEGY |
| setCount | 1~5 | INVALID_SET_COUNT |
| cronExpression | null이면 기본값 `0 0 17 ? * SAT` 적용 | - |
| messagingAccountId | 존재하는 본인 계정이어야 함 | MESSAGING_ACCOUNT_NOT_FOUND |

**응답 (201 Created):**
```json
{
  "id": 1,
  "lotteryType": "LOTTO_645",
  "strategy": "RANDOM",
  "setCount": 5,
  "cronExpression": "0 0 17 ? * SAT",
  "messagingAccountId": 1,
  "enabled": true,
  "lastRunAt": null,
  "createdAt": "2026-02-18T10:00:00",
  "updatedAt": "2026-02-18T10:00:00"
}
```

### 4.2 GET /lotto/api/v1/schedules — 내 스케줄 목록

**응답 (200 OK):**
```json
[
  {
    "id": 1,
    "lotteryType": "LOTTO_645",
    "strategy": "RANDOM",
    "setCount": 5,
    "cronExpression": "0 0 17 ? * SAT",
    "messagingAccountId": 1,
    "enabled": true,
    "lastRunAt": "2026-02-15T17:00:00",
    "createdAt": "2026-02-10T10:00:00",
    "updatedAt": "2026-02-15T17:00:00"
  }
]
```

### 4.3 PUT /lotto/api/v1/schedules/{id} — 스케줄 수정

**요청:** POST와 동일한 body. 본인 소유 스케줄만 수정 가능.

**응답 (200 OK):** 수정된 스케줄 객체 반환.

**에러:** `SCHEDULE_NOT_FOUND` (404), `FORBIDDEN` (403)

### 4.4 DELETE /lotto/api/v1/schedules/{id} — 스케줄 삭제

**응답 (204 No Content)**

### 4.5 PATCH /lotto/api/v1/schedules/{id}/toggle — 활성화/비활성화

**응답 (200 OK):**
```json
{
  "id": 1,
  "enabled": false
}
```

---

### 4.6 POST /lotto/api/v1/messaging-accounts — 메신저 계정 등록

**요청:**
```json
{
  "provider": "TELEGRAM",
  "displayName": "내 텔레그램",
  "credentials": {
    "chatId": "123456789"
  }
}
```

**검증:**
| 필드 | 규칙 | 에러 코드 |
|---|---|---|
| provider | TELEGRAM, EMAIL, KAKAO 중 하나 | INVALID_PROVIDER |
| credentials | provider별 필수 필드 존재 여부 | INVALID_CREDENTIALS |
| 최대 개수 | 사용자당 5개 | MAX_ACCOUNTS_EXCEEDED |

**provider별 credentials 필수 필드:**
| Provider | 필수 필드 | 예시 |
|---|---|---|
| TELEGRAM | chatId | `{"chatId": "123456789"}` |
| EMAIL | email | `{"email": "user@example.com"}` |
| KAKAO | (추후 정의) | - |

**응답 (201 Created):**
```json
{
  "id": 1,
  "provider": "TELEGRAM",
  "displayName": "내 텔레그램",
  "isVerified": false,
  "createdAt": "2026-02-18T10:00:00"
}
```

> 주의: `credentials`는 응답에 포함하지 않음 (보안)

### 4.7 GET /lotto/api/v1/messaging-accounts — 내 계정 목록

**응답 (200 OK):**
```json
[
  {
    "id": 1,
    "provider": "TELEGRAM",
    "displayName": "내 텔레그램",
    "isVerified": true,
    "createdAt": "2026-02-18T10:00:00"
  }
]
```

### 4.8 PUT /lotto/api/v1/messaging-accounts/{id} — 계정 수정

**요청:** POST와 동일. 수정 시 `isVerified`가 false로 리셋됨.

### 4.9 DELETE /lotto/api/v1/messaging-accounts/{id} — 계정 삭제

**응답 (204 No Content)**

**주의:** 이 계정을 참조하는 스케줄의 `messaging_account_id`가 NULL로 설정됨 (FK ON DELETE SET NULL).

### 4.10 POST /lotto/api/v1/messaging-accounts/{id}/test — 테스트 메시지

**설명:** 해당 계정으로 테스트 메시지를 전송하고, 성공 시 `isVerified`를 true로 변경.

**요청:** Body 없음

**응답 (200 OK):**
```json
{
  "success": true,
  "message": "테스트 메시지가 전송되었습니다."
}
```

**에러 (502):**
```json
{
  "code": "MESSAGING_SEND_FAILED",
  "message": "메시지 전송에 실패했습니다. 계정 정보를 확인해주세요."
}
```

---

### 4.11 GET /lotto/api/v1/generated-numbers — 생성된 번호 이력

**요청 파라미터:**
| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| page | INT | N | 0 | 페이지 번호 |
| size | INT | N | 20 | 페이지 크기 (최대 100) |

**응답 (200 OK):**
```json
{
  "content": [
    {
      "id": 1,
      "lotteryType": "LOTTO_645",
      "strategy": "BALANCED",
      "sets": [[3, 12, 25, 31, 38, 44], [1, 9, 18, 27, 36, 42]],
      "generatedAt": "2026-02-15T17:00:00",
      "scheduleId": 1,
      "messageStatus": "SENT"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 50
}
```

> `messageStatus`는 `lotto_message_history`에서 조인하여 가져옴. 이력이 없으면 null.

### 4.12 DELETE /lotto/api/v1/generated-numbers/{id} — 생성 번호 삭제

**응답 (204 No Content)**

### 4.13 GET /lotto/api/v1/messages — 전송 이력 조회

**요청 파라미터:**
| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| page | INT | N | 0 | 페이지 번호 |
| size | INT | N | 20 | 페이지 크기 |

**응답 (200 OK):**
```json
{
  "content": [
    {
      "id": 1,
      "provider": "TELEGRAM",
      "displayName": "내 텔레그램",
      "generatedNumberId": 1,
      "status": "SENT",
      "errorMessage": null,
      "sentAt": "2026-02-15T17:00:05",
      "createdAt": "2026-02-15T17:00:00"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 30
}
```

---

## 5. 에러 코드 정의

| 코드 | HTTP | 메시지 | 상황 |
|---|---|---|---|
| INVALID_LOTTERY_TYPE | 400 | 유효하지 않은 복권 종류입니다. | 잘못된 lotteryType |
| INVALID_STRATEGY | 400 | 유효하지 않은 전략입니다. | 잘못된 strategy |
| INVALID_SET_COUNT | 400 | 게임 수는 1~5 사이여야 합니다. | setCount 범위 초과 |
| INVALID_PROVIDER | 400 | 지원하지 않는 메신저입니다. | 잘못된 provider |
| INVALID_CREDENTIALS | 400 | 계정 인증 정보가 올바르지 않습니다. | credentials 필수 필드 누락 |
| MAX_ACCOUNTS_EXCEEDED | 400 | 메신저 계정은 최대 5개까지 등록 가능합니다. | 계정 5개 초과 |
| SCHEDULE_NOT_FOUND | 404 | 스케줄을 찾을 수 없습니다. | 존재하지 않는 스케줄 |
| MESSAGING_ACCOUNT_NOT_FOUND | 404 | 메신저 계정을 찾을 수 없습니다. | 존재하지 않는 계정 |
| GENERATED_NUMBER_NOT_FOUND | 404 | 생성된 번호를 찾을 수 없습니다. | 존재하지 않는 번호 |
| FORBIDDEN | 403 | 접근 권한이 없습니다. | 다른 사용자의 리소스 접근 |
| MESSAGING_SEND_FAILED | 502 | 메시지 전송에 실패했습니다. | 텔레그램 API 오류 등 |
| UNAUTHORIZED | 401 | 인증이 필요합니다. | JWT 누락/만료 |
| INTERNAL_ERROR | 500 | 서버 내부 오류입니다. | 예상치 못한 오류 |

**에러 응답 형식 (공통):**
```json
{
  "code": "SCHEDULE_NOT_FOUND",
  "message": "스케줄을 찾을 수 없습니다."
}
```

---

## 6. 번호 생성 로직 (도메인)

프론트엔드 구현(`lotto-app/components/LottoTab.vue`)을 Java로 포팅.

### 6.1 NumberGenerator 인터페이스

```java
public interface NumberGenerator {
    List<List<Integer>> generate(int setCount);
    Strategy getStrategy();
}
```

### 6.2 RandomGenerator (완전 랜덤)

```java
// 1~45에서 6개 무작위 추출
List<Integer> pool = IntStream.rangeClosed(1, 45).boxed().collect(toList());
Collections.shuffle(pool);
List<Integer> numbers = pool.subList(0, 6);
Collections.sort(numbers);
```

### 6.3 BalancedGenerator (균형 배분)

```java
// 5개 구간에서 각 1개 + 나머지에서 1개 = 6개
int[][] ranges = {{1,9}, {10,19}, {20,29}, {30,39}, {40,45}};
List<Integer> result = new ArrayList<>();
for (int[] range : ranges) {
    result.add(randomInRange(range[0], range[1]));
}
// 전체 1~45 중 아직 선택되지 않은 번호에서 1개 추가
List<Integer> remaining = IntStream.rangeClosed(1, 45)
    .filter(n -> !result.contains(n))
    .boxed().collect(toList());
Collections.shuffle(remaining);
result.add(remaining.get(0));
Collections.sort(result);
```

### 6.4 OddEvenGenerator (홀짝 균형)

```java
// 홀수 3개 + 짝수 3개
List<Integer> odds = IntStream.rangeClosed(1, 45).filter(n -> n % 2 == 1).boxed().collect(toList());
List<Integer> evens = IntStream.rangeClosed(1, 45).filter(n -> n % 2 == 0).boxed().collect(toList());
Collections.shuffle(odds);
Collections.shuffle(evens);
List<Integer> result = new ArrayList<>();
result.addAll(odds.subList(0, 3));
result.addAll(evens.subList(0, 3));
Collections.sort(result);
```

### 6.5 NumberGeneratorFactory

```java
@Component
public class NumberGeneratorFactory {
    private final Map<Strategy, NumberGenerator> generators;

    public NumberGeneratorFactory(List<NumberGenerator> generatorList) {
        this.generators = generatorList.stream()
            .collect(Collectors.toMap(NumberGenerator::getStrategy, Function.identity()));
    }

    public NumberGenerator getGenerator(Strategy strategy) {
        NumberGenerator generator = generators.get(strategy);
        if (generator == null) throw new IllegalArgumentException("Unknown strategy: " + strategy);
        return generator;
    }
}
```

---

## 7. 메신저 연동

### 7.1 MessagingProviderITF (출력 포트)

```java
public interface MessagingProviderITF {
    /**
     * 메시지 전송
     * @param message 전송할 메시지 본문
     * @param credentials provider별 인증 정보 (JSON 파싱 결과)
     * @throws MessagingSendException 전송 실패 시
     */
    void send(String message, Map<String, String> credentials);

    Provider getProvider();
}
```

### 7.2 TelegramMessagingAdapter (MVP 구현)

```java
@Component
public class TelegramMessagingAdapter implements MessagingProviderITF {

    private static final String TELEGRAM_API = "https://api.telegram.org/bot%s/sendMessage";

    @Value("${messaging.telegram.bot-token}")
    private String botToken;

    private final WebClient webClient;

    @Override
    public void send(String message, Map<String, String> credentials) {
        String chatId = credentials.get("chatId");
        if (chatId == null) throw new MessagingSendException("chatId is required");

        String url = String.format(TELEGRAM_API, botToken);

        webClient.post()
            .uri(url)
            .bodyValue(Map.of(
                "chat_id", chatId,
                "text", message,
                "parse_mode", "Markdown"
            ))
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }

    @Override
    public Provider getProvider() {
        return Provider.TELEGRAM;
    }
}
```

### 7.3 메시지 포맷

```
🎰 머니뚝 로또 번호 알림

📅 2026-02-21 (토) 17:00
🎯 전략: 균형 배분
🎮 5게임

Game 1: 3 - 12 - 25 - 31 - 38 - 44
Game 2: 1 - 9 - 18 - 27 - 36 - 42
Game 3: 7 - 15 - 23 - 34 - 40 - 45
Game 4: 2 - 11 - 21 - 33 - 39 - 43
Game 5: 6 - 14 - 26 - 35 - 41 - 44

행운을 빕니다! 🍀
```

### 7.4 새 메신저 추가 방법

1. `MessagingProviderITF` 구현 클래스 생성 (예: `EmailMessagingAdapter`)
2. `@Component`로 등록
3. `Provider` enum에 값 추가
4. credentials 검증 로직 추가
5. **기존 코드 수정 불필요** — Factory/Strategy 패턴으로 자동 등록

---

## 8. Spring Batch 설계

### 8.1 BatchConfig

```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {
    // Spring Batch가 JobRepository를 위해 사용할 DataSource 설정
    // 기존 MySQL DataSource를 그대로 사용
}
```

### 8.2 NumberGenerationJobConfig

```java
@Configuration
public class NumberGenerationJobConfig {

    @Bean
    public Job numberGenerationJob(JobRepository jobRepository, Step generateAndSendStep) {
        return new JobBuilder("numberGenerationJob", jobRepository)
            .start(generateAndSendStep)
            .build();
    }

    @Bean
    public Step generateAndSendStep(JobRepository jobRepository,
                                     PlatformTransactionManager transactionManager,
                                     GenerateAndSendTasklet tasklet) {
        return new StepBuilder("generateAndSendStep", jobRepository)
            .tasklet(tasklet, transactionManager)
            .build();
    }
}
```

### 8.3 GenerateAndSendTasklet

```java
@Component
public class GenerateAndSendTasklet implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        // 1. enabled=true인 스케줄 전체 조회
        List<Schedule> schedules = scheduleRepository.findAllEnabled();

        for (Schedule schedule : schedules) {
            // 2. 전략에 따라 번호 생성
            NumberGenerator generator = generatorFactory.getGenerator(schedule.getStrategy());
            List<List<Integer>> numbers = generator.generate(schedule.getSetCount());

            // 3. DB 저장
            GeneratedNumber generated = GeneratedNumber.create(
                schedule.getMemberId(), schedule.getLotteryType(),
                schedule.getStrategy(), numbers, schedule.getId()
            );
            generatedNumberRepository.save(generated);

            // 4. 메신저로 전송
            if (schedule.getMessagingAccountId() != null) {
                MessagingAccount account = messagingAccountRepository
                    .findById(schedule.getMessagingAccountId()).orElse(null);
                if (account != null) {
                    String message = formatMessage(schedule, numbers);
                    try {
                        messagingProvider.send(message, account.getCredentials());
                        messageHistoryRepository.save(
                            MessageHistory.sent(schedule.getMemberId(), account.getId(), generated.getId())
                        );
                    } catch (Exception e) {
                        messageHistoryRepository.save(
                            MessageHistory.failed(schedule.getMemberId(), account.getId(),
                                generated.getId(), e.getMessage())
                        );
                    }
                }
            }

            // 5. lastRunAt 업데이트
            schedule.markExecuted();
            scheduleRepository.save(schedule);
        }

        return RepeatStatus.FINISHED;
    }
}
```

### 8.4 BatchScheduler

```java
@Component
@EnableScheduling
public class BatchScheduler {

    private final JobLauncher jobLauncher;
    private final Job numberGenerationJob;

    @Scheduled(cron = "0 0 17 ? * SAT")  // 매주 토요일 17:00
    public void runNumberGenerationJob() {
        JobParameters params = new JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(numberGenerationJob, params);
    }
}
```

---

## 9. JWT 인증

auth-server와 동일한 JWT 구조 사용. 공유 Secret Key.

### 9.1 JwtTokenProvider

```java
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secretKey;

    public Long extractMemberId(String token) {
        DecodedJWT decoded = JWT.require(Algorithm.HMAC256(secretKey))
            .build()
            .verify(token);
        return decoded.getClaim("memberId").asLong();
    }
}
```

### 9.2 SecurityConfig

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

---

## 10. build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.0"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "com.my_back_office"
version = "0.0.1-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

dependencies {
    // Web
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")

    // Security + JWT
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("com.auth0:java-jwt:4.5.0")

    // Data
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.mysql:mysql-connector-j")

    // Spring Batch
    implementation("org.springframework.boot:spring-boot-starter-batch")

    // WebClient (Telegram API)
    implementation("org.springframework.boot:spring-boot-starter-webflux")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.batch:spring-batch-test")
    testImplementation("org.springframework.security:spring-security-test")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## 11. application.yml

```yaml
server:
  port: 8082

spring:
  application:
    name: lotto-server

  datasource:
    url: jdbc:mysql://localhost:3306/member?useSSL=false&serverTimezone=Asia/Seoul
    username: root
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        format_sql: true

  batch:
    jdbc:
      initialize-schema: always
    job:
      enabled: false  # 앱 시작 시 자동 실행 방지 (스케줄러가 관리)

jwt:
  secret: ${JWT_SECRET:myboproject}

messaging:
  telegram:
    bot-token: ${TELEGRAM_BOT_TOKEN}
```

---

## 12. 구현 체크리스트

### Phase 1: 프로젝트 셋업
- [ ] Spring Initializr로 프로젝트 생성 (또는 auth-server 구조 복사)
- [ ] `build.gradle.kts` 의존성 설정
- [ ] `application.yml` 설정
- [ ] `LottoServerApplication.java` 메인 클래스
- [ ] MySQL 테이블 4개 생성 (DDL 실행)

### Phase 2: 도메인 계층
- [ ] Enum: `LotteryType`, `Strategy`, `Provider`, `MessageStatus`
- [ ] VO: `NumberSets`
- [ ] 도메인 엔티티: `GeneratedNumber`, `Schedule`, `MessagingAccount`, `MessageHistory`
- [ ] `NumberGenerator` 인터페이스 + 3개 구현체
- [ ] `NumberGeneratorFactory`
- [ ] 도메인 단위 테스트 (각 생성기 전략 검증)

### Phase 3: 인프라 계층
- [ ] JPA 엔티티 4개 (`*Entity.java`) + `fromDomain()` / `toDomain()` 매핑
- [ ] JPA Repository 인터페이스 4개
- [ ] Repository 구현체 4개 (출력 포트 구현)
- [ ] `TelegramMessagingAdapter`
- [ ] `WebClientConfig`

### Phase 4: 애플리케이션 계층
- [ ] 입력 포트 인터페이스 4개
- [ ] 출력 포트 인터페이스 5개
- [ ] 서비스 구현체 4개

### Phase 5: 웹 계층
- [ ] DTO 클래스 (Request/Response)
- [ ] `PageResponse` 공통 래퍼
- [ ] `ErrorResponse` 공통 에러
- [ ] 컨트롤러 3개 (`ScheduleController`, `MessagingAccountController`, `GeneratedNumberController`)
- [ ] `GlobalExceptionHandler`
- [ ] `ErrorCode` enum

### Phase 6: 보안
- [ ] `JwtTokenProvider`
- [ ] `JwtAuthenticationFilter`
- [ ] `SecurityConfig`

### Phase 7: Spring Batch
- [ ] `BatchConfig`
- [ ] `NumberGenerationJobConfig`
- [ ] `GenerateAndSendTasklet`
- [ ] `BatchScheduler`
- [ ] 배치 통합 테스트

### Phase 8: 테스트
- [ ] 도메인 단위 테스트 (NumberGenerator 3종)
- [ ] 서비스 단위 테스트 (Mockito)
- [ ] 컨트롤러 통합 테스트 (@WebMvcTest)
- [ ] 배치 통합 테스트 (@SpringBatchTest)
- [ ] 전체 E2E: 스케줄 생성 → 배치 실행 → 번호 생성 → 텔레그램 수신
