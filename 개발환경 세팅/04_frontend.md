# 💻 my-bo-frontend 세팅 가이드

Nuxt 3 기반의 사용자 인터페이스 프론트엔드 프로젝트입니다.

## 1. 프레임워크 및 버전
- **Framework**: Nuxt 3 (v4.2.2)
- **Language**: TypeScript / Vue 3
- **CSS**: Tailwind CSS

## 2. 사전 준비 사항
- Node.js v20 이상이 설치되어 있어야 합니다.

## 3. 설치 및 실행 방법

```bash
# 1. 프로젝트 폴더 이동 (실제 소스코드가 있는 곳)
cd my-bo-frontend/frontend

# 2. 의존성 패키지 설치
npm install

# 3. 개발 서버 실행
npm run dev
```

## 4. 참고 사항
- 실행 후 `http://localhost:3000` 접속하여 확인 가능합니다.
- 백엔드와 통신 시 `gateway`(`localhost:8080`)를 통하도록 설정되어 있는지 확인하세요.
