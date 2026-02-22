# Frontend — Domain Context

Nuxt 4.2.2 + Vue 3.5 + Tailwind CSS 3.4 기반 백오피스 프론트엔드.

## 핵심 구조

```
my-bo-frontend/frontend/
├── pages/
│   ├── login.vue, signup.vue, index.vue
│   ├── settings.vue, stocks.vue
│   ├── lotto/          ← index, dashboard, history, purchase
│   └── my-career/      ← index, [platform]
├── composables/
│   ├── useAuth.ts      ← 로그인/로그아웃, 쿠키 기반 유저 상태
│   ├── useScraper.ts   ← 스크래핑 트리거, 이력 조회
│   ├── useExcel.ts     ← XLSX 다운로드
│   ├── usePagePermissions.ts ← 보호 경로 목록
│   └── lotto/useLotto.ts ← 로또 CRUD
├── components/
│   ├── domain/         ← 비즈니스 컴포넌트
│   ├── layout/         ← AppHeader, AppSidebar, HistoryFilter
│   └── lotto/          ← 로또 전용 컴포넌트
└── layouts/
    ├── default.vue     ← 공개 페이지
    └── dashboard.vue   ← 인증 필요 페이지 (헤더+사이드바)
```

## 핵심 패턴

### 공유 상태 (컴포저블 외부 정의)
```typescript
// composables/useScraper.ts
const isScraping = ref(false); // ← 함수 밖에 정의 (모듈 레벨)

export function useScraper() {
  return { isScraping };
}
```

### API 호출
```typescript
const config = useRuntimeConfig();
await $fetch(`${config.public.apiBase}/endpoint`, {
  method: 'POST',
  credentials: 'include',
  body: { ... }
});
```

### 레이아웃 적용
```vue
<script setup>
definePageMeta({ layout: 'dashboard' }); // 인증 필요 페이지
</script>
```

## 라우팅 & 인증
- 전역 가드: `middleware/auth.global.ts`
- 보호 경로: `usePagePermissions.ts`에서 관리
- 유저 정보: `useCookie('user')`
- 앱 유저 ID: `localStorage('last_user_id')`

## 도메인 스킬
`/frontend-domain` 스킬 사용 시 이 컨텍스트 자동 로드.
