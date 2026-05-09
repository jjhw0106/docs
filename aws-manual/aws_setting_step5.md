# AWS 배포 매뉴얼 - 5단계: EC2 배포 실행

## 개요

EC2에 레포를 clone하고, `.env` 파일을 작성한 뒤 Docker로 auth-server를 실행한다.

---

## 1. EC2 SSH 접속

```bash
ssh -i ~/.ssh/moneyttuk.com.pem ubuntu@43.202.153.119
```

> 접속 안 되면 현재 IP가 Security Group에서 막힌 것.
> AWS 콘솔 → EC2 → 보안 그룹 → `launch-wizard-1` → SSH 인바운드 규칙 → **내 IP**로 업데이트.

---

## 2. git 설치 및 레포 clone

```bash
# git 설치 확인
git --version

# 없으면 설치
sudo apt update && sudo apt install -y git

# 레포 clone (홈 디렉토리에 서버별로 관리)
# /home/ubuntu/auth-server/, /home/ubuntu/gateway/, ... 구조
git clone https://github.com/jjhw0106/auth-server.git
cd auth-server
```

---

## 3. .env 파일 작성

`.env.example`을 복사해서 실제 값을 채워 넣는다.

```bash
cp .env.example .env
nano .env
```

아래 항목을 실제 값으로 채운다:

```
SPRING_PROFILES_ACTIVE=dev

DB_URL=jdbc:mysql://<RDS_ENDPOINT>:3306/authdb?useSSL=false&serverTimezone=Asia/Seoul
DB_USERNAME=authuser
DB_PASSWORD=실제_DB_비밀번호

JWT_SECRET=실제_JWT_시크릿_키
```

> - RDS 엔드포인트: AWS 콘솔 → RDS → `auth-server-db` → 연결 & 보안 탭에서 확인
> - auth-server-db.cxoiks04o8uv.ap-northeast-2.rds.amazonaws.com
> - nano 저장: `Ctrl+O` → Enter → `Ctrl+X`

---

## 4. Docker Compose로 실행

```bash
docker compose up -d --build
```

**옵션 설명:**

| 옵션 | 의미 |
|------|------|
| `up` | 컨테이너 시작 |
| `-d` | 백그라운드 실행 (detached) |
| `--build` | 이미지를 새로 빌드 후 실행 |

> 첫 빌드는 Gradle 의존성 다운로드로 5~10분 소요.

---

## 5. 로그 확인

```bash
# 실시간 로그 확인
docker compose logs -f

# 컨테이너 상태 확인
docker compose ps
```

정상 기동 시 로그 마지막에 아래와 같이 출력됨:
```
Started AuthServerApplication in X.XXX seconds
```

---

## 6. 헬스체크

```bash
# EC2 내부에서 테스트
curl localhost:9001/actuator/health
```

정상 응답:
```json
{"status":"UP"}
```

---

## 7. 외부 접속 테스트

로컬 PC 브라우저 또는 터미널에서:

```bash
curl http://43.202.153.119:9001/actuator/health
```

---

## 트러블슈팅

### ❌ docker compose 명령어 없음

```bash
# Docker Compose V2 확인
docker compose version

# 없으면 설치
sudo apt install -y docker-compose-plugin
```

### ❌ 빌드 실패 (Gradle 메모리 부족)

t3.micro는 메모리가 1GB라 Gradle 빌드 시 OOM이 날 수 있음.

```bash
# Swap 메모리 추가
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 재시도
docker compose up -d --build
```

### ❌ 컨테이너 실행 후 바로 종료됨

```bash
# 종료된 컨테이너 로그 확인
docker compose logs auth-server
```

DB 연결 실패인 경우 → `.env`의 `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` 확인.

### ❌ 외부에서 9001 포트 접속 안 됨

EC2 Security Group 인바운드에 9001 포트가 열려 있는지 확인.
AWS 콘솔 → EC2 → 보안 그룹 → `launch-wizard-1` → 인바운드 규칙.

---

## 완료 체크리스트

- [x] EC2 SSH 접속 성공
- [x] git clone 완료
- [x] `.env` 파일 작성 완료
- [x] `docker compose up -d --build` 성공
- [x] `curl localhost:9001/actuator/health` → `{"status":"UP"}`
- [x] 외부 IP(43.202.153.119:9001)에서 접속 성공
