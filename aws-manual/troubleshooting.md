# 트러블슈팅 기록

> auth-server AWS 배포 과정에서 발생한 이슈와 해결법 정리.
> 새 문제가 생기면 이 문서 하단에 추가한다.

---

## 1. POST /auth/login 403 — Spring Security CORS 차단

### 증상
- nginx까지는 요청 정상 도달
- Spring Security가 CORS 요청을 차단하여 403 반환
- 브라우저 콘솔: `Invalid CORS request`

### 원인
원래 설계는 Gateway가 CORS를 처리하는 구조였음:

```
프론트 → Gateway → auth-server (CORS 꺼도 됨)
```

AWS 배포에서는 Gateway 없이 nginx 사용:

```
프론트 → nginx → auth-server
```

nginx는 HTTP 헤더만 추가할 뿐 Spring Security CORS 필터를 우회하지 못함.
`corsConfigurationSource()`에 `localhost:3000`만 허용되어 있어 운영 도메인 요청을 Spring Security가 먼저 막음.

### 해결
`SecurityConfig.java`의 `corsConfigurationSource()`에 운영 도메인 추가.

```java
configuration.setAllowedOrigins(java.util.List.of(
    "http://localhost:3000",
    "https://moneyttuk.com",
    "https://www.moneyttuk.com"
));
```

### 해결 일자
2026-05-06 — preflight 응답에서 `Access-Control-Allow-Origin: https://moneyttuk.com` 확인

---

## 2. 재배포 시 Gradle 빌드 실패 (디스크 부족)

### 증상
```
Could not update /app/.gradle/8.13/fileChanges/last-build.bin
Could not receive a message from the daemon.
failed to solve: process "/bin/sh -c ./gradlew bootJar --no-daemon" exit code: 1
BUILD FAILED
```

### 원인
EC2 t3.micro 기본 디스크 8GB 중 실제 사용량:
- OS/시스템: ~2GB
- containerd (Docker 이미지): ~970MB
- snap: ~860MB
- Docker 빌드 캐시: ~300MB+

소스 변경 전엔 Docker 레이어 캐시로 `bootJar`를 건너뛰다가, 소스 수정 후 첫 빌드 시 디스크 부족으로 실패.

### 해결 (응급)

```bash
df -h /                          # 여유 1GB 미만이면 prune 필수
docker system prune -a -f        # -a 필수 (이미지까지 삭제)
docker compose up -d --build auth-server
```

> `-a` 없이 prune하면 이미지가 남아 디스크 부족이 재발한다.

### 근본 해결
EBS 볼륨 8GB → 20GB 확장 (프리티어 30GB 이내라 추가 비용 없음).
**볼륨 확장 후 파티션/파일시스템 확장이 별도로 필요함 → 아래 4번 항목 참조.**

---

## 3. 로그인 시 `IllegalArgumentException: 없는 계정입니다.` 500 에러

### 증상
- CORS 해결 후 요청은 정상 도달
- 로그인 API가 500 응답 반환

### 원인
RDS에 테이블 미생성 상태라 계정 조회 실패. 도메인 예외(`IllegalArgumentException`)가 그대로 500으로 노출됨.

### 해결
1. RDS에 4개 테이블 생성 (DDL 실행) — 2026-05-09 완료
   - `member`, `refresh_token`, `winning_numbers`, `purchase_games`
   - 로컬 `member` DB 스키마를 `SHOW CREATE TABLE`로 추출 후 RDS에 그대로 적용
2. 테스트 계정 INSERT (사용자 측에서 진행)

### TODO
- `IllegalArgumentException` → `@ExceptionHandler`로 잡아 400/404 반환하도록 예외 처리 추가

---

## 4. EBS 볼륨 확장 후에도 디스크가 늘어나지 않음

### 증상
AWS 콘솔에서 EBS 볼륨을 8GB → 20GB로 확장했지만 EC2에서 `df -h /` 결과는 여전히 6.7G.

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       6.7G  6.0G  702M  90% /
```

### 원인
EBS 볼륨 확장은 2단계 작업:

| 단계 | 설명 | 비유 |
|------|------|------|
| 1 (AWS 콘솔) | EBS 볼륨 자체 크기 변경 | 하드디스크를 더 큰 것으로 교체 |
| 2 (EC2 OS 내부) | 파티션/파일시스템 확장 | 늘어난 공간을 OS가 인식하도록 설정 |

콘솔에서는 1단계만 처리되므로 OS 내부에서 2단계를 직접 실행해야 함.

### 해결

```bash
# 디바이스명 확인 (Nitro 기반 인스턴스는 nvme)
lsblk

# 파티션 확장
sudo growpart /dev/nvme0n1 1

# 파일시스템 확장 (ext4)
sudo resize2fs /dev/nvme0n1p1

