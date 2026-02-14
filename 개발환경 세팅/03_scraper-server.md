# 🕷 scraper-server 세팅 가이드

데이터 수집 및 크롤링을 담당하는 NestJS 기반 서버입니다.

## 1. 프레임워크 및 버전
- **Framework**: NestJS (v11 이상)
- **Language**: TypeScript
- **Runtime**: Node.js v20+

## 2. 사전 준비 사항
1. **MongoDB 설치 및 실행**: 27017 포트 확인
2. **환경 변수 설정**: `.env` 파일을 생성하거나 시스템 환경 변수에 `MONGODB_URI`를 등록해야 합니다.
   - 예: `MONGODB_URI=mongodb://localhost:27017/scraper`

## 3. 설치 및 실행 방법

```bash
# 1. 프로젝트 폴더 이동
cd scraper-server

# 2. 의존성 패키지 설치
npm install

# 3. Playwright 브라우저 설치 (크롤링을 위해 필수)
npx playwright install

# 4. 서버 실행 (개발 모드)
npm run start:dev
```

## 4. 참고 사항
- 서버는 기본적으로 `4000` 포트에서 동작합니다.
- Playwright를 사용하므로 첫 실행 전 반드시 `npx playwright install` 명령어를 수행해야 합니다.
