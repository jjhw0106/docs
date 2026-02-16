# 🚀 My-Bo 프로젝트 개발환경 세팅 가이드

이 문서는 My-Bo 프로젝트의 전체 개발 환경을 구축하기 위한 메인 메뉴얼입니다. 처음 프로젝트에 합류하셨다면 아래 순서대로 세팅을 진행해 주세요.

## 📋 세팅 순서

1. [**공통 환경 세팅**](./00_공통_세팅.md): JDK, Node, DB 등 기초 도구 설치
2. [**Auth-Server 세팅**](./01_auth-server.md): 인증 서버 구축 (MySQL)
3. [**Gateway 세팅**](./02_gateway.md): API 게이트웨이 설정
4. [**Scraper-Server 세팅**](./03_scraper-server.md): 크롤러 서버 구축 (MongoDB, Playwright)
5. [**Frontend 세팅**](./04_frontend.md): UI 프로젝트 실행

---

## 💡 팁
- 모든 서버를 실행할 때는 **DB -> Backend -> Frontend** 순서로 실행하는 것을 권장합니다.
- 각 문서에는 상세한 에러 해결 방법이나 주의사항이 포함되어 있습니다.
- 세팅 중 문제가 발생하면 `개발일지` 폴더를 참고하거나 담당자에게 문의하세요.
