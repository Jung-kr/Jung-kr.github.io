---
title: 'GitHub Actions에 대해 알아보자!'
tags:
  - CI/CD
  - GitHub Actions
date: '2025-06-18'
thumbnail: '/assets/img/infra/01/infra01_0.png'
---

# GitHub Actions....

---

## ⚙️ GitHub Actions란

GitHub에서 제공하는 **CI/CD 플랫폼**입니다. Pull Request가 생성되면 해당 코드에 대한 **테스트와 빌드를 자동으로 실행**하거나, Merge된 PR에 대한 **배포를 자동화**할 수 있습니다. 이런 DevOps 작업을 넘어, 단순히 Issue가 생성되었을 때 적절한 label을 등록하는 등의 단순한 워크플로도 작성해볼 수 있습니다.

`Jenkins`, `Travis CI`, `Circle CI` 등 여러 CI/CD를 위한 제품이 많이 출시되어 있지만, GitHub Actions는 GitHub 자체에서 지원하므로 GitHub과 함께 사용할 때 그 사용성이 매우 매끄럽습니다. 또한, 물론 어느정도 제한이 있지만 **컴퓨팅 리소스를 GitHub에서 제공**해주며, 무료로 (public 저장소 기준) 사용할 수 있습니다.

> 출처: [GitHub Actions의 핵심 개념](https://hudi.blog/github-actions-concepts/)

하나의 워크플로를 정의하는 `.yml` 파일을 봅시다.

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: echo "Building..."

  test:
    runs-on: ubuntu-latest
    needs: build # test가 build에 의존(needs) 하고 있으므로 build가 끝난 후 실행
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: echo "Testing..."
```

위 예시가 지금은 조금 복잡하게 느껴질 수 있지만, **GitHub Actions** 구성 요소를 이해한다면 위 예시에 대한 이해가 수월해질 것입니다.

## ⚙️ GitHub Actions 구성 요소

> <img src="/assets/img/infra/01/infra01_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

각각의 구성 요소를 살펴보기 전에 전체적인 흐름을 요약해보면 아래와 같습니다.

> 하나의 **Event** (`push`, `pull_request` 등) 발생
> → 하나의 **Workflow**가 실행되고
> → 그 **Workflow** 안에는 여러 **Job**이 있고
> → 각 **Job**은 자기만의 **Runner**에서 실행되며
> → **Job** 안에는 여러 **Step**이 순차적으로 실행

### ✔ Workflow (워크플로)

- 하나 이상의 **Job**이 실행되는 자동화 프로세스
- GitHub 리포지토리의 `.github/workflows` 디렉토리에 정의

### ✔ Event (이벤트)

GitHub Actions를 트리거하는 행위로, 대표적으로 `push`, `pull_request`, `schedule`등이 있음

```yaml
on:
  push:
    branches: [main]
```

- **main 브랜치**에 `push`가 발생했을 때 워크플로우가 실행

### ✔ Runner (러너)

- GitHub Actions 워크플로우를 실행하는 **실제 서버**
- GitHub에서는 `Ubuntu`, `Windows`, `macOS` 환경의 러너를 제공
- **Jobs**의 `runs-on`에서 설정
- 그 외 OS나 특정 하드웨어 스펙에서 실행하고 싶거나, GitHub과 독립적인 환경에서 실행하고 싶을 때는 Self-Hosted Runner를 사용할 수 있음

### ✔ Job (작업)

**Runner**(서버)에서 실행되는 단위 작업, **여러 Job**은 병렬 혹은 순차적으로 실행할 수 있음

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Run build script
        run: ./build.sh
```

- `build`: **Job**의 이름, `build`, `test`, `deploy` 등 원하는 대로 작명 가능
- `runs-on`: **Job**을 실행할 환경인 **Runner** 지정, GitHub이 제공하는 Ubuntu 리눅스 최신 버전의 가상머신 사용

| 타입                  | 예시                                        | 설명                           |
| --------------------- | ------------------------------------------- | ------------------------------ |
| GitHub-hosted runners | ubuntu-latest, windows-latest, macos-latest | GitHub이 제공하는 러너         |
| Self-hosted runners   | self-hosted                                 | 사용자가 직접 구축한 러너 서버 |

- `steps`: **Job** 내부에서 실제로 실행할 작업 단위 목록

### ✔ Step (단계)

**Job**은 여러 **Step**으로 구성되고, 각 **Step**은 순차적으로 실행, **Step**은 두 가지 중 하나임

> `uses`: Action 사용 (외부 기능 재사용)
> `run`: Shell 명령 실행

```yaml
steps:
  - name: Checkout code # 저장소 코드 다운로드
    uses: actions/checkout@v3

  - name: Install dependencies # 의존성 설치
    run: npm install

  - name: Run tests # 테스트 실행
    run: npm test
```

- 각 `-`가 하나의 **Step**
- `name`: 해당 Step이 어떤 역할을 하는지 쉽게 확인하려고 붙이는 label
- **Job**에서 설정한 Runner(서버)에 저장소 코드를 다운로드(checkout), 의존성 설치(npm install), 테스트 수행(npm test)

이제 위로 올라가, 아까는 복잡하게 느껴졌던 .yml 워크플로 코드를 다시 한 번 살펴봅시다! 🔥🔥

다음 시간에는 **CI/CD 개념**을 정리하고, **GitHub Actions**를 활용해 **Spring 애플리케이션**을 **AWS에 배포**해보겠습니다.
