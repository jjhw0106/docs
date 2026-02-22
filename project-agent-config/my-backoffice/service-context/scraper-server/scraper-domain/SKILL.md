# Scraper Domain Skill

scraper-server 작업 시 Playwright 및 데이터 패턴을 강제하는 스킬.

## 새 플랫폼 추가 체크리스트

1. **플랫폼별 private 메서드** 생성 (스크래핑 로직 분리)
2. **로그인 처리**: 자동/수동 분기
3. **Playwright 패턴**:
   - `finally` 블록에서 반드시 browser.close()
   - headless: false (디버깅 용이)
4. **데이터 저장**:
   - 저장 전 `deleteMany({ appUserId, platform })` 실행
   - 스크래핑 로직과 저장 로직 분리
5. **페이지네이션**: 플랫폼별 최대 페이지 수 설정

## 금지 패턴
- 브라우저를 finally 없이 열기 (❌ — 리소스 누수)
- 스크래핑과 저장 로직 혼재 (❌ — 관심사 분리)
- 전체 기존 데이터를 남긴 채 append (❌ — deleteMany 필수)

## 텍스트 추출 패턴
- Wanted: `page.textContent()` + 정규식 파싱
- JobKorea: `page.$$(selector)` + CSS 셀렉터 기반
