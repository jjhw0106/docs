# Lotto Service - Frontend Handoff Document

> 작성일: 2026-02-17 | 최종 수정: 2026-02-17

---

## 1. 서비스 개요

로또 번호 추천 기능을 제공하는 **순수 프론트엔드 서비스**. 백엔드 없음.

---

## 2. 기술 스택

| 항목 | 내용 |
|---|---|
| 프레임워크 | Nuxt 4.2.2 + Vue 3.5 |
| 스타일 | Tailwind CSS |
| 백엔드 연동 | **없음** |
| 레이아웃 | `dashboard` 레이아웃 사용 |

---

## 3. 파일 구조

```
pages/lotto/
└── index.vue              # 번호 추천 페이지 (유일한 페이지)

components/lotto/
├── LottoBall.vue           # 사용 중 - 번호 볼 컴포넌트
├── WinningNumberCard.vue   # 미사용
├── PurchaseForm.vue        # 미사용
├── PurchaseHistoryTable.vue # 미사용
├── PurchaseSummaryCards.vue # 미사용
├── HistoryFilter.vue       # 미사용
├── LottoNumberInput.vue    # 미사용
└── RankBadge.vue           # 미사용

composables/lotto/
└── useLotto.ts             # 미사용 (번호 추천 로직은 페이지에 직접 구현)
```

---

## 4. 사이드바 메뉴

`layouts/dashboard.vue`의 `sidebarMenuItems`:

```typescript
{
  name: 'Lotto',
  path: '/lotto',
  icon: '🎰',
  children: [
    { name: '번호 추천', path: '/lotto' },
  ]
}
```

---

## 5. 번호 추천 로직 (pages/lotto/index.vue)

백엔드 없이 페이지 내 함수로 직접 구현.

**전략별 로직:**

| 전략 | 방식 |
|---|---|
| 완전 랜덤 | 1~45 셔플 후 6개 추출 |
| 균형 배분 | 5개 구간에서 각 1개 + 전체에서 추가 1개 |
| 홀짝 균형 | 홀수 pool 3개 + 짝수 pool 3개 |

**주요 기능:**
- 게임 수: 1~5 선택
- 전체 재생성 / 게임별 개별 재생성
- 전체 클립보드 복사
- 페이지 진입 시 자동 생성 (기본값: 완전 랜덤, 5게임)

---

## 6. LottoBall 컴포넌트 스펙

**Props:**
| Prop | Type | Default | 설명 |
|---|---|---|---|
| `number` | `number` | 필수 | 표시할 번호 (1~45) |
| `size` | `'sm'│'md'│'lg'` | `'md'` | 볼 크기 |
| `isBonus` | `boolean` | `false` | 보너스 번호 여부 |
| `isMatched` | `boolean` | `false` | 매칭 번호 강조 여부 |

**번호별 색상:**
| 범위 | 클래스 |
|---|---|
| 1~10 | `bg-yellow-500 text-yellow-950` |
| 11~20 | `bg-blue-500 text-white` |
| 21~30 | `bg-red-500 text-white` |
| 31~40 | `bg-gray-500 text-white` |
| 41~45 | `bg-green-500 text-white` |
