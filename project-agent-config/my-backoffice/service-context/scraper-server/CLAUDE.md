# Scraper Server — Domain Context

NestJS 11 + TypeScript 기반 스크래핑 서비스.

## 기술 스택
- NestJS 11 with TypeScript (strict mode)
- Playwright 1.57 for browser automation
- MongoDB with Mongoose 9.1
- Port: 4000
- CORS: 개발 환경에서 전체 허용

## 핵심 패턴

### Playwright 필수 패턴
```typescript
let browser;
try {
  browser = await chromium.launch({ headless: false });
  // 스크래핑 로직
} finally {
  if (browser) await browser.close(); // 반드시 finally에서 close
}
```

### 데이터 저장 패턴
```typescript
// 저장 전 기존 데이터 삭제 (upsert 아님)
await this.model.deleteMany({ appUserId, platform });
await this.model.insertMany(newData);
```

### MongoDB 스키마
```
appUserId, platformUserId, platform, company, position, status, appliedAt
```

## 지원 플랫폼
- **Wanted**: 페이지네이션 최대 20페이지, 텍스트 패턴 기반 추출
- **JobKorea**: 페이지네이션 최대 5페이지, CSS 셀렉터 기반 추출

## 로그인 처리
- 자격증명 있을 시: 자동 로그인
- 없을 시: 수동 로그인 대기 (2-3분 타임아웃)

## API 엔드포인트 (직접 접근, 게이트웨이 거치지 않음)
- `POST /scraper/:platform` — 스크래핑 시작
- `GET /scraper/history/:userId` — 이력 조회

## 도메인 스킬
`/scraper-domain` 스킬 사용 시 이 컨텍스트 자동 로드.
