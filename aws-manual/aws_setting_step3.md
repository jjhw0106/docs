# AWS RDS 세팅 매뉴얼 - 3단계

## 개요

RDS MySQL 생성, Security Group 설정, EC2에서 연결 테스트까지.

---

## 1. DB 서브넷 그룹 생성

> RDS는 VPC 내 서브넷 그룹이 필요. 가용 영역 2개 이상 포함해야 함.

1. AWS 콘솔 검색창 → **RDS**
2. 왼쪽 메뉴 **서브넷 그룹** → **DB 서브넷 그룹 생성**
3. 아래 설정:

| 항목 | 값 |
|------|-----|
| 이름 | `auth-server-subnet-group` |
| 설명 | auth-server RDS subnet group |
| VPC | 기본 VPC 선택 |

4. **가용 영역**: `ap-northeast-2a`, `ap-northeast-2c` 선택
5. **서브넷**: 각 AZ의 서브넷 선택 (기본 VPC면 자동으로 뜸)
6. **생성**

---

## 2. RDS Security Group 생성

> EC2에서만 3306 접근 허용. 인터넷에서 직접 접근 차단.

1. EC2 콘솔 → 왼쪽 메뉴 **보안 그룹** → **보안 그룹 생성**
2. 아래 설정:

| 항목 | 값 |
|------|-----|
| 이름 | `rds-sg` |
| 설명 | Allow MySQL from EC2 only |
| VPC | 기본 VPC |

3. **인바운드 규칙 추가**:

| 유형 | 포트 | 소스 |
|------|------|------|
| MySQL/Aurora | 3306 | EC2의 Security Group ID (검색으로 선택) |

> 소스에 IP 대신 EC2 Security Group ID를 입력하면 EC2만 접근 가능.  
> EC2 Security Group ID 확인: EC2 콘솔 → 인스턴스 → 보안 탭에서 확인.

4. **아웃바운드**: 기본값(전체 허용) 유지
5. **생성**

---

## 3. RDS MySQL 인스턴스 생성

1. RDS 콘솔 → **데이터베이스 생성**
2. 아래 설정으로 구성:

| 항목 | 값 |
|------|-----|
| 생성 방식 | 표준 생성 |
| 엔진 | MySQL |
| 엔진 버전 | 8.0.x (최신 8.0) |
| 템플릿 | **프리 티어** (t3.micro 자동 선택) |
| DB 인스턴스 식별자 | `auth-server-db` |
| 마스터 사용자 이름 | `admin` |
| 마스터 암호 | 강력한 암호로 설정 (기록 필수) |

3. **인스턴스 구성**: `db.t3.micro` 확인

4. **스토리지**:

| 항목 | 값 |
|------|-----|
| 스토리지 유형 | gp2 |
| 할당 스토리지 | 20 GB |
| 스토리지 자동 조정 | **비활성화** (비용 관리) |

5. **연결**:

| 항목 | 값 |
|------|-----|
| VPC | 기본 VPC |
| DB 서브넷 그룹 | `auth-server-subnet-group` |
| 퍼블릭 액세스 | **아니요** (외부 직접 접근 차단) |
| VPC 보안 그룹 | **기존 항목 선택** → `rds-sg` |
| 가용 영역 | ap-northeast-2a |

6. **추가 구성**:

| 항목 | 값 |
|------|-----|
| 초기 데이터베이스 이름 | `authdb` |
| 자동 백업 | 비활성화 (비용 절감, 학습 목적) |
| 마이너 버전 자동 업그레이드 | 비활성화 |

7. **데이터베이스 생성** 클릭

> 생성까지 약 5~10분 소요.

---

## 4. RDS 엔드포인트 확인

1. RDS 콘솔 → **데이터베이스** → `auth-server-db` 클릭
2. **연결 & 보안** 탭 → **엔드포인트** 복사

```
예시: auth-server-db.xxxxxxxx.ap-northeast-2.rds.amazonaws.com
```

> 이 엔드포인트가 `DB_URL`에 들어갈 값.

---

## 5. EC2에서 RDS 연결 테스트

EC2 SSH 접속 후:

