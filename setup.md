# GitHub Actions CI/CD 배포 가이드 (Windows Server)

> GitHub Actions + Docker + GHCR + Self-hosted Runner 기반  
> main 브랜치 push/merge 시 운영 서버 자동 배포

---

## 목차

1. [전체 흐름](#1-전체-흐름)
2. [프로젝트 파일 구성](#2-프로젝트-파일-구성)
3. [운영 서버 최초 설정](#3-운영-서버-최초-설정-windows-server)
4. [SSH 키 생성 및 등록](#4-ssh-키-생성-및-등록)
5. [Self-hosted Runner 설치](#5-self-hosted-runner-설치)
6. [GitHub Secrets 설정](#6-github-secrets-설정)
7. [GitHub Actions Workflow](#7-github-actions-workflow)
8. [테스트 방법](#8-테스트-방법)
9. [실제 프로젝트 적용 가이드 (barrier-free-tool-v2)](#9-실제-프로젝트-적용-가이드-barrier-free-tool-v2)
10. [트러블슈팅](#10-트러블슈팅)

---

## 1. 전체 흐름

```
개발자 → main 브랜치 push/merge
    ↓
GitHub Actions (ubuntu-latest)
  1. 코드 checkout
  2. Docker 이미지 빌드 (linux/amd64)
  3. GHCR(ghcr.io)에 이미지 push
    ↓
GitHub Actions (self-hosted → 운영 서버)
  4. GHCR 로그인
  5. 최신 이미지 pull
  6. docker compose up -d (컨테이너 교체)
  7. 이전 이미지 prune
```

- 빌드는 GitHub 인프라(ubuntu)에서 수행
- 배포는 운영 서버에 설치된 Self-hosted Runner가 직접 실행 (SSH 불필요)

---

## 2. 프로젝트 파일 구성

```
프로젝트/
├── app.py                        # 애플리케이션 코드
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
└── .github/
    └── workflows/
        └── deploy.yml
```

### app.py
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World v1"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### requirements.txt
```
flask==3.1.0
```

### Dockerfile
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

### .dockerignore
```
.git
.github
__pycache__
*.pyc
*.pyo
.env
```

### docker-compose.yml
```yaml
services:
  web:
    image: ghcr.io/YOUR_GITHUB_USERNAME/YOUR_REPO:latest
    ports:
      - "31298:5000"    # 외부포트:컨테이너포트
    restart: always
```

---

## 3. 운영 서버 최초 설정 (Windows Server)

RDP로 서버 접속 후 **PowerShell (관리자 권한)** 실행.

### OpenSSH 서버 설치 (GitHub Actions SSH 배포 방식 사용 시)
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName "OpenSSH Server" -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

### Docker Desktop 설치
- https://www.docker.com/products/docker-desktop/ 에서 설치
- WSL2 backend 권장
- 설치 후 재시작

### 포트 방화벽 허용
```powershell
# 앱 외부 포트 허용 (예: 31298)
New-NetFirewallRule -DisplayName "Docker App Port" -Direction Inbound -Protocol TCP -LocalPort 31298 -Action Allow
```

### 공유기 포트포워딩
공유기 관리 페이지에서:
- 외부 포트 31298 → 서버 내부 IP:31298 (TCP)
- SSH용: 외부 포트 31222 → 서버 내부 IP:22 (TCP) ← SSH 배포 방식 사용 시

---

## 4. SSH 키 생성 및 등록

> Self-hosted Runner 방식에서는 SSH 직접 접속이 필요 없음.  
> SSH 배포 방식이 필요할 경우에만 설정.

### 로컬(macOS)에서 키 생성
```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions_key
# 패스프레이즈: Enter (빈값)
```

### 공개키 운영 서버에 등록 (관리자 계정 기준)
서버 PowerShell:
```powershell
New-Item -Path "C:\ProgramData\ssh" -ItemType Directory -Force
New-Item -Path "C:\ProgramData\ssh\administrators_authorized_keys" -ItemType File -Force
Add-Content -Path "C:\ProgramData\ssh\administrators_authorized_keys" -Value "ssh-ed25519 AAAA...pub키내용..."
icacls "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r /grant "SYSTEM:(R)" /grant "Administrators:(R)"
```

### 접속 테스트
```bash
ssh -i ~/.ssh/github_actions_key -p 31222 USERNAME@chamelion.iptime.org
```

---

## 5. Self-hosted Runner 설치

SSH 대신 **운영 서버에 Runner를 직접 설치**하는 방식.  
외부에서 서버로 들어오는 SSH 연결 없이, 서버가 직접 GitHub에서 작업을 가져와 실행함.

### GitHub에서 Runner 등록

`https://github.com/YOUR_USERNAME/YOUR_REPO/settings/actions/runners/new`

- **Operating System: Windows** 선택
- 페이지에서 제공하는 명령어 복사

### 서버 PowerShell (관리자 권한)에서 실행

```powershell
# 원하는 경로에 폴더 생성
mkdir D:\dev\actions-runner
cd D:\dev\actions-runner

# GitHub 페이지 명령어 그대로 실행 (다운로드, 압축 해제, 등록)
# 마지막 config.cmd 실행 시 --runasservice 플래그 추가
.\config.cmd --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_TOKEN --runasservice
```

> `--runasservice` 를 붙이면 Windows 서비스로 자동 등록되어  
> 서버 재시작 후에도 Runner가 자동 실행됨.

### Runner 서비스 계정 변경 (Docker 접근 권한)

Runner가 서비스로 실행될 때 기본 계정(SYSTEM)은 Docker Desktop에 접근 불가.  
`services.msc`에서 Runner 서비스의 로그온 계정을 변경해야 함.

```
Win + R → services.msc
→ GitHub Actions Runner 서비스 찾기
→ 우클릭 → 속성 → 로그온 탭
→ 계정 지정: .\서버계정명 (예: .\chamelion-pc2)
→ 비밀번호 입력 후 확인
→ 서비스 재시작
```

### Runner 상태 확인

`https://github.com/YOUR_USERNAME/YOUR_REPO/settings/actions/runners`  
→ **Idle** 상태이면 정상

---

## 6. GitHub Secrets 설정

`https://github.com/YOUR_USERNAME/YOUR_REPO/settings/secrets/actions`

**New repository secret** 으로 등록:

| Name | Value | 비고 |
|---|---|---|
| `HOST` | `chamelion.iptime.org` | SSH 배포 방식 사용 시만 필요 |
| `USERNAME` | 서버 계정명 | SSH 배포 방식 사용 시만 필요 |
| `SSH_KEY` | 비밀키 전체 내용 | SSH 배포 방식 사용 시만 필요 |
| `GITHUB_TOKEN` | (자동 제공) | 별도 등록 불필요 |

> Self-hosted Runner 방식에서는 HOST, USERNAME, SSH_KEY 불필요.

---

## 7. GitHub Actions Workflow

### Self-hosted Runner 방식 (최종 채택)

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

env:
  IMAGE: ghcr.io/YOUR_GITHUB_USERNAME/YOUR_REPO

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: ${{ env.IMAGE }}:latest

  deploy:
    runs-on: self-hosted
    needs: build
    permissions:
      contents: read
      packages: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Pull latest image
        run: docker compose pull

      - name: Restart container
        run: docker compose up -d

      - name: Prune old images
        run: docker image prune -f
```

---

## 8. 테스트 방법

### v1 배포 확인
```bash
curl http://chamelion.iptime.org:31298
# 응답: Hello World v1
```

### v2 재배포 검증
```bash
# 브랜치 생성
git checkout -b feature/update-v2

# app.py 수정: "Hello World v1" → "Hello World v2"

# 커밋 및 푸시
git add app.py
git commit -m "test: update to v2"
git push origin feature/update-v2
```

GitHub에서 PR 생성 → Merge → Actions 자동 실행 확인  
→ `curl http://chamelion.iptime.org:31298` → `Hello World v2`

---

## 9. 실제 프로젝트 적용 가이드 (barrier-free-tool-v2)

### 현재 구조

| 서비스 | 이미지 | 포트 | 비고 |
|---|---|---|---|
| db | postgres:15-alpine | 5436:5432 | 표준 이미지, 빌드 불필요 |
| redis | redis:7-alpine | 6397:6379 | 표준 이미지, 빌드 불필요 |
| backend | ./backend/Dockerfile | 8100:8000 | 커스텀 이미지, 빌드 필요 |
| frontend | ./frontend/Dockerfile.dev | 5173:5173 | 커스텀 이미지, 빌드 필요 |

### 핵심 문제: 소스 볼륨 마운트

현재 docker-compose.yml은 **개발 모드**로, 소스 코드가 볼륨으로 마운트되어 있음:

```yaml
# 현재 (개발 모드) - CI/CD 불가
backend:
  build: ./backend
  volumes:
    - ./backend:/app    # 소스 코드 마운트
```

운영 배포 시에는 소스 코드를 이미지 안에 포함시켜야 함:

```yaml
# 운영 모드 - CI/CD 가능
backend:
  image: ghcr.io/YOUR_ORG/barrier-free-tool-v2-backend:latest
  volumes:
    - ./data:/app/data  # 데이터만 마운트 (소스 제외)
```

### 적용 절차

#### Step 1. 프론트엔드 프로덕션 Dockerfile 작성

현재 `Dockerfile.dev`(개발용)만 있으므로 `Dockerfile`(프로덕션용) 추가:

```dockerfile
# frontend/Dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

#### Step 2. docker-compose.prod.yml 작성 (운영 전용)

개발용 docker-compose.yml은 그대로 유지하고, 운영용을 별도로 작성:

```yaml
# docker-compose.prod.yml
services:
  db:
    image: postgres:15-alpine
    container_name: bf_db_v2
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

  redis:
    image: redis:7-alpine
    container_name: bf_redis_v2
    volumes:
      - redis_data:/data
    restart: always

  backend:
    image: ghcr.io/YOUR_ORG/barrier-free-tool-v2-backend:latest
    container_name: bf_backend_v2
    ports:
      - "${BACKEND_PORT:-8100}:8000"
    volumes:
      - ./data:/app/data    # 소스 제외, 데이터만 유지
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      SECRET_KEY: ${SECRET_KEY}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: always

  frontend:
    image: ghcr.io/YOUR_ORG/barrier-free-tool-v2-frontend:latest
    container_name: bf_frontend_v2
    ports:
      - "${FRONTEND_PORT:-5173}:80"
    depends_on:
      - backend
    restart: always

volumes:
  postgres_data:
  redis_data:
```

#### Step 3. GitHub Actions Workflow 작성

백엔드/프론트엔드 각각 이미지를 빌드하고 배포:

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  ORG: YOUR_ORG_OR_USERNAME

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push backend
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          platforms: linux/amd64
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.ORG }}/barrier-free-tool-v2-backend:latest

      - name: Build and push frontend
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          file: ./frontend/Dockerfile
          platforms: linux/amd64
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.ORG }}/barrier-free-tool-v2-frontend:latest

  deploy:
    runs-on: self-hosted
    needs: build
    permissions:
      contents: read
      packages: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Pull latest images
        run: docker compose -f docker-compose.prod.yml pull

      - name: Restart containers
        run: docker compose -f docker-compose.prod.yml up -d

      - name: Prune old images
        run: docker image prune -f
```

#### Step 4. .env 파일 처리

`.env`는 레포에 포함하지 않고 서버에 직접 관리:
- 운영 서버의 프로젝트 경로에 `.env` 파일 유지
- GitHub Secrets에 민감 정보 등록 후 workflow에서 주입하는 방식도 가능

#### Step 5. Self-hosted Runner 등록

[5번 항목](#5-self-hosted-runner-설치)과 동일하게 **barrier-free-tool-v2 레포에도** Runner 등록.

---

## 10. 트러블슈팅

| 오류 | 원인 | 해결 |
|---|---|---|
| `missing server host` | GitHub Secrets HOST 미등록 | Secrets 등록 확인 |
| `i/o timeout` (SSH) | 공유기/ISP에서 클라우드 IP 차단 | Self-hosted Runner 방식으로 전환 |
| `permission denied to docker API` | Runner 서비스 계정이 Docker 접근 불가 | services.msc에서 Runner 계정을 사용자 계정으로 변경 |
| `denied: denied` (GHCR login) | deploy job에 packages 권한 없음 | workflow deploy job에 `permissions: packages: read` 추가 |
| `port already in use` | 기존 컨테이너가 포트 점유 | `docker compose down` 후 재시작 |
| Runner 서비스 시작 안 됨 | 계정 비밀번호 오류 또는 권한 부족 | 로컬 시스템 계정으로 복구 후 재설정 |
| `image not found` | GHCR 이미지명 대소문자 불일치 | 이미지명 항상 소문자로 통일 |
