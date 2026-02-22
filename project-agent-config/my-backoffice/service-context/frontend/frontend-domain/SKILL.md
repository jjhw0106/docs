# Frontend Domain Skill

my-bo-frontend/frontend 작업 시 Nuxt/Vue 패턴을 강제하는 스킬.

## 새 페이지 추가 체크리스트

1. `pages/` 하위 적절한 디렉토리에 `.vue` 파일 생성
2. 인증 필요 페이지: `definePageMeta({ layout: 'dashboard' })`
3. 공개 페이지: `definePageMeta({ layout: 'default' })`
4. 보호 라우트 추가 시: `usePagePermissions.ts` 업데이트

## 새 컴포저블 추가 체크리스트

1. 공유 상태는 **함수 외부(모듈 레벨)**에 `ref`/`reactive`로 정의
2. TypeScript로 반환 타입 명시
3. 정리 필요한 사이드이펙트: `onUnmounted`에서 처리
4. `composables/` 디렉토리에 저장 (Nuxt 자동 임포트)

## 새 컴포넌트 추가 체크리스트

1. 비즈니스 컴포넌트: `components/domain/`
2. 레이아웃 컴포넌트: `components/layout/`
3. 도메인별 컴포넌트: `components/{domain}/` (예: `components/lotto/`)
4. `<script setup lang="ts">` 사용
5. Props: `defineProps<{...}>()` 타입 정의
6. Emits: `defineEmits<{...}>()` 타입 정의

## 금지 패턴
- Options API 사용 (❌ — Composition API만)
- 공유 상태를 컴포저블 함수 내부에 정의 (❌ — 모듈 레벨로)
- 커스텀 CSS 남용 (❌ — Tailwind 유틸리티 클래스 우선)
- 템플릿에 복잡한 로직 직접 작성 (❌ — computed 사용)

## Tailwind 가이드
- 반응형: `sm:`, `md:`, `lg:` 접두사
- 색상: 기존 컴포넌트 색상 체계 참조
- 레이아웃: flexbox/grid 유틸리티 활용
