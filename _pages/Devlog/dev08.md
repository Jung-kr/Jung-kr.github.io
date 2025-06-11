---
title: '[코인게임 프로젝트] 실시간 게임 상태 공유 기능'
tags:
  - Redis
  - Spring Boot
  - WebSocket
  - STOMP
  - Project
date: '2025-06-11'
thumbnail: '/assets/img/devlog/06/dev06_0.png'
---

이번 포스팅에서는 **WebSocket**과 **STOMP 프로토콜**을 활용해 **실시간 게임 상태를 공유**하는 방법에 대해 다룹니다. 두 플레이어가 동시에 게임에 참여하고 상태를 주고받기 위해, 방 생성부터 메시지 발행, 그리고 게임 시작, 진행, 종료에 이르는 전 과정을 **Redis**와 **SimpleBroker** 기반의 메시지 흐름으로 구성했습니다.

# 방 생성

---

초대 수락 시, 두 플레이어가 들어갈 고유한 **게임 방**을 생성해줘야 합니다.

```text
roomId = UUID.randomUUID().toString().substring(0, 16);
```

- **Redis**에 `[roomId → playerA, playerB]` 구조로 저장하여, **roomId**에 속해 있는 player가 보낸 요청인지 검증
- 플레이어 초대 수락 시 **roomId**를 생성하여 메시지를 발행하는 로직 추가

# 실시간 게임 상태 공유

---

## 1) 게임 토픽 구독 & 게임 준비 메시지

> <img src="/assets/img/devlog/08/dev08_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 클라이언트: **서버**로부터 받은 **roomId**로 채널 구독
  - `/topic/game/{roomId}` : 게임 상태
  - `/topic/game/{roomId}/start` : 게임 시작 신호 받는 채널
  - `/topic/game/{roomId}/init` : 상대방의 초기 게임 보드 받는 채널
  - `/topic/game/{roomId}/update` : 상대방의 업데이트된 좌표를 받는 채널
  - `/topic/game/{roomId}/end` : 게임 종료 신호 받는 채널

> `RabbitMQ STOMP`, `ActiveMQ` 등 일부 브로커는 와일드카드 구독(/topic/game/\*) 지원
> `SimpleBroker`는 와일드카드 구독(/topic/game/\*) 지원하지 않음

- 클라이언트: **서버**에 `/app/game/ready`로 게임 준비 메시지 전송
- 서버:
  - 사용자의 게임 준비 상태를 **Redis**의 **game_ready:{roomId}** 키에 `Set({userId, ...})` 형태로 저장
  - 해당 **Set**의 크기가 2가 되면 `/topic/game/{roomId}/start`로 게임 시작 메시지 발행한 뒤 **Redis**에서 **Set** 삭제

## 2) 초기 게임보드 메시지

상대방의 게임보드를 나의 화면에서 보여주기 위함

> <img src="/assets/img/devlog/08/dev08_2.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- 클라이언트: **서버**에 `/app/game/init`로 초기 게임보드 메시지 전송
- 서버: `/topic/game/{roomId}/init`로 초기 게임보드 메시지 발행
- 클라이언트: **GameInitMessage**를 수신했을 때, **userId**가 본인이 아니라면 상대 게임보드 저장 및 화면 표시

```json
//GameInitMessage
{
  "roomId": "abc-123",
  "userId": "user123",
  "initialBoard": [
    [1, 5, 9, 4],
    [1, 4, ...],
    ...
  ]
}
```

## 3) 게임 업데이트 메시지

실시간으로 상대방의 게임보드를 업데이트하기 위함

> <img src="/assets/img/devlog/08/dev08_3.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 500px; max-height: 400px;">

- 클라이언트: **서버**에 `/app/game/update`로 **사용자가 부순 코인의 좌표**가 담겨있는 게임 업데이트 메시지 전송

```json
//GameUpdateMessage
{
  "roomId": "abc-123",
  "userId": "user123",
  "positions": [
    [0, 5],
    [1, 4]
  ]
}
```

- 서버: `/topic/game/{roomId}/update`로 게임 업데이트 메시지 발행
- 클라이언트: **GameUpdateMessage**를 수신했을 때, **userId**가 본인이 아니라면 상대 게임보드 업데이트

## 4) 게임 종료 메시지

게임 종료 및 점수 저장

> <img src="/assets/img/devlog/08/dev08_4.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 500px; max-height: 400px;">

- 클라이언트: 게임 종료 시 **서버**에 `/app/game/end`로 게임 종료 메시지 전송
- 서버: `/topic/game/{roomId}/end`로 게임 종료 메시지 발행
- 클라이언트: **GameEndMessage**를 수신 후, 점수 기록 및 후처리
