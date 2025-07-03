---
title: 'CI/CD의 개념과 GitHub Actions 실습!'
tags:
  - CI/CD
  - GitHub Actions
date: '2025-06-25'
thumbnail: '/assets/img/infra/02/infra02_0.png'
---

# CI/CD의 개념과 GitHub Actions 실습!

---

## ⚙️ CI (Continuous Integration) 란

동일한 프로젝트에서 작업하는 모든 사람이 **정기적으로** 코드 베이스의 변경 사항을 **중앙 저장소(main 브랜치 등)에 병합**하도록 하는 방식

> - main 브랜치에 코드 push
> - 자동으로 빌드 및 테스트 실행
> - 테스트 결과를 피드백으로 받아 확인

```yaml
on:
  push:
    branches: [main]

jobs:
  build-and-test: #워크플로의 첫번째 작업 이름
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: ./gradlew build
```

## ⚙️ CD (Continuous Delivery / Continuous Deployment) 란

### ① 자동 배포 방식 (Continuous Deployment)

> <img src="/assets/img/infra/02/infra02_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 테스트를 통과하면 운영 서버까지 **자동 배포**
- **배포 속도**가 중요한 서비스 (스타트업, SaaS 등)에서 자주 사용

```yaml
deploy: #워크플로의 두번째 작업 이름
  runs-on: ubuntu-latest
  needs: build-and-test
  steps:
    - name: Deploy to AWS
      run: ./deploy.sh
```

### ② 수동 배포 방식 (Continuous Delivery)

> <img src="/assets/img/infra/02/infra02_2.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 운영 환경에 배포하기 전에 사람이 **직접 검토하고 승인**
- 금융/보안 시스템처럼 **신중한 배포**가 필요한 곳에서 자주 사용

```yaml
deploy: #워크플로의 두번째 작업 이름
  runs-on: ubuntu-latest
  needs: build-and-test
  environment:
    #GitHub 저장소의 Settings → Environments에 정의한 환경 (수동 승인 여부, 시크릿 변수, 보호 정책 등을 설정)
    name: production
    url: https://your-app.com
    - name: Deploy to AWS
      run: ./deploy.sh
```

## AWS EC2 인스턴스 생성 및 보안 그룹 설정

### 1. AWS EC2 생성

<img src="/assets/img/infra/02/infra02_3.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

> - **EC2**: AWS 클라우드 환경에서 제공되는 가상 머신(Virtual Machine)
> - **AMI (Amazon Machine Image)**: 어떤 운영체제를 설치할지 (예: Ubuntu, Amazon Linux, Windows 등)
> - **인스턴스 타입**: CPU, 메모리, 네트워크 성능 등 하드웨어 성능 선택 (예: t2.micro)
> - **키 페어 (Key Pair)**: SSH 접속 시 사용하는 공개/개인 키
> - **보안 그룹 (Security Group)**: 방화벽 역할을 하며, 인바운드/아웃바운드 트래픽을 제어
> - **스토리지 (EBS 볼륨)**: 인스턴스의 하드디스크 역할

### 2. 보안 그룹 설정

<img src="/assets/img/infra/02/infra02_4.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

> 보안 그룹은 EC2 인스턴스에 연결된 접근 허용 규칙 집합,
> nginx 설정 이후에는 `80`포트와 `443`포트만 허용
>
> - **인바운드 규칙**: EC2로 들어오는 트래픽을 허용 (예: SSH 접속, 웹서버 요청 등)
> - **아웃바운드 규칙**: EC2에서 나가는 트래픽을 허용

<img src="/assets/img/infra/02/infra02_5.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

생성한 보안그룹을 EC2 인스턴스에 설정해줍니다.

<img src="/assets/img/infra/02/infra02_6.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

보안그룹이 EC2 인스턴스에 설정된 것을 확인할 수 있습니다. 이제 EC2 인스턴스에 Spring Boot 프로젝트를 GitHub Actions로 자동 배포(CI/CD)할 준비가 완료 됐습니다.

## GitHub Actions을 통한 배포

### 1. 🔧 [Docker관련 파일 작성]

- `Dockerfile`: Spring Boot 애플리케이션을 하나의 **Docker 이미지로 만드는 설계도**
- 내용: `.jar` 파일을 컨테이너에서 실행하도록 설정

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
```

- `docker-compose.yml`: 여러 Docker 이미지(dockerfile로부터 만들어지는 이미지 or 기존 이미지)를 어떻게 실행할지 정의하는 **구성도**
- 내용: 세 개의 컨테이너 정의 (spring-app, mongo, redis)

```yml
services:
  mongo: ...
  redis: ...
  spring-app: ...
```

### 2. 🔐 [GitHub Secrets 설정]

- `EC2_HOST`, `EC2_USER`, `EC2_KEY` : EC2 SSH 접속에 필요한 정보
- `MONGODB_URI`, `JWT_SECRET` 등 : Spring application-prod.yml에 필요한 환경 변수

### 3. 🚀 [GitHub Actions Workflow 실행 조건]

```yml
on:
  pull_request:
    branches: [main]
    types: [closed]
```

- PR이 **main 브랜치**에 `merge`될 때 배포

### 4. 🔧 [CI/CD 동작 순서]

```text
GitHub Actions 실행
 ├─ 1. 프로젝트 체크아웃
 ├─ 2. JDK 17 설정
 ├─ 3. Gradle wrapper 실행 권한 부여
 ├─ 4. application-prod.yml의 환경 변수 치환 (envsubst 이용)
 ├─ 5. Gradle 빌드
 └─ 6. EC2에 전체 프로젝트 전송 (scp-action)
```

### 5. 🛠 [EC2 서버 내 동작 (자동 실행)]

```bash
cd ~/cote-house/backend
docker compose down
docker compose up -d --build
```

`docker compose up -d --build`

- 현재 디렉토리에 있는 **docker-compose.yml** 파일을 기반으로 Docker 엔진이 정의된 서비스들을 컨테이너로 실행하는 명령어
  - spring-app의 이미지를 Dockerfile 기준으로 새로 빌드
  - mongo, redis는 기존 이미지로 컨테이너 생성
  - 모든 서비스를 백그라운드(-d) 모드로 실행

### 6. 🌐 [EC2 서버의 NGINX 설정]

- EC2 서버 단일 인스턴스에 프론트 + 백엔드를 모두 실행
- NGINX를 설치하고 **80, 443 포트**만 보안 그룹에 허용
- NGINX 설정 예시:

```nginx
server {
    listen 80;
    server_name my-domain.com;

    location /api {
        proxy_pass http://localhost:8080;
    }

    location / {
        proxy_pass http://localhost:3000;
    }
}
```
