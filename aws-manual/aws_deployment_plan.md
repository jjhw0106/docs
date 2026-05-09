# AWS 배포 계획서 - my-backoffice auth-server

## 목표
- auth-server (Spring Boot) + RDS MySQL을 AWS에 배포
- 프론트(Vercel) → EC2 auth-server → RDS 연결
- AWS 학습 목적

## 구성도

```
[Vercel 프론트 / moneyttuk.com] ─HTTPS→ [EC2 t3.micro 서울 / api.moneyttuk.com / 43.202.153.119]
                                          └─ Docker: nginx (443) + auth-server (9001)
                                                                     ↓
                                          [RDS MySQL t4g.micro (3306, EC2 SG에서만 허용)]
```

## DB 프로파일 전략

| 환경 | 프로파일 | DB |
|------|---------|-----|
| 로컬 개발 | `local` | 로컬 MySQL |
| EC2 운영 | `dev` | RDS MySQL (`authdb`) |

## 현재 상태 (2026-05-09)

| 항목 | 상태 |
|------|------|
| AWS 계정 | 신규 플랜 ($100 크레딧) |
| EC2 | t3.micro Ubuntu 26.04, 서울, Elastic IP: 43.202.153.119 |
| EBS 볼륨 | 20GB (8GB → 확장) |
| 키 페어 | `~/.ssh/moneyttuk.com.pem` |
| Docker | 설치 완료 |
| nginx | Docker 컨테이너로 443/80 서비스 중 |
| HTTPS | Let's Encrypt (api.moneyttuk.com) 발급 완료 |
| RDS | `auth-server-db.cxoiks04o8uv.ap-northeast-2.rds.amazonaws.com` 운영 중 |
| RDS 스키마 | `authdb`에 4개 테이블 적용 완료 |
| Vercel ↔ API 연동 | 회원가입/로그인 E2E 통신 완료 |

## 예상 비용

| 항목 | 월 비용 |
|------|---------|
| EC2 t3.micro | ~$7.6 |
| RDS MySQL t3.micro | ~$12 |
| Elastic IP | 무료 (인스턴스 연결 시) |
| **합계** | **~$20/월** |

---

## Phase 0: AWS 기초 세팅 ✅ 완료
> 매뉴얼: `aws_setting_step1.md`

- [x] 루트 계정 MFA 활성화
- [x] IAM 사용자 생성 (AdministratorAccess)
- [x] IAM 로그인 URL 북마크
- [x] 결제 IAM 액세스 허용
- [x] 예산 알림 설정 ($20)

---

## Phase 1~2: EC2 생성 + Docker 설치 ✅ 완료
> 매뉴얼: `aws_setting_step2.md`

- [x] EC2 t3.micro 생성 (Ubuntu 26.04, 서울)
- [x] 키 페어 생성 및 보관
- [x] Security Group 설정 (22/80/443/9001)
- [x] Elastic IP 할당 (43.202.153.119)
- [x] SSH 접속 성공
- [x] Docker 설치 완료

---

## Phase 3: RDS 생성 ✅ 완료
> 매뉴얼: `aws_setting_step3.md`

- [x] DB 서브넷 그룹 생성
- [x] RDS Security Group 생성 (EC2 SG에서만 3306 허용)
- [x] RDS MySQL t4g.micro 생성 (프리 티어)
- [x] RDS 엔드포인트 확인 및 기록
- [x] EC2에서 RDS 연결 테스트 성공
- [x] 애플리케이션용 DB 유저 생성

---

## Phase 4: 배포 파일 작성 ✅ 완료
> 매뉴얼: `aws_setting_step4.md`

- [x] `docker-compose.yml` — auth-server 단독 구성
- [x] `.env.example` — 환경변수 템플릿
- [x] `.dockerignore` — 빌드 컨텍스트 최적화
- [x] GitHub push

주요 환경변수:
- `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `SPRING_PROFILES_ACTIVE=dev`

---

## Phase 5: EC2 배포 실행 ✅ 완료
> 매뉴얼: `aws_setting_step5.md`

- [x] EC2에 git 설치 → 레포 clone
- [x] EC2에 `.env` 파일 작성 (실제 비밀값)
- [x] `docker compose up -d --build` 실행
- [x] `curl localhost:9001` 헬스체크
- [x] 외부 IP에서 접속 테스트

---

## Phase 6: HTTPS + 도메인 연결 ✅ 완료
> 매뉴얼: `aws_setting_step6.md`

- [x] nginx 컨테이너 추가 (리버스 프록시)
- [x] Let's Encrypt(certbot)으로 SSL 인증서 발급 (api.moneyttuk.com)
- [x] DNS A 레코드 추가 (Vercel DNS → 43.202.153.119)
- [x] HTTPS 접속 확인

---

## Phase 7: Vercel 프론트 연결 ✅ 완료
> 매뉴얼: `aws_setting_step7.md`

- [x] auth-server CORS 정책 정합 (nginx 단일 책임, Spring 비활성화)
- [x] Vercel 환경변수 API URL 설정
- [x] RDS 스키마 적용 (`member`, `refresh_token`, `winning_numbers`, `purchase_games`)
- [x] 회원가입 → DB INSERT → 로그인 E2E 통과

### 미해결 TODO
- 도메인 예외(`IllegalArgumentException` 등) → `@ExceptionHandler`로 400/404 매핑
- 프론트 에러 메시지 UX 개선
- refresh token 만료/재발급 통합 검증

---

## 주의사항

- `.env` 파일은 절대 git 커밋 금지
- JWT_SECRET은 강력한 랜덤값 사용
- EC2 SSH(22) 포트는 내 IP만 허용 (와이파이 IP 바뀌면 Security Group 수정)
- RDS는 중지해도 7일 후 자동 재시작됨 (AWS 정책)
- RDS 중지 중에도 스토리지 비용 발생
- 안 쓸 때 EC2만 중지 가능 (Elastic IP는 연결 해제 시 과금 주의)

---

## 트러블슈팅

배포 과정에서 발생한 모든 이슈와 해결법은 `troubleshooting.md` 참조.

주요 항목:
1. POST /auth/login 403 — Spring Security CORS 차단
2. 재배포 시 Gradle 빌드 실패 (디스크 부족)
3. 로그인 시 `IllegalArgumentException` 500 (RDS 테이블 미생성)
4. EBS 볼륨 확장 후 디스크 미반영 (`growpart` + `resize2fs`)
5. CORS 헤더 중복 (`Access-Control-Allow-Origin` multiple values)
6. WSL Ubuntu에서 Windows MySQL 접속 불가
7. RDS GUI 접속 (DBeaver SSH 터널)
