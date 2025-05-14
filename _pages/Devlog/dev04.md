---
title: 'Refresh Token, Redis로 안전하게 관리하기'
tags:
  - Redis
  - Spring Boot
  - Refresh Token
  - Project
date: '2025-05-14'
thumbnail: '/assets/img/devlog/04/dev04_0.png'
---

최근 코인 게임이라는 실시간 대결 기반 게임의 백엔드 개발 요청을 받았습니다. 온라인 유저 간의 실시간 대결, 상태 동기화, 초대 기능 등 기술적으로 성장할 수 있는 부분이 많다고 판단하여 수락하게 되었습니다.

> https://coingame0.netlify.app/

실시간 데이터 처리와 확장성을 고려한 백엔드 아키텍처 설계 과정을 공유하기에 앞서, **Refresh Token**을 **Redis**에 저장하고 처리한 방법을 먼저 소개하고자 합니다.

# 🔒 Refresh Token이란?

---

<img src="/assets/img/devlog/04/dev04_1.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

사실 하나의 토큰만으로도 사용자 인증에는 큰 문제가 없습니다. 하지만 토큰은 클라이언트가 보관하기 때문에 **만료 시간을 길게 설정하면 보안에 취약**해지고, 반대로 짧게 설정하면 **사용자는 자주 로그인해야 하는 불편**을 겪게 됩니다. 이러한 단점을 보완하기 위해 등장한 것이 `Refresh Token`입니다.

| 항목      | Acess Token         | Refresh Token       |
| --------- | ------------------- | ------------------- |
| 사용 목적 | 사용자 인증         | Access Token 재발금 |
| 저장 위치 | 쿠키, 로컬 스토리지 | 서버, Redis         |
| 만료 시간 | 짧음                | 김                  |

`Refresh Token`은 만료 시간이 긴 토큰으로, `Access Token`**이 만료되었을 때** 사용자가 다시 로그인하지 않아도 `Access Token`을 재발급받을 수 있도록 도와주는 역할을 합니다.

## 사용흐름

<img src="/assets/img/devlog/04/dev04_2.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 사용자가 로그인하면, 서버는 `Access Token`과 `Refresh Token`을 함께 생성하여 반환
- 서버는 `Refresh Token`을 `Redis`에 **{key, value}** 형식으로 저장하고 사용자에게 반환
- 사용자는 `Access Token`은 로컬스토리지, `Refresh Token`은 **HTTP-only 쿠키**에 저장
  <br>

<img src="/assets/img/devlog/04/dev04_3.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 사용자가 만료된 `Access Token`으로 요청을 전송하면, 서버는 `401 Unauthorized`응답 반환
- 클라이언트는 이 응답을 감지하고 `Refresh Token`을 포함해 `Access Token` 재발급 요청 전송
- 서버는 `Redis`에 저장된 `Refresh Token`과 전송받은 토큰을 비교하여 일치하면 `Access Token`을 재발급

# 🔒 Refresh Token을 왜 Redis에 저장할까?

---

`Redis`는 데이터를 메모리에 저장하는 **In-Memory 방식**의 데이터베이스입니다.

> Redis 특징
>
> - Key-Value의 형태의 데이터베이스이기 때문에 적은 메모리로도 데이터를 저장할 수 있음
> - 인메모리 DB 방식으로 빠르게 접근이 가능
> - 캐시처럼 데이터 만료일을 정할 수 있음

`Redis`는 데이터를 메모리에 저장하는 `In-Memory 방식`의 데이터베이스입니다. 일반적인 `RDB`와는 다르게 데이터의 만료 시간을 설정할 수 있는 **TTL(Time-To-Live)** 기능이 있어, `Refresh Token`의 만료 시간과 동일하게 설정하면 토큰이 만료되었을 때 **Redis에서도 자동으로 삭제되어 리소스를 효율적으로 관리**할 수 있습니다.

또한 `Refresh Token`은 `Access Token`보다 수명이 길고, `Access Token` 재발급을 위해 자주 사용되기 때문에 **호출 빈도가 높습니다.** 이런 특성상, 매번 `RDB`를 거치는 것보다 메모리 기반의 Redis에 저장하여 빠르게 조회하는 것이 **속도 측면에서 유리**합니다.

# 🔒 Refresh Token 저장 및 처리

---

