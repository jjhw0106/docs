# AWS EC2 세팅 매뉴얼 - 2단계

## 개요

EC2 인스턴스 생성, SSH 접속, Docker 설치까지의 과정.

---

## 1. EC2 인스턴스 생성

1. AWS 콘솔 검색창 → **EC2** → 우측 상단 리전 **서울(ap-northeast-2)** 확인
2. **인스턴스 시작** 클릭
3. 아래 설정으로 구성:

| 항목 | 값 |
|------|-----|
| 이름 | `auth-server` |
| AMI | Ubuntu Server 26.04 LTS |
| 인스턴스 유형 | `t3.micro` |
| 스토리지 | 기본 8GB |

---

## 2. 키 페어 생성

1. **새 키 페어 생성** 클릭
2. 이름: 원하는 이름 (예: `프로젝트명.pem`)
3. 유형: RSA, 형식: `.pem`
4. **키 페어 생성** → `.pem` 파일 자동 다운로드
5. 안전한 폴더에 보관 (절대 git 커밋 금지)

> `.pem` 파일 = EC2에 SSH 접속할 때 사용하는 열쇠. 분실 시 재발급 불가.

---

## 3. Security Group 설정

**네트워크 설정 → 편집** 클릭 후 아래 규칙 추가:

| 포트 | 프로토콜 | 소스 | 용도 |
|------|---------|------|------|
| 22 | SSH | 내 IP | SSH 접속 |
| 80 | HTTP | 0.0.0.0/0 | 웹 트래픽 |
| 443 | HTTPS | 0.0.0.0/0 | HTTPS 트래픽 |
| 9001 | TCP | 0.0.0.0/0 | auth-server |

> SSH(22)는 내 IP만 허용. 와이파이 IP는 유동이므로 바뀌면 Security Group에서 수정 필요.

---

## 4. Elastic IP 할당 (고정 IP)

1. 왼쪽 메뉴 **탄력적 IP** (네트워크 및 보안 섹션)
2. **탄력적 IP 주소 할당** → 설정 그대로 **할당**
3. 할당된 IP 체크박스 선택 → **작업** → **탄력적 IP 주소 연결**
4. 인스턴스: `auth-server` 선택 → **연결**

> Elastic IP 없으면 EC2 재시작 시 IP가 바뀜.

---

## 5. SSH 접속

`.pem` 파일은 WSL Linux 파일시스템으로 복사 후 사용해야 chmod가 정상 적용됨.

```bash
# pem 파일을 WSL .ssh 폴더로 복사(/home/jjhw/.ssh)
cp "/mnt/d/경로/키이름.pem" ~/.ssh/
chmod 400 ~/.ssh/키이름.pem

# SSH 접속
ssh -i ~/.ssh/키이름.pem ubuntu@[Elastic IP]
```

> `/mnt/d/...` 경로는 Windows 드라이브 마운트라 chmod가 적용 안 됨.
> SSH는 키 파일 권한이 400(나만 읽기)이 아니면 접속 거부.

---

## 6. Docker 설치

EC2 접속 후 실행:

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu
```

재접속 후 확인:

```bash
docker ps
```

---

## 완료 체크리스트

- [ ] EC2 t3.micro 생성 (Ubuntu 26.04, 서울)
- [ ] 키 페어 생성 및 안전한 위치에 보관
- [ ] Security Group 포트 설정 (22/80/443/9001)
- [ ] Elastic IP 할당 및 연결
- [ ] SSH 접속 성공
- [ ] Docker 설치 및 동작 확인
