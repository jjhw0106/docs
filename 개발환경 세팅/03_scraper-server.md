# 🕷 scraper-server 세팅 가이드

데이터 수집 및 크롤링을 담당하는 NestJS 기반 서버입니다.

## 1. 프레임워크 및 버전
- **Framework**: NestJS (v11 이상)
- **Language**: TypeScript
- **Runtime**: Node.js v20+

## 2. 사전 준비 사항

### .env 파일 생성 (git에 포함되지 않으므로 직접 생성 필수)

`scraper-server/.env` 파일을 생성하고 아래 내용 작성:

```env
MONGODB_URI=mongodb://[유저]:[패스워드]@[샤드-00-00]:[포트],[샤드-00-01]:[포트],[샤드-00-02]:[포트]/[DB명]?ssl=true&replicaSet=[레플리카셋명]&authSource=admin&retryWrites=true&w=majority
```

> **주의**: `mongodb+srv://` 형식은 일부 네트워크(핫스팟, 특정 공유기)에서 SRV DNS 조회 실패로 연결 안 될 수 있음. 위의 직접 샤드 주소 형식 권장.

### MongoDB Atlas IP 화이트리스트 등록
- Atlas 콘솔 → Network Access → 현재 공인 IP 등록 (네이버 "내 IP 확인" 등으로 확인)
- 네트워크 변경 시(핫스팟 전환 등)마다 IP가 바뀌므로 재등록 필요

## 3. 설치 및 실행 방법

```bash
# 1. 프로젝트 폴더 이동
cd scraper-server

# 2. 의존성 패키지 설치
npm install

# 3. Playwright 브라우저 설치 (chromium만 설치, 첫 실행 시 1회)
npx playwright install chromium

# 4. 서버 실행 (개발 모드)
npm run start:dev
```

## 4. 참고 사항
- 서버는 기본적으로 `4000` 포트에서 동작합니다.
- `npx playwright install`(전체)이 아닌 `npx playwright install chromium`만 설치하면 됩니다. 전체 설치 시 Firefox, WebKit까지 받아 불필요하게 용량을 차지합니다.
