---
title: 'AWS와 CI/CD 리팩토링을 해보며'
tags:
  - CI/CD
  - GitHub Actions
  - AWS
date: '2025-07-02'
thumbnail: '/assets/img/infra/03/infra03_0.png'
---

`Spring Boot` 프로젝트를 `AWS EC2`에 배포하면서 놓쳤던 부분과 개선해야 할 점들을 정리해두었고, 이를 바탕으로 리팩토링하며 학습을 진행했습니다.

## (1) `MongoDB`, `Redis` 재배포 시 데이터가 유지되는 이유

아래는 제가 작성한 `docker-compose.yml`입니다. `MongoDB`와 `Redis` 컨테이너를 여러 번 재배포했지만 **데이터가 유지된 이유**는 `volumes` 설정 덕분이었습니다. `Docker`는 `mongo_data`라는 이름의 **볼륨을 따로 생성**하고, 컨테이너가 삭제되어도 데이터를 해당 볼륨에 보존합니다.

```yml
services:
  mongo:
    image: mongo:6
    container_name: cote-mongo
    restart: always
    ports:
      - '27017:27017'
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:alpine
    container_name: cote-redis
    restart: always
    ports:
      - '6379:6379'
```

> 📌 개선 & 공부할 부분:
>
> - `Docker`의 볼륨, 캐시가 어떻게 동작하는지 개념 정확히 이해

## (2) `Github Secrets` 수정 후에도 적용되지 않던 문제

`Github Actions`에서 환경 변수를 `Github Secrets`에 등록해 사용했습니다. 그러나 Secrets 내용을 수정한 뒤 재배포해도 **EC2에 반영되지 않는 문제**가 발생했습니다. 검색을 통해 **Docker 볼륨, 캐시, 폴더를 수동으로 삭제한 후 재배포**하면 반영된다는 것을 확인했고, 이후 `deploy.yml`에 해당 명령어를 추가하여 자동화했습니다.

> 📌 **개선 & 공부할 부분**
>
> - `Secrets` 변경 시에도 안정적으로 반영되도록 배포 스크립트 구성

## (3) CI/CD `Job` 분리 필요성

현재 `Github Actions`에서 `deploy` job 안에 **빌드와 배포** 단계가 모두 포함돼 있습니다. 이를 **빌드와 배포로 job을 분리**하면, 각 단계의 실패 원인을 명확히 파악하고, 재사용성도 높일 수 있습니다.

> 📌 개선 & 공부할 부분:
>
> - `build` job에서 빌드 후 아티팩트를 생성 및 업로드
> - `deploy` job은 해당 아티팩트를 기반으로 `EC2`에 배포

# 1. `MongoDB`, `Redis` 재배포 시 데이터가 유지되는 이유

---

```yml
services:
  mongo:
    //...
    volumes:
      - mongo_data:/data/db
```

## ✅ Docker Volume이란?

Docker 컨테이너는 **휘발성**입니다. 컨테이너 내부에서 만든 파일은 컨테이너가 삭제되면 함께 사라집니다. 하지만 영구적으로 보존해야 할 데이터가 존재할 수 있습니다. 예를 들어, 배포할 때마다 `docker compose down` 을 하여 DB가 초기화된다면 매우 불편하겠죠. 이럴 때, **컨테이너 외부 저장소인 volume**을 사용하면 컨테이너가 사라져도 데이터는 유지됩니다.

> `docker compose down`
> 컨테이너 삭제, 네트워크 삭제, 볼륨 삭제 X, 이미지 삭제 X

> 📂 Volume의 종류
>
> - **volume**: Docker가 자동으로 관리하는 저장소. `/var/lib/docker/volumes/` 아래에 생성됨
> - **bind mount**: 호스트 OS의 특정 디렉토리를 직접 지정해 컨테이너와 연결
> - **tmpfs**: 메모리 기반 임시 저장소. 컨테이너 재시작 시 데이터 사라짐 (고속 처리용)

위 설정은 `mongo_data`라는 **volume을 생성**하고, MongoDB의 `/data/db` 디렉토리와 연결합니다. MongoDB는 모든 데이터를 `/data/db`에 저장하므로, 이 경로를 volume으로 분리하면 데이터가 유지됩니다. `docker volume inspect [볼륨명]` 명령어를 통해 실제 저장 경로 및 메타 정보를 확인할 수 있습니다.

