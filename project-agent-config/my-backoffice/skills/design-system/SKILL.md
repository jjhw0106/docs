# Design System Skill

my-bo-frontend 프로젝트의 디자인 시스템 & UI 스타일 가이드를 정의하는 스킬.

## 핵심 테마 아이덴티티

- **스타일 명칭**: 프리미엄 다크 글래스모피즘 (Premium Dark Glassmorphism)
- **기본 모드**: 다크 모드 (CSS 변수가 기본적으로 다크 테마 값으로 설정됨)
- **배경**: 고정된 선형 그라디언트 배경 사용
  - `linear-gradient(135deg, #0f172a 0%, #1e293b 50%, #0f172a 100%)`

## 컬러 팔레트 (Tailwind CSS 변수, HSL 포맷)

### 기본 색상 (Primitive Colors)

| 토큰 | HSL 값 | 설명 |
|------|---------|------|
| Background | `222.2 84% 4.9%` | 딥 블루/블랙 |
| Foreground | `210 40% 98%` | 오프 화이트 |
| Primary | `217.2 91.2% 59.8%` | 브라이트 블루 |
| Primary 텍스트 | `222.2 47.4% 11.2%` | 어두운 파란색 |
| Secondary | `217.2 32.6% 17.5%` | 뮤트 블루-그레이 |
| Muted | `217.2 32.6% 17.5%` | 비활성/배경 |
| Accent | `217.2 32.6% 17.5%` | 강조 |
| Destructive | `0 62.8% 30.6%` | 레드 |
| Border | `217.2 32.6% 17.5%` | 테두리 |
| Input | `217.2 32.6% 17.5%` | 입력창 |
| Ring | `224.3 76.3% 48%` | 포커스 링 (파란색) |

### 테두리 둥글기 (Border Radius)

- **Default**: `0.5rem` (8px)
- **lg**: `0.5rem`
- **md**: `calc(0.5rem - 2px)`
- **sm**: `calc(0.5rem - 4px)`

## UI 효과 및 컴포넌트

### 글래스모피즘 (`.glass-effect`)

카드, 모달, 컨테이너 등에 사용하여 반투명한 유리를 덧댄 듯한 효과를 줍니다.

```css
.glass-effect {
  background: rgba(30, 41, 59, 0.7);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(148, 163, 184, 0.1);
}
```

### 버튼 (Buttons)

- **Hover**: 살짝 위로 떠오르며(`translateY(-1px)`), 그림자가 생김(`box-shadow`)
- **Active**: 원래 위치로 돌아옴(`translateY(0)`)
- **전환 효과**: 모든 변화는 0.2초(200ms) 동안 부드럽게 전환

### 입력창 (Inputs)

- **Focus**: 파란색 빛무리가 생김 (`box-shadow`로 glow 효과 구현)

## 애니메이션

### 페이드 인 (`.animate-fade-in`)

- **동작**: 아래에서 위로 살짝 올라오며 투명도 0 → 1
- **지속 시간**: 0.5초 (ease-out)

### 쉬머 효과 (`.animate-shimmer`)

- **동작**: 왼쪽에서 오른쪽으로 빛이 지나가는 듯한 로딩 효과
- **용도**: 스켈레톤 UI, 로딩 상태
- **지속 시간**: 2초 (무한 반복)

## 글로벌 CSS 동작

- **부드러운 전환**: 모든 요소(`*`)에 색상, 배경, 위치 등의 변화가 생길 때 0.2초간 부드럽게 전환
- **폰트 설정**: Inter 폰트 최적화 (`rlig`, `calt` 기능 활성화)

## Nuxt 3 적용 가이드

1. **Tailwind CSS** 설치 및 설정
2. `tailwind.config.ts`에 위의 CSS 변수(`colors`, `borderRadius` 등)를 `extend`로 추가
3. `assets/css/main.css`를 생성하여 `:root` 변수와 `.glass-effect` 같은 유틸리티 클래스 복사
4. `layouts/default.vue`에 전체 배경 그라디언트 적용
