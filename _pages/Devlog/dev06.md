---
title: '[코인게임 프로젝트] 사용자 초대 기능'
tags:
  - Redis
  - Spring Boot
  - WebSocket
  - STOMP
  - Project
date: '2025-06-04'
thumbnail: '/assets/img/devlog/06/dev06_0.png'
---

지난 포스팅에서는 온라인 사용자 관리를 위해 `STOMP` 기반의 **WebSocket 연결 기능**을 구현한 과정을 소개했습니다. 이번 글에서는 **사용자 초대 기능**을 개발하면서 마주하게 된 `Message Broker`의 역할과 필요성에 대해 정리해보려 합니다.

# 간단하게 알아보는 STOMP + Message Broker

---

## ✔ STOMP란

**메시지 브로커를 활용**하여 쉽게 메시지를 주고 받을 수 있는 **프로토콜**

- Pub, Sub: 발신자가 메시지를 발행하면 수신자가 그것을 수신하는 메시징 패러다임
- **메시지 브로커**: 발신자의 메시지를 받아와서 수진자들에게 메시지를 전달하는 어떤 것

즉, **웹소켓 위에 얹어** 함께 사용할 수 있는 서브 프로토콜

## ✔ 메시지 브로커란

발신자(Publisher)가 보낸 메시지를 받아서, 적절한 수신자(Subscriber)에게 전달해주는 **중간 관리자** 역할
ex. RabbitMQ, Kafka, Redis Pub/Sub, Spring의 SimpleBroker

- 직접 연결 없이 발신자와 수신자를 이어줌 (비동기 통산)
- 수신자와 발신자가 동시에 연결되어 있지 않아도 메시지를 전달할 수 있음
- **다수의 수신자에게 브로드캐스트**하거나, **특정 수신자에게만 전달하는 라우팅**도 가능

# 사용자 초대 기능 개발 흐름 정리

---

### 1. 사용자 A가 사용자 B에게 게임 초대 요청 전송

```javascript
client.send('/app/game/invite', headers, JSON.stringify(inviteRequest));
```

- 사용자 A가 입력한 toUserId를 기반으로 초대 메시지 생성
- **STOMP**를 통해 서버로 메시지 전송

### 2. 서버에서 초대 요청 처리 (@MessageMapping)

```java
@MessageMapping("/game/invite")
public void handleGameInvite(@Payload GameInviteRequest request) {
    gameInviteService.sendInvite(request);
}
```

- 초대 대상(B)의 상태 확인 (WAITING, IN_GAME, OFFLINE)
- 조건에 따라:
  - `OFFLINE` or `IN_GAME`: A에게 **/queue/{fromUserId}/invite-fail** 전송
  - `WAITING` 상태: B에게 **/queue/{toUserId}/invite** 전송

### 3. 클라이언트 B가 초대 메시지를 수신하고, 수락/거절 선택

```javascript
client.subscribe(`/queue/${userId}/invite`, function (message) {
  const invite = JSON.parse(message.body);
  const accept = confirm(
    `${invite.fromNickname} 님이 초대했습니다. 수락하시겠습니까?`
  );

  const inviteResponse = {
    fromUserId: invite.fromUserId,
    toUserId: userId,
    accepted: accept,
  };

  client.send(
    '/app/game/invite/response',
    headers,
    JSON.stringify(inviteResponse)
  );
});
```

### 4. 서버에서 초대 응답 처리 (@MessageMapping)

```java
@MessageMapping("/game/invite/response")
public void handleInviteResponse(@Payload GameInviteResponse response) {
    gameInviteService.processInviteResponse(response);
}
```

- 수락한 경우: A, B 상태를 `IN_GAME`으로 변경
- A에게 초대 응답 결과 전송: **/queue/{fromUserId}/invite-response**

### 5. 클라이언트 A가 응답 결과 수신

```javascript
client.subscribe(`/queue/${userId}/invite-response`, function (message) {
  const response = JSON.parse(message.body);
  alert(response.accepted ? '상대가 수락했습니다.' : '상대가 거절했습니다.');
});
```

# 현재 방식의 한계

---

```java
registry.enableSimpleBroker("/topic", "/queue");
```

현재 사용하고 있는 `SimpleBroker`는 Spirng에서 제공하는 **인메모리 메시지 브로커**입니다.

> <img src="/assets/img/devlog/06/dev06_3.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

**단일 WAS 환경**에서는 메시지가 문제없이 해당 **subscriber**들에게 전달됩니다. 그러나 **Scale-out**으로 인해 서버가 여러 대로 분산되면 문제가 발생할 수 있습니다.

> <img src="/assets/img/devlog/06/dev06_4.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

위 그림처럼, 메시지가 **같은 서버에 연결된 subscriber에게만 전달**되는 문제가 생기며, 이는 다른 서버에 연결된 구독자는 메시지를 수신하지 못하게 됩니다.

> <img src="/assets/img/devlog/06/dev06_5.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

이러한 문제를 해결하려면, **메시징 서버를 별도**로 두고, 그 안에 **Message Broker**를 구성하는 것이 필요합니다. 대표적인 메시지 브로커로는 `RabbitMQ`, `Kafka`가 있으며, 이를 통해 여러 WAS 간 메시지를 공유할 수 있게 됩니다.

다만, 현재 개발 중인 코인 게임은 **단일 서버 환경에서 운영할 계획**이므로, 일단은 **인메모리 메시지 브로커로 충분**합니다. 향후 사용자가 증가하거나 시스템을 분산 환경으로 확장할 경우, 메시지 브로커를 도입하는 방향으로 **리팩토링을 고려**할 예정입니다.
