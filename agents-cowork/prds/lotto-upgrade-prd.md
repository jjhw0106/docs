# Lotto Service Upgrade - Product Requirement Document (PRD)

> 작성일: 2026-02-18 | 최종 수정: 2026-02-18 | 상태: In Progress

---

## 1. 서비스 개요

### 1.1 목적
기존 프론트엔드 전용 로또 번호 생성기(머니뚝)에 두 가지 핵심 기능을 추가한다:
1. **PWA** — 모바일 홈 화면에 앱처럼 설치하여 즉시 접근
2. **자동 번호 생성 + 메신저 전송** — 백엔드에서 스케줄 기반으로 번호를 생성하고, 등록된 메신저 계정으로 자동 전송

### 1.2 배경
- 현재 lotto-app은 API 호출 0개, 100% 클라이언트 전용
- 사용자가 매번 접속해서 수동으로 번호를 생성해야 하는 불편함 존재
- 정기적으로 자동 생성 + 알림을 받고 싶은 니즈

### 1.3 기술 스택

| 구성 요소 | 기술 | 비고 |
|---|---|---|
| Frontend | Nuxt 4 + Vue 3 + Tailwind CSS | 기존 lotto-app |
| Backend | Spring Boot 3.5.0 + Java 17 | 신규 생성 |
| Batch | Spring Batch | 스케줄 기반 번호 생성 |
| DB | MySQL (기존 member DB 공유) | |
| Messaging | Telegram Bot API (MVP) | 추후 카카오/이메일 등 확장 |
| Gateway | Spring Cloud Gateway (기존) | `/lotto/**` → port 8082 |

### 1.4 핵심 제약 조건
- **서버 교체 가능성**: 백엔드는 학습 목적(Spring Batch)으로 Java/Spring 사용. 추후 Node.js로 교체 예정이므로, **프론트엔드 수정 없이 서버만 교체 가능**하도록 API 계약을 명확히 분리
- **기존 기능 유지**: 프론트엔드의 수동 번호 생성 기능은 그대로 유지

---

## 2. 사용자 스토리

### US-01: PWA 설치
> **사용자로서**, 모바일 브라우저에서 머니뚝을 홈 화면에 설치하고 싶다.
> **그래야** 앱스토어 없이도 네이티브 앱처럼 빠르게 접근할 수 있다.

**인수 조건:**
- 모바일 브라우저(Chrome, Safari)에서 "홈 화면에 추가" 가능
- 설치 후 전체 화면(standalone)으로 실행됨
- 앱 아이콘과 스플래시 화면이 표시됨
- 오프라인 시 캐시된 페이지 표시

### US-02: 메신저 계정 등록
> **사용자로서**, 번호를 받을 메신저 계정(텔레그램 등)을 등록하고 싶다.
> **그래야** 자동 생성된 번호를 메신저로 받을 수 있다.

**인수 조건:**
- 메신저 종류(Telegram 등) 선택 후 계정 정보 입력
- 테스트 메시지 전송으로 연결 확인 가능
- 여러 계정 등록 가능 (최대 5개)
- 계정 수정/삭제 가능

### US-03: 자동 번호 생성 스케줄 설정
> **사용자로서**, 원하는 전략과 주기로 자동 번호 생성 스케줄을 설정하고 싶다.
> **그래야** 매번 접속하지 않아도 정기적으로 번호를 받을 수 있다.

**인수 조건:**
- 복권 종류 선택: 로또 6/45 또는 연금복권 720+
- 생성 전략 선택: 완전 랜덤 / 균형 배분 / 홀짝 균형
- 게임 수 선택: 1~5세트
- 전송할 메신저 계정 선택
- 스케줄 활성화/비활성화 토글
- 기본 스케줄: 매주 토요일 17:00 (추첨 2시간 전)

### US-04: 자동 생성된 번호 확인
> **사용자로서**, 자동 생성된 번호 이력을 확인하고 싶다.
> **그래야** 과거에 어떤 번호가 생성/전송되었는지 추적할 수 있다.

**인수 조건:**
- 생성된 번호 목록 조회 (페이지네이션)
- 각 항목: 생성일시, 전략, 번호 세트, 전송 상태
- 개별 삭제 가능

### US-05: 메신저로 번호 수신
> **사용자로서**, 스케줄 시간에 맞춰 텔레그램으로 생성된 번호를 받고 싶다.
> **그래야** 추첨 전에 자동으로 번호를 확인할 수 있다.

