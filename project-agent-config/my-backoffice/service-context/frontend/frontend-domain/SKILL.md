# Frontend Domain Skill

my-bo-frontend/frontend 작업 시 Nuxt/Vue 패턴을 강제하는 스킬.

## 프로젝트 개요

**Vue 3 + Nuxt 3 + Composition API** 기반 프론트엔드 아키텍처.
**유지보수성**, **확장성**, **재사용성**을 최우선 목표로 하며, 백엔드(Spring Boot)와의 협업을 고려하여 설계.

## 디렉토리 구조

```text
my-nuxt-app/
├── components/          # 컴포넌트 (재사용성 중심 분리)
│   ├── base/            # [UI] 비즈니스 로직이 없는 순수 디자인 컴포넌트 (Button, Input)
│   ├── domain/          # [Biz] 특정 기능에 종속된 복합 컴포넌트 (LoginForm, ProductCard)
│   └── layout/          # 레이아웃 전용 컴포넌트 (Header, Sidebar, Footer)
├── composables/         # 비즈니스 로직 및 API 호출 (Service/Repository 역할)
├── layouts/             # 페이지 레이아웃 템플릿
├── middleware/          # 라우팅 가드 (권한 체크)
├── pages/               # 실제 화면 (라우팅 엔트리 포인트)
├── stores/              # 전역 상태 관리 (Pinia)
└── types/               # TypeScript 인터페이스 및 타입 정의
```

## 레이아웃 시스템 (Layout Strategy)

모든 페이지에 공통 네비게이션(Nav)을 적용하되, 사이드바 유무에 따라 레이아웃을 교체.

- **Default Layout (`layouts/default.vue`)**: Top Navbar + Content — 메인 페이지, 소개 페이지 등 넓은 화면
- **Dashboard Layout (`layouts/dashboard.vue`)**: Top Navbar + Side Navbar + Content — 관리자 페이지, 마이 페이지 등

## 인증 및 권한 관리

페이지 단위 `if`문 대신 Nuxt **Middleware**를 통해 진입 시점에 권한 제어.

- `middleware/auth.ts` 작성
- `definePageMeta`로 적용:
  ```typescript
  definePageMeta({
    middleware: ['auth'],
    layout: 'dashboard'
  });
  ```

## 컴포넌트 설계 원칙

| 구분 | 경로 | 역할 | 특징 |
|------|------|------|------|
| **Base** | `components/base` | 순수 UI 렌더링 | 상태 없음, API 호출 금지, Props로만 데이터 수신 |
| **Domain** | `components/domain` | 기능 단위 구현 | Base 컴포넌트 조합, API 호출 수행, 비즈니스 로직 포함 |
| **Layout** | `components/layout` | 뼈대 구성 | 헤더, 푸터, 사이드바 등 레이아웃 전용 |

## Composables 전략

Vue 컴포넌트(`script setup`) 내부에는 UI 제어 로직만 남기고, 데이터 통신 및 복잡한 계산 로직은 `composables`로 추출 (Java Service 계층과 유사).

- 예시: `useApi.ts`, `useAuth.ts`
- Vue의 Auto-import 기능 활용하여 별도 import 없이 사용

## 기술 스택

- **Core**: Nuxt 3, Vue 3 (Composition API)
- **Language**: TypeScript
- **State Management**: Pinia
- **Styling**: Tailwind CSS
- **Package Manager**: npm

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