```bash
ssh -i ~/.ssh/moneyttuk.com.pem ubuntu@43.202.153.119
```

MySQL 클라이언트 설치:

```bash
sudo apt install -y mysql-client
```

RDS 연결 테스트:

```bash
mysql -h [RDS 엔드포인트] -u admin -p
```

비밀번호 입력 후 아래처럼 뜨면 성공:

```
Welcome to the MySQL monitor. ...
mysql>
```

DB 확인:

```sql
SHOW DATABASES;
USE authdb;
```

---

## 6. 애플리케이션용 DB 유저 생성 (선택)

> admin 계정을 앱에 직접 사용하는 것보다 전용 유저 권한 분리 권장.

```sql
CREATE USER 'authuser'@'%' IDENTIFIED BY '강력한비밀번호';
GRANT ALL PRIVILEGES ON authdb.* TO 'authuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

---

## 트러블슈팅

### ❌ SSH 접속 안 됨 (Connection timed out)

**원인**: EC2 Security Group의 SSH(22) 인바운드 규칙에 등록된 IP가 현재 IP와 다름

AWS에서 SSH 접속을 허용할 IP를 Security Group 인바운드 규칙에 명시해야 한다.
집/카페 등 와이파이는 유동 IP라서 네트워크가 바뀔 때마다 IP가 달라짐.
EC2의 IP(Elastic IP)는 고정이지만, **접속하는 내 IP가 바뀌는 것**이 문제.

**해결**: EC2 콘솔 → 보안 그룹 → `launch-wizard-1` → 인바운드 규칙 편집 → SSH 소스를 **내 IP**로 다시 선택 → 저장

> `launch-wizard-1`은 별다른 의미 없이 EC2 생성 시 콘솔이 자동으로 붙인 이름. EC2에 붙어있는 인바운드/아웃바운드 정책 묶음이다.

---

### ❌ RDS 연결 안 됨 (ERROR 2003: Can't connect, 110)

**원인**: RDS에 붙은 Security Group에 3306 인바운드 규칙이 없었음

Security Group은 리소스에 붙이는 **인바운드/아웃바운드 정책 묶음**이고, 이름 자체는 아무 의미 없다.
`rds-sg`라는 이름도 직접 지은 것으로, "RDS용으로 만든 SG"라는 의미일 뿐이다.

RDS 생성 시 보안 그룹을 명시하지 않으면 AWS가 `default`를 자동으로 붙이는데,
`default`에는 3306 인바운드 규칙이 없으므로 EC2에서 접근이 불가능하다.

`rds-sg`를 써야 하는 이유:
- `rds-sg`에 **"3306 포트, 소스: EC2의 SG(`launch-wizard-1`)"** 인바운드 규칙을 직접 추가했기 때문
- 소스를 IP가 아닌 SG로 지정하면 "그 SG가 붙은 리소스에서 오는 트래픽만 허용"이 되어 EC2만 접근 가능해짐
- EC2는 재시작 시 IP가 바뀔 수 있으므로 IP 대신 SG로 지정하는 것이 안정적

**해결**: RDS → 수정 → 보안 그룹에서 `default` 제거, `rds-sg` 추가 → 즉시 적용

---

### ❌ GRANT 실패 (ERROR 1046: No database selected)

**원인 1**: `authdb`가 존재하지 않음 (RDS 생성 시 추가 구성에서 초기 DB 이름 미입력)

**원인 2**: GRANT 문법에서 `.*` 누락 (`authdb`가 아닌 `authdb.*` 로 써야 함)

**해결**:
```sql
CREATE DATABASE authdb;
GRANT ALL PRIVILEGES ON authdb.* TO 'username'@'%';
FLUSH PRIVILEGES;
```

---

## 완료 체크리스트

- [x] DB 서브넷 그룹 생성
- [x] RDS Security Group 생성 (EC2 SG에서만 3306 허용)
- [x] RDS MySQL t4g.micro 생성 (프리 티어)
- [x] RDS에 `rds-sg` 연결 확인
- [x] RDS 엔드포인트 확인 및 기록
- [x] EC2에서 RDS 연결 테스트 성공
- [x] 애플리케이션용 DB 유저 생성
- [x] authdb 생성