**인수 조건:**
- 스케줄 시간에 배치 잡 실행 → 번호 생성 → 메신저 전송
- 메시지 포맷: 복권 종류, 전략, 번호 세트 포함
- 전송 실패 시 이력에 FAILED 상태 기록
- 전송 이력 조회 가능

---

## 3. 기능 범위

### 3.1 이번 릴리스 (MVP)

| 기능 | 영역 | 우선순위 |
|---|---|---|
| PWA 설치 | Frontend | P0 |
| 메신저 계정 CRUD | Backend | P0 |
| 스케줄 CRUD + 토글 | Backend | P0 |
| Spring Batch 번호 생성 잡 | Backend | P0 |
| 텔레그램 메시지 전송 | Backend | P0 |
| 생성 번호 이력 조회 | Backend | P1 |
| 전송 이력 조회 | Backend | P1 |
| 프론트엔드 설정 UI | Frontend | P2 (선택) |

### 3.2 향후 확장 (Out of Scope)

| 기능 | 비고 |
|---|---|
| 카카오톡 AlimTalk 연동 | 비즈앱 등록 필요 |
| 이메일 전송 | SMTP/SES 설정 |
| SMS 전송 | 유료 |
| 디스코드/LINE 연동 | 추가 어댑터 |
| Node.js 서버 전환 | API 계약 동일, 서버만 교체 |

---

## 4. API 설계

> **원칙**: 프론트엔드는 이 API만 호출. 서버를 Node.js로 교체해도 프론트 수정 0.

### 4.1 엔드포인트

모든 엔드포인트는 게이트웨이(port 8080)를 통해 접근. JWT 인증 필수.

```
Base Path: /lotto/api/v1

# 스케줄 관리
GET    /schedules                           # 내 스케줄 목록
POST   /schedules                           # 스케줄 생성
PUT    /schedules/{id}                      # 스케줄 수정
DELETE /schedules/{id}                      # 스케줄 삭제
PATCH  /schedules/{id}/toggle               # 활성화/비활성화

# 메신저 계정 관리
GET    /messaging-accounts                  # 내 계정 목록
POST   /messaging-accounts                  # 계정 등록
PUT    /messaging-accounts/{id}             # 계정 수정
DELETE /messaging-accounts/{id}             # 계정 삭제
POST   /messaging-accounts/{id}/test        # 테스트 메시지 전송

# 생성된 번호
GET    /generated-numbers                   # 생성 이력 (페이지네이션)
DELETE /generated-numbers/{id}              # 삭제

# 전송 이력
GET    /messages                            # 전송 이력 (페이지네이션)
```

### 4.2 요청/응답 형식

#### POST /schedules

```json
// Request
{
  "lotteryType": "LOTTO_645",
  "strategy": "RANDOM",
  "setCount": 5,
  "cronExpression": null,
  "messagingAccountId": 1,
  "enabled": true
}

// Response (201 Created)
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

#### POST /messaging-accounts

```json
// Request
{
  "provider": "TELEGRAM",
  "displayName": "내 텔레그램",
  "credentials": {
    "chatId": "123456789"
  }
}