## Docker로 Redis 실행

최근에는, `Redis`를 로컬에 직접 설치하기 보다는 `Docker`로 띄우는 방식이 가장 보편적이라고 합니다. 우선 `build.gradle`에 **dependency**를 추가해줍니다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

application.yml 설정을 해줍니다.

```yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

만약 `Docker`가 다른 네트워크에서 돌아간다면 host에 **실제 컨테이너 IP** or **Docker 네트워크 이름**을 넣어야 합니다. **Redis 이미지**를 다운로드하겠습니다.

```bash
$ docker pull redis
```

- `Docker Hub`에서 **공식 Redis 이미지**를 찾고 로컬에 다운로드
- 해당 이미지는 이후 `docker run` 명령을 통해 컨테이너로 실행

```bash
$ docker run --name redis -p 6379:6379 -d redis
```

- `-p 6379:6379`: 내 컴퓨터 포트 : Docker 컨테이너 내부 포트
  - 컨테이너 내부 6379번 포트로 통신하는 Redis 서버를, 내 컴퓨터의 6379번 포트로 외부에 노출
- `-d`: 백그라운드 실행
- `redis`: 컨테이너 이름을 redis로 지정

<img src="/assets/img/devlog/04/dev04_4.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

위의 명령어로 실행 중인 컨테이너를 확인할 수 있습니다. `Docker Desktop`을 종료하면 당연히 컨테이너도 종료됩니다.

## Refresh Token 저장을 위한 Redis 설정과 적용

### Lettuce vs Jedis

**Spring Data Redis**에서 사용할 수 있는 **Redis Client 구현체**는 `Lettuce`와 `Jedis`가 있습니다. `Jedis`는 별도의 설정이 필요하지만, `Lettuce`는 의존성 설정 없이 사용할 수 있기 때문에 `Lettuce`를 사용하겠습니다.

### RedisRepository vs RedisTemplate

**Spring Data Redis**에서 `Redis`에 접근하는 API 스타일 중 어떤 걸 쓸지 고르는 것입니다. `RedisRepository`는 **JPARepository**처럼 사용할 수 있습니다. 하지만 그것 외에는 실용성이나 유연성 면에서 **RedisTemplate**보다 제약이 많습니다.

예를 들어, **JPARepository**는 **TTL 설정**이 어렵고 실시간 처리에 제약이 많습니다. 저는 `Refresh Token` 외에도 실시간 로직 처리가 필요하기 때문에 **RedisTemplate**를 사용하겠습니다.

### Refresh Token 저장 로직 적용하기

```java
@Service
@RequiredArgsConstructor
public class TokenService {

    private final StringRedisTemplate redisTemplate;

    public void saveRefreshToken(String key, String refreshToken, Long expiresIn) {
        redisTemplate.opsForValue().set(key, refreshToken, Duration.ofMillis(expiresIn));
    }

    public void deleteRefreshToken(String key) {
        redisTemplate.delete(key);
    }

    public String getRefreshToken(String key) {
        return redisTemplate.opsForValue().get(key);
    }
}
```

`redisTemplate`: **Spring**에서 제공하는 **Redis** 연동 객체
`opsForValue()`: **Redis**에서 **String 타입**(단순 key-value)을 조작할 수 있게 해주는 API
`Duration.ofMillis(expiresIn)`: **expiresIn** 밀리초 후에 자동 삭제

```java
@Service
@RequiredArgsConstructor
public class UserLoginService {
    //...
    private final TokenService tokenService;
    public TokenResponse login(UserLoginRequest request) {
        //...
        saveRefreshToken(user.getId(), refreshToken);
        //...
    }
    private void saveRefreshToken(String userId, String refreshToken) {
        long refreshExpiration = jwtProvider.getRefreshTokenExpiration();
        String redisKey = "refresh:" + userId;
        tokenService.saveRefreshToken(redisKey, refreshToken, refreshExpiration);
    }
}
```

이제 로그인시 `redis`에 `Refresh Token`을 저장하는 로직을 작성합니다.

<img src="/assets/img/devlog/04/dev04_5.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

로그인을 하고 `redis`에 저장된 key 목록을 확인하면 `Refresh Token`이 저장된 것을 확인할 수 있습니다. 이제 위에 소개해드린 내용을 바탕으로 추가 로직을 작성하면 됩니다.