# 결과 확인
df -h /
```

> 구형 인스턴스 타입은 `/dev/xvda` 일 수 있음. `lsblk`로 실제 디바이스명 확인 후 진행.

### 해결 일자
2026-05-09 — 6.7G → 19G 확장 완료. 이후 `docker system prune` 없이도 정상 빌드.

---

## 5. CORS 헤더 중복 — `Access-Control-Allow-Origin` multiple values

### 증상
RDS 테이블 생성 후 회원가입 시도 시 브라우저 콘솔:

```
Access to fetch at 'https://api.moneyttuk.com/auth/sign-up' from origin
'https://www.moneyttuk.com' has been blocked by CORS policy:
The 'Access-Control-Allow-Origin' header contains multiple values
'https://www.moneyttuk.com, https://www.moneyttuk.com',
but only one is allowed.
```

### 원인
**nginx와 Spring Security 둘 다 CORS 헤더를 추가**하고 있어 응답에 같은 헤더가 2번 붙음.

- nginx: `add_header Access-Control-Allow-Origin $cors_origin always;`
- Spring SecurityConfig: `CorsConfigurationSource` 빈으로 헤더 추가

1번 트러블 해결 시점에는 nginx 설정이 단순했지만, 이후 nginx CORS 블록이 추가되면서 중복 발생.

### 해결
**Spring CORS를 비활성화하고 nginx에만 위임.**

```java
// SecurityConfig.java
return http.csrf(AbstractHttpConfigurer::disable)
    .cors(AbstractHttpConfigurer::disable)   // ← Spring CORS 끔
    .securityMatcher("/**")
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/auth/**").permitAll()
        .requestMatchers("/actuator/**").permitAll()
        .anyRequest().authenticated())
    .build();
```

`corsConfigurationSource()` 빈은 통째로 제거.

### 재배포

```bash
df -h /
docker compose up -d --build auth-server
```

### 해결 일자
2026-05-09

### 교훈
CORS는 **한 곳에서만** 처리한다. 게이트웨이/프록시/애플리케이션 중 책임 위치를 명확히 정해두지 않으면 헤더 중복이 발생.

---

## 6. WSL Ubuntu에서 Windows 로컬 MySQL 접속 불가

### 증상

```
ERROR 2002: Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'
ERROR 2003: Can't connect to MySQL server on '127.0.0.1:3306' (111)
ERROR 1130: Host '172.25.255.158' is not allowed to connect to this MySQL server
```

### 원인
1. WSL2는 Windows와 별도 네트워크 인터페이스를 가지므로 `127.0.0.1`은 Ubuntu 자기 자신을 가리킴
2. Windows MySQL이 `127.0.0.1`만 바인딩 → 외부(WSL) 연결 거부
3. MySQL 유저 권한이 `localhost`로만 등록 → WSL IP에서 접근 차단

### 해결

**1) Windows 호스트 IP 확인 (WSL에서):**

```bash
cat /etc/resolv.conf | grep nameserver
# nameserver 10.255.255.254  ← Windows 호스트 IP
```

또는 Windows에서 `ipconfig` → `vEthernet (WSL)` 의 IPv4 주소 확인.

**2) Windows MySQL `my.ini` 수정 (관리자 권한):**

```ini
[mysqld]
bind-address=0.0.0.0
```

경로 예시: `C:\ProgramData\MySQL\MySQL Server 8.0\my.ini`

**3) MySQL 서비스 재시작 (관리자 CMD):**

```cmd
net stop mysql80
net start mysql80
```

**4) MySQL 유저 권한 확장 (Windows mysql 셸에서):**

```sql
UPDATE mysql.user SET host='%' WHERE user='root';
FLUSH PRIVILEGES;
```

**5) WSL에서 접속:**

```bash
mysql -h 10.255.255.254 -P 3306 -u root -p
```

### 보안 주의
`bind-address=0.0.0.0` + `host='%'` 조합은 같은 와이파이의 다른 PC에서도 접근 가능. 카페/공용 와이파이에서는 host를 WSL IP로 제한하거나 Windows 방화벽으로 3306 포트 차단 권장.

### 해결 일자
2026-05-09

---

## 7. RDS GUI 접속 (DBeaver SSH 터널)

### 상황
RDS Security Group이 EC2에서만 3306을 허용하므로 로컬 PC에서 직접 접속 불가.

### 해결 (정석)
DBeaver의 **SSH Tunnel** 기능으로 EC2를 경유하여 접속.

**Main 탭:**
- Host: `auth-server-db.cxoiks04o8uv.ap-northeast-2.rds.amazonaws.com`
- Port: `3306`
- Database: `authdb`
- Username/Password: 앱 계정

**SSH 탭 (Use SSH Tunnel 체크):**
- Host: `43.202.153.119` (EC2 Elastic IP)
- Port: `22`
- User Name: `ubuntu`
- Authentication: `Public Key`
- Private Key: `~/.ssh/moneyttuk.com.pem`

### 대안 (비추천)
RDS Security Group에 내 PC IP를 임시 허용. 와이파이 IP가 바뀌면 재설정 필요하고, 외부 노출이 늘어남.

### 해결 일자
2026-05-09

---

## 작업 전 체크리스트

재배포 전:

```bash
df -h /
# Avail이 1G 미만이면 docker system prune -a -f 먼저 실행
# 또는 EBS 확장 + growpart/resize2fs 진행
```

CORS 변경 시:
- nginx와 Spring 둘 중 한 곳에서만 처리되는지 확인
- 응답 헤더에 `Access-Control-Allow-Origin`이 1개만 있는지 curl로 검증

```bash
curl -I -H "Origin: https://www.moneyttuk.com" https://api.moneyttuk.com/auth/sign-up
```