// Response (201 Created)
{
  "id": 1,
  "provider": "TELEGRAM",
  "displayName": "내 텔레그램",
  "isVerified": false,
  "createdAt": "2026-02-18T10:00:00"
}
```

#### GET /generated-numbers

```json
// Response
{
  "content": [
    {
      "id": 1,
      "lotteryType": "LOTTO_645",
      "strategy": "BALANCED",
      "sets": [[3, 12, 25, 31, 38, 44], [1, 9, 18, 27, 36, 42]],
      "generatedAt": "2026-02-15T17:00:00",
      "messageStatus": "SENT"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 50
}
```

#### 에러 응답 (공통)

```json
{
  "code": "SCHEDULE_NOT_FOUND",
  "message": "스케줄을 찾을 수 없습니다."
}
```

### 4.3 서버 교체 안전장치

| 항목 | 규칙 |
|---|---|
| 응답 형식 | 순수 JSON (Spring HATEOAS 등 프레임워크 의존 구조 금지) |
| 인증 | JWT via Authorization 헤더 (쿠키 의존 금지) |
| 에러 | `{ "code", "message" }` 통일 |
| 버전 | `/api/v1/` 프리픽스 |
| 페이지네이션 | `{ content, page, size, totalElements }` 통일 |

---

## 5. 데이터 모델

### 5.1 DB: MySQL (기존 `member` 스키마 공유)

```sql
-- 스케줄 설정
CREATE TABLE lotto_schedule (
    id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id             BIGINT NOT NULL,
    lottery_type          VARCHAR(20) NOT NULL,         -- LOTTO_645, PENSION_720
    strategy              VARCHAR(20) NOT NULL,         -- RANDOM, BALANCED, ODD_EVEN
    set_count             INT NOT NULL DEFAULT 5,       -- 1~5
    cron_expression       VARCHAR(50),                  -- null = 기본값 (토요일 17:00)
    messaging_account_id  BIGINT,
    enabled               BOOLEAN NOT NULL DEFAULT TRUE,
    last_run_at           DATETIME,
    created_at            DATETIME NOT NULL,
    updated_at            DATETIME NOT NULL,
    FOREIGN KEY (member_id) REFERENCES member(member_id),
    FOREIGN KEY (messaging_account_id) REFERENCES lotto_messaging_account(id)
);

-- 메신저 계정
CREATE TABLE lotto_messaging_account (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id         BIGINT NOT NULL,
    provider          VARCHAR(30) NOT NULL,             -- TELEGRAM, EMAIL, KAKAO 등
    display_name      VARCHAR(100),
    credentials_json  JSON NOT NULL,                    -- provider별 인증 정보
    is_verified       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at        DATETIME NOT NULL,
    updated_at        DATETIME NOT NULL,
    FOREIGN KEY (member_id) REFERENCES member(member_id),
    INDEX idx_member_provider (member_id, provider)
);

-- 생성된 번호
CREATE TABLE lotto_generated_number (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id       BIGINT NOT NULL,
    lottery_type    VARCHAR(20) NOT NULL,
    strategy        VARCHAR(20) NOT NULL,
    numbers_json    JSON NOT NULL,                      -- [[3,12,25,31,38,44], ...]
    generated_at    DATETIME NOT NULL,
    schedule_id     BIGINT,
    FOREIGN KEY (member_id) REFERENCES member(member_id),
    INDEX idx_member_generated (member_id, generated_at DESC)
);

-- 메시지 전송 이력
CREATE TABLE lotto_message_history (
    id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
    member_id             BIGINT NOT NULL,
    messaging_account_id  BIGINT NOT NULL,
    generated_number_id   BIGINT NOT NULL,
    status                VARCHAR(20) NOT NULL,          -- SENT, FAILED, PENDING
    error_message         VARCHAR(500),
    sent_at               DATETIME,
    created_at            DATETIME NOT NULL,
    FOREIGN KEY (member_id) REFERENCES member(member_id),
    FOREIGN KEY (messaging_account_id) REFERENCES lotto_messaging_account(id),
    FOREIGN KEY (generated_number_id) REFERENCES lotto_generated_number(id)
);
```

### 5.2 Enum 정의

| Enum | 값 |
|---|---|
| LotteryType | `LOTTO_645`, `PENSION_720` |
| Strategy | `RANDOM`, `BALANCED`, `ODD_EVEN` |
| Provider | `TELEGRAM`, `EMAIL`, `KAKAO` |
| MessageStatus | `PENDING`, `SENT`, `FAILED` |

---

## 6. 번호 생성 전략

프론트엔드에 이미 구현된 로직을 백엔드(Java)로 포팅.

| 전략 | 설명 | 로직 |
|---|---|---|
| RANDOM | 1~45 전체에서 순수 랜덤 | `shuffle(1~45).slice(0, 6)` |
| BALANCED | 각 구간에서 1개씩 + 나머지 | 5개 구간(1-9, 10-19, 20-29, 30-39, 40-45)에서 각 1개 + 전체에서 1개 추가 |
| ODD_EVEN | 홀수 3개 + 짝수 3개 | 홀수 pool / 짝수 pool에서 각 3개 |

> 참고: 프론트엔드 구현은 `lotto-app/components/LottoTab.vue` 참조

---

## 7. 메신저 연동

### 7.1 MVP: 텔레그램

| 항목 | 내용 |
|---|---|
| API | `https://api.telegram.org/bot{token}/sendMessage` |
| 인증 | Bot Token (@BotFather에서 발급) |
| 사용자 식별 | chat_id (사용자가 봇에게 /start 후 획득) |
| 비용 | 무료 |
| Rate Limit | 초당 30메시지 (충분) |

### 7.2 메시지 포맷 (예시)

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

### 7.3 확장 구조

```java
public interface MessagingProviderITF {
    void send(String message, Map<String, String> credentials);
    Provider getProvider();
}
```

새 메신저 추가 = 이 인터페이스 구현 + Spring Bean 등록. 기존 코드 수정 불필요.

---

## 8. Spring Batch 설계

### 8.1 Job: `numberGenerationJob`

| 항목 | 내용 |
|---|---|
| 트리거 | `@Scheduled(cron = "0 0 17 ? * SAT")` |
| 방식 | Tasklet (작업량이 적으므로 Reader/Writer 불필요) |
| 대상 | `lotto_schedule` 테이블에서 `enabled = true`인 레코드 |

### 8.2 Tasklet 실행 흐름

```
1. 활성화된 스케줄 전체 조회
2. 각 스케줄에 대해:
   a. 전략에 따라 번호 생성 (setCount만큼)
   b. lotto_generated_number에 저장
   c. 연결된 messaging_account로 메시지 전송
   d. lotto_message_history에 전송 결과 기록
   e. schedule.lastRunAt 업데이트
3. Job 완료 로그
```

### 8.3 Spring Batch 메타 테이블
- `spring.batch.jdbc.initialize-schema: always` 설정으로 자동 생성
- `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등

---

## 9. PWA 설계

### 9.1 기술
- `@vite-plugin-pwa/nuxt` (Nuxt 4 호환)

### 9.2 Manifest

| 항목 | 값 |
|---|---|
| name | 머니뚝 (moneyttuk) |
| short_name | 머니뚝 |
| display | standalone |
| start_url | /lotto |
| theme_color | #0f172a |
| background_color | #0f172a |
| orientation | portrait |

### 9.3 아이콘
- `public/icons/icon-192x192.png`
- `public/icons/icon-512x512.png` (maskable 포함)

### 9.4 Service Worker
- `registerType: autoUpdate`
- 정적 자산 캐싱 (`*.js, *.css, *.html, *.png`)
- `navigateFallback: /lotto`

---

## 10. 아키텍처

### 10.1 전체 구조

```
[lotto-app (Nuxt 4, port 3001)]
    |
    | REST API (JSON/HTTP)
    v
[Gateway (port 8080)]
    |
    | /lotto/** → localhost:8082
    v
[lotto-server (Spring Boot, port 8082)]
    |
    |-- web/         REST 컨트롤러
    |-- application/ 유스케이스
    |-- domain/      순수 도메인 (생성기, 엔티티, VO)
    |-- infra/       JPA, 텔레그램 어댑터
    |-- batch/       Spring Batch
    |-- global/      Security, JWT, 예외
    |
    ├──→ [MySQL localhost:3306/member]
    └──→ [Telegram Bot API]
```

### 10.2 백엔드 헥사고날 구조

auth-server의 패턴을 그대로 따름. 상세 구조는 핸드오프 문서 참조.

---

## 11. 구현 순서

| Phase | 작업 | 영역 | 의존성 |
|---|---|---|---|
| 1 | PWA 적용 | Frontend | 없음 |
| 2 | lotto-server 스캐폴드 + 도메인 | Backend | 없음 |
| 3 | REST API + 게이트웨이 연동 | Backend | Phase 2 |
| 4 | Spring Batch 스케줄러 | Backend | Phase 3 |
| 5 | 텔레그램 메신저 연동 | Backend | Phase 4 |
| 6 | 프론트엔드 설정 UI (선택) | Frontend | Phase 3 |

> Phase 1과 Phase 2는 독립적이므로 병렬 진행 가능.

---

## 12. 검증 방법

| 대상 | 방법 |
|---|---|
| PWA | Chrome DevTools > Application > Manifest, 모바일 "홈 화면 추가" |
| API | curl/Postman으로 CRUD 테스트 |
| Batch | `JobLauncher.run()` 수동 트리거 → DB 확인 |
| 메신저 | `/messaging-accounts/{id}/test` → 텔레그램 수신 확인 |
| E2E | 스케줄 생성 → 배치 실행 → 번호 생성 → 텔레그램 수신 |