<img src="/assets/img/infra/03/infra03_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- `Name`: Docker가 실제 사용하는 볼륨 이름 (서비스명\_볼륨명)
- `Driver`: local은 Docker가 로컬 디스크에 저장한다는 의미
- `Labels`: **docker-compose.yml**에서 자동으로 붙여준 메타데이터
- `Mountpoint`: 실제 데이터가 저장되는 디렉토리 경로

`docker-compose.yml`에서 volumes를 설정하면 volume은 **컨테이너 외부의 /var/lib/docker/volumes/...에 실제 데이터가 저장**되고, **컨테이너 내부에서는 /app/data 같은 경로로 연결**(mount)되어 사용됩니다. 따라서 **볼륨이 삭제되지 않는 한,** 컨테이너가 사라져도 데이터는 유지됩니다.

# 2. `Github Secrets` 수정 후에도 적용되지 않던 문제

---

사실, 저는 `Github Secrets`의 변경 사항이 반영되지 않는 문제를 겪었고, 이를 해결하기 위해 Docker 볼륨과 캐시를 삭제하는 스크립트를 추가하여 해결했었습니다.

```yml
script: |
  cd ~/cote-house/backend

  docker compose down
  docker image prune -f
  docker volume prune -f
  docker compose up -d --build
```

하지만 Docker 볼륨에 대해 개념을 이해하니, 이 문제는 볼륨과는 관련이 없고, **이미지 캐시**로 인해 발생한 것이었다는 확신이 들었습니다.

```yml
run: |
  envsubst < ./src/main/resources/application-prod.yml > ./src/main/resources/application-prod.yml.tmp
```

`deploy.yml`에서는 위 작업을 통해 GitHub Actions 내에서 Secrets를 치환하여 **application-prod.yml** 파일을 생성합니다. 이 파일은 EC2로 복사된 후, Docker 빌드 시 함께 포함됩니다.

그럼에도 변경 내용이 반영되지 않았던 이유는, `application-prod.yml`이 바뀌었더라도 Docker가 이전 캐시를 사용해 이미지를 빌드했기 때문입니다. Docker는 build cache를 적극적으로 활용하기 때문에, 복사되는 파일의 변경을 감지하지 못하면 이전 이미지를 그대로 재사용합니다.

## ✅ Docker Build Cache 동작 방식

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
```

이 `Dockerfile`에는 총 4개의 명령어가 있습니다. Docker는 `Dockerfile`의 각 명령어를 **하나의 Layer**로 인식하여 이미지를 빌드하며, 처음 빌드 시 각 명령어에 대한 **캐시를 저장**합니다. 이후 동일한 명령어가 **동일한 입력(파일, 디렉토리 등)**을 받으면 해당 레이어는 **캐시된 결과를 재사용**합니다.
특히, Docker는 `COPY` 명령어에서 지정한 파일들의 내용이 **이전과 동일**하다고 판단되면(파일 경로, 이름, 수정 시각 또는 해시값이 동일할 경우) 해당 레이어를 캐시로 대체합니다.
GitHub Actions에서 `envsubst`로 `application-prod.yml`을 동적으로 생성하고 이를 EC2로 업로드했다고 하더라도, `Dockerfile`에서 `COPY` 명령어로 해당 파일을 복사하는 단계에서 Docker가 **파일의 변경을 감지하지 못하면**, 이전 레이어를 그대로 재사용합니다.결과적으로, **변경된 설정이 반영되지 않은 상태의 application-prod.yml**로 빌드된 이미지가 그대로 재사용된 것입니다.

# 3. CI/CD `Job` 분리 필요성

`Job`을 분리하기 전에, 현재 배포 흐름에 대해 먼저 설명하겠습니다.

## ✅ 현재 배포 흐름

```yml
name: Deploy Spring Boot to EC2

