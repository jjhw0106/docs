# PRD Writer Skill

PRD(Product Requirements Document)를 작성하는 표준 스킬.

## 출력 위치
`docs/prd/prd-{feature-name}.md`

## PRD 작성 단계

### 1. 컨텍스트 파악
- 요청된 기능/서비스의 목적 확인
- 관련 서비스 및 기존 구조 파악
- 사용자 스토리 정의

### 2. PRD 문서 구조

```markdown
# PRD: {기능명}

> 작성일: {date} | 버전: v{N} | 상태: DRAFT | 작성자: planner

---

## 1. 개요
### 1.1 목적
### 1.2 배경 및 문제 정의
### 1.3 범위

## 2. 사용자 스토리
| As a | I want to | So that |
|------|-----------|---------|

## 3. 기능 요구사항
### 3.1 핵심 기능
### 3.2 비기능 요구사항 (성능, 보안, 가용성)

## 4. 기술 스펙
### 4.1 영향받는 서비스
### 4.2 API 엔드포인트
### 4.3 데이터 모델 변경

## 5. UI/UX 요구사항
### 5.1 화면 목록
### 5.2 주요 플로우

## 6. 제약사항 및 가정
## 7. 완료 기준 (Definition of Done)
## 8. 미결 사항 (Open Questions)
```

### 3. 검토 및 저장
- 완료 기준이 검증 가능한지 확인
- Open Questions에 불명확한 항목 기록
- `docs/prd/prd-{feature-name}.md`로 저장

## 네이밍 규칙
- 파일명: `prd-{kebab-case-feature-name}.md`
- 예: `prd-resume-service.md`, `prd-lotto-upgrade.md`
