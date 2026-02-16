# 📅 2026년 1월 25일 To-Do

---

### FE (프론트엔드)
- [ ] 로컬 스토리지 암호화 적용
  - `crypto-js` 등 라이브러리 활용
  - `useCrypto.ts` 컴포저블 생성
- [v] 전역 인증 상태 관리 개선 (임시)
  - ~~`useAuth.ts` 및 `LoginForm`에서 앱 유저 ID(`last_user_id`) 관리 로직 구현~~
  - ~~모든 조회 및 스크래핑 요청 시 앱 유저 ID를 기준으로 통신하도록 리팩토링 완료~~
- [ ] (GEMINI 제안) 스크래핑/조회 시 로딩 및 에러 상태 UI 개선
  - 로딩 스피너 및 에러 메시지 컴포넌트 추가
- [v] 레이아웃 시스템 통합 및 개선
  - ~~`AppHeader`, `AppSidebar` 컴포넌트 분리~~
  - ~~`default` / `dashboard` 레이아웃 간 네비게이션 공유 및 스타일 최적화 완료~~

---

### BE (백엔드)
- [v] `ApplyHistory` 스키마에 `appUserId` 필드 추가
  - ~~MongoDB NoSQL 특성을 활용하여 마이그레이션 없이 필드 추가 및 적용 완료~~
- [v] 스크래핑 시 `appUserId`를 함께 저장하도록 로직 수정
  - ~~컨트롤러 Body 확장 및 서비스 레이어 매핑 로직 수정 완료~~
- [v] `GET /scraper/history/:appUserId` API 구현
  - ~~앱 유저 ID 기반의 통합 조회 기능 구현 완료~~
- [ ] 잡코리아 팝업 자동 제거 로직 강화
  - `page.evaluate()`를 사용한 DOM 직접 제어
- [v] 스크래핑 데이터 중복 저장 방지
  - ~~`deleteMany({ appUserId, platform })` 쿼리 버그 수정 및 최신 데이터 교체 방식 확립~~
- [ ] (신규) MongoDB Atlas IP Access List 관리 기능 구현
  - 현재 접속 IP 확인 및 Atlas API 연동을 통한 자동 IP 허용 기능

---