# 트리거 조건: PR이 main 브랜치에 merge되어 close될 때 실행
on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  deploy:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest

    steps:
      # 코드 체크아웃 및 빌드 준비: JDK 세팅, Gradle 실행 권한 부여
      - name: Checkout source code
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Grant permission to Gradle wrapper
        run: chmod +x ./gradlew

      # 환경 변수 치환: application-prod.yml 안의 ${...} 같은 환경변수를 실제 값으로 바꿈
      # Github Secrets에서 주입
      - name: Replace environment variables in application-prod.yml
        run: |
          envsubst < ./src/main/resources/application-prod.yml > ./src/main/resources/application-prod.yml.tmp
          mv ./src/main/resources/application-prod.yml.tmp ./src/main/resources/application-prod.yml
        env:
          MONGODB_URI: ${{ secrets.MONGODB_URI }}
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
          JWT_ACCESS_EXPIRATION: ${{ secrets.JWT_ACCESS_EXPIRATION }}
          JWT_REFRESH_EXPIRATION: ${{ secrets.JWT_REFRESH_EXPIRATION }}
          PROD_ORIGIN: ${{ secrets.PROD_ORIGIN }}
          MAIL_USERNAME: ${{ secrets.MAIL_USERNAME }}
          MAIL_PASSWORD: ${{ secrets.MAIL_PASSWORD }}

      # Spring Boot 프로젝트 빌드: 테스트 제외하고 build/libs/에 jar 파일 생성
      - name: Build Spring Boot JAR (skip tests)
        run: ./gradlew clean build -x test

      # EC2로 전체 프로젝트 업로드: Dockerfile, docker-compose.yml, application-prod.yml, JAR 등 모두 포함된 프로젝트 전체를 EC2 특정 디렉토리에 복사
      - name: Upload project to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_KEY }}
          source: '.'
          target: '~/cote-house/backend'

      # EC2에 SSH 접속 후 Docker 배포: 기존 컨테이너 종료 및 정리
      # docker-compose.yml에 새 이미지 빌드 & 컨테이너 실행
      - name: SSH to EC2 and deploy
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_KEY }}
          script: |
            cd ~/cote-house/backend

            echo "✅ Starting deployment with Docker Compose"
            docker compose down
            docker image prune -f
            # --build: 이미지 빌드
            # up: 빌드한 이미지를 기반으로 컨테이너 생성 및 실행
            docker compose up -d --build
```

**_EC2로 전체 프로젝트 업로드_**
현재는 전체 프로젝트를 EC2에 복사하고 있지만, Spring Boot 애플리케이션은 **JAR 파일만 있어도 실행**이 가능합니다. 다만, **Docker 이미지를 EC2에서 직접 빌드**하고 있기 때문에, `Dockerfile`과 `docker-compose.yml` 파일도 함께 복사해야 합니다.
가장 이상적인 방식은 GitHub Actions에서 Docker 이미지를 빌드한 뒤 EC2에 전달하는 것이지만, GitHub Actions는 EC2로 Docker 이미지를 직접 전송하는 기능을 지원하지 않기 때문에, 일반적으로는 **Docker Hub와 같은 외부 레지스트리**를 거치는 방식이 사용됩니다. 하지만 추가 설정이 필요하고 복잡성이 증가하기 때문에, 현재는 기존 방식을 유지하되 **전체 프로젝트가 아닌, 필요한 파일만 선택적으로 복사**하도록 개선할 예정입니다.

## ✅ docker-compose.yml 설정

```yml
version: "3.8"

services:
  # mongo:6 공식 이미지 사용 (직접 Dockerfile로 빌드 X)
  mongo:
    image: mongo:6
    //...

  # redis:alpine 공식 이미지 사용 (직접 Dockerfile로 빌드 X)
  redis:
    image: redis:alpine
    //...

  # 로컬(EC2)에 있는 Dockerfile을 사용해서 직접 이미지 빌드
  spring-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: cote-spring
    ports:
      - "8080:8080"
    depends_on:
      - mongo
      - redis
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    restart: always

volumes:
  mongo_data:
```

## ✅ Dockerfile 설정

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
```

Spring Boot 애플리케이션 Docker 이미지 빌드 과정 정의

- `FROM`: 베이스 이미지 지정
- `WORKDIR`: **/app** 디렉토리를 작업 디렉토리로 지정
- `COPY`: EC2에 배포된 프로젝트의 **build/libs/** 폴더의 jar파일을 **/app/app.jar**로 복사
- `EXPOSE`: 컨테이너가 사용하는 8080 포트를 외부에 열어줌
- `ENTRYPOINT`: 컨테이너가 시작될 때 실행할 명령

이제 `deploy`에 포함되어 있던 모든 **step**을 `build`와 `deploy`로 분리하겠습니다. `build` Job에서는 jar 파일을 빌드하고, `deploy` Job에서는 jar 파일과 Docker 관련 파일만 EC2에 복사한 뒤, 이미지를 빌드하고 컨테이너를 실행하겠습니다.

```yml
name: Deploy Spring Boot to EC2

on:
  pull_request:
    branches:
      - main
    types:
      - closed

jobs:
  build:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest

    steps: //...

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps: //...
```
