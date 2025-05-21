---
title: 'WebSocket + STOMP을 이용한 온라인 유저 관리 기능'
tags:
  - Redis
  - Spring Boot
  - WebSocket
  - STOMP
  - Project
date: '2025-05-21'
thumbnail: '/assets/img/devlog/05/dev05_0.png'
---

코인 게임을 개발하면서 **WebSocket + Stomp**에 대한 이해의 필요성을 인지했고, 개발하면서 공부한 내용을 정리해보는 시간을 가져보겠습니다.

# WebSocket + Stomp에 대해

---

실시간 데이터 통신이 필요한 애플리케이션에서는 **WebSocket**이 자주 활용됩니다. 하지만 **WebSocket은 단순히 양방향 연결을 위한 통신 수단**일 뿐, 메시지를 어떻게 주고받고 어떤 규칙으로 처리할지는 개발자가 직접 구현해야 합니다. 여기서 등장하는 것이 `STOMP`입니다.

## ✨ WebSocket이란?

서버와 클라이언트 간에 실시간 양방향 통신을 가능하게 해주는 프로토콜입니다. 기본적으로는 **HTTP Handshake를 통해 연결을 시작**하고, 연결 이후에는 **지속적인 TCP 소켓을 유지**하면서 데이터를 주고받습니다.

> 📌 대표적인 사용 예:
>
> - 실시간 채팅
> - 알림 시스템
> - 게임 대기방 등

하지만 **WebSocket** 자체는 메시지의 목적, 구조, 수신 대상 등을 구분하는 기능이 없습니다.

## ✨ STOMP란?

STOMP는 WebSocket 위에서 동작하는 **메시징 프로토콜**입니다. 쉽게 말하면, **WebSocket을 실시간 메시지 브로커처럼 사용**할 수 있도록 도와주는 표준 규약입니다.

- 메시지를 **destination**으로 `publish`/`send`
- SUBSCRIBE, SEND, MESSAGE, ERROR 등 프레임 기반으로 구성
- 사용자별 메시지 큐(/user/queue/), 전체 브로드캐스트(/topic/) 등 구조적 전송 가능

## ✨ STOMP 메시지 경로 구조 정리

| 경로(prefix)  | 용도                     | 송신 주체  | 수신 대상                | 예시 용도       |
| ------------- | ------------------------ | ---------- | ------------------------ | --------------- |
| `/topic`      | 전체 브로드캐스트        | 서버       | 모든 구독자              | 공지, 방송      |
| `/queue`      | 1:1 또는 내부 큐         | 서버       | 하나의 클라이언트        | (거의 안 씀)    |
| `/user/queue` | 사용자 전용 큐           | 서버       | 특정 사용자              | 알림, 개인 응답 |
| `/app`        | 클라이언트 → 서버 메시지 | 클라이언트 | 서버의 `@MessageMapping` | 요청 메시지     |

# 기능 흐름 요약

---

<img src="/assets/img/devlog/05/dev05_1.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

코인 게임의 대결 모드에서 초대 방식을 어떻게 구현할지 많은 고민을 했습니다. 최종적으로는 온라인 상태의 유저 리스트를 표시하고, 해당 유저를 초대할 수 있는 방식을 선택했습니다.
이를 위해 온라인/오프라인 상태를 실시간으로 감지할 수 있어야 했고, **WebSocket 기반 연결 방식**을 도입하게 되었습니다. **WebSocket**이 연결된 유저를 온라인으로 간주하며, 특정 유저가 접속하거나 접속을 종료할 때마다 서버는 전체 사용자에게 최신 온라인 유저 목록을 브로드캐스트하는 방식으로 구현했습니다.
이때 클라이언트 간 메시지 송수신을 더 구조적으로 관리하기 위해 **STOMP 프로토콜**을 사용했습니다. **STOMP**를 통해 **subscribe/publish** 모델을 적용하고, 사용자별 큐와 브로드캐스트 채널을 쉽게 분리하였습니다.

## ✨ WebSocket 연결 성공 시 처리 흐름

<img src="/assets/img/devlog/05/dev05_2.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

- Client: WebSocket 연결 요청 & 연결 성공시 /topic/online-users subscribe
- Server: WebSocket 연결 성공 시 userId WebSocket 세션에 저장 & Redis에 userId 추가
  > 📌 Redis에 userId를 저장하는데 WebSocket 세션에도 저장하는 이유
  > 웹소켓 Disconnect 시 해당 유저를 식별하여 Redis에서 제거하기 위함
- Server: 온라인 상태 유저 목록 변경 시 /topic/online-users subscribe한 Client 전원에게 온라인 상태 user 리스트 push

## ✨ Jwt 검증 실패 시 연결 흐름

<img src="/assets/img/devlog/05/dev05_3.png" alt="velog 404"
     style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

**STOMP CONNECT 프레임**이 수신되면 해당 프레임의 헤더에 접근할 수 있으며, 이 시점에 `ChannelInterceptor`가 실행됩니다. JWT가 유효하지 않은 경우 서버는 에러 메시지를 전송하고, 클라이언트는 이를 감지하여 **콜백 함수 내에서 WebSocket을 종료하는 처리**를 할 수 있습니다.

> 📌 이 상태는?
> WebSocket 자체는 연결된 상태지만, STOMP 세션 생성에 실패했기 때문에 더 이상 유효한 동작은 수행할 수 없는 상태입니다. 따라서, 클라이언트에서 WebSocket 종료 처리를 해줍니다.

개발 초기에는 `HandshakeInterceptor`와 `HandshakeHandler`에서 **STOMP 헤더**의 Jwt 토큰을 받아 처리하려고 했습니다. 하지만 이 두 컴포넌트는 **WebSocket** 연결 이전 단계인 **HTTP 핸드셰이크 과정에서 동작**한다는 것을 알게 되었고, STOMP 헤더에는 접근할 수 없었습니다. 결과적으로, 웹소켓 연결 이후에 실행되는 `ChannelInterceptor`에서 **STOMP 헤더**로부터 Jwt를 추출하고 인증하는 방식으로 구현하였습니다.

> 📌 **STOMP 헤더**
> WebSocket 연결이 성립된 이후에 등장하는 **프레임 기반 프로토콜**에서 사용하는 헤더

# 실제 코드

---

## ✨ Spring WebSocket + STOMP 메시징 설정

```java
@Configuration
@EnableWebSocketMessageBroker
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    private final WebSocketAuthInterceptor webSocketAuthInterceptor;
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*");
    }
    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(webSocketAuthInterceptor);
    }
}

```

`WebSocketMessageBrokerConfigurer`

- STOMP 메시지 브로커 설정을 위한 인터페이스
- STOMP 메시지의 라우팅, 엔드포인트, 브로커 설정 등 전반을 담당

`@EnableWebSocketMessageBroker`:

- STOMP 기반 메시징 기능을 활성화하는 어노테이션
- 내부적으로 메시지 브로커와 관련된 인프라가 설정됨 (@MessageMapping 등 사용 가능)

`enableSimpleBroker("/topic", "/queue")`:

- 서버 → 클라이언트로 메시지를 전달할 때 사용하는 **구독 대상 경로** 설정
- **/topic**: 브로드캐스트(다수 대상)
- **/queue**": 개인 메시지 또는 특정 유저 대상 (1:1 통신 등)

`setApplicationDestinationPrefixes("/app")`

- 클라이언트가 서버로 메시지를 보낼 때 사용하는 경로의 prefix
- ex) 클라이언트가 **/app/match/request**로 전송 → 서버의 @MessageMapping("/match/request")로 라우팅

`addEndpoint("/ws")`

- 클라이언트가 WebSocket 연결을 시도할 URL 경로 지정
- ex) SockJS("/ws") 또는 WebSocket("/ws")로 연결

## ✨ STOMP CONNECT 시 JWT 인증 처리

```java
@Component
@RequiredArgsConstructor
public class WebSocketAuthInterceptor implements ChannelInterceptor {
    private final JwtProvider jwtProvider;
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);

        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            String authHeader = accessor.getFirstNativeHeader("Authorization");
            //... Jwt 검증
        }
        return message;
    }
}
```

- Jwt 검증 성공: 메시지를 그대로 체인에 전달 -> **WebSocketEventListener.handleConnect()** 실행
- Jwt 검증 실패: STOMP Error 응답 -> 클라이언트가 웹소켓 연결 종료 처리

## ✨ 웹소켓 연결 및 해제 이벤트 처리 및 온라인 유저 관리

```java
@Component
@RequiredArgsConstructor
public class WebSocketEventListener {
    private final StringRedisTemplate redisTemplate;
    private final SimpMessageSendingOperations messagingTemplate;
    @EventListener
    public void handleConnect(SessionConnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());

        String userId = (String) accessor.getSessionAttributes().get("userId");
        if (userId != null) {
            redisTemplate.opsForSet().add("online_users", userId);
            broadcastOnlineUsers();
        }
    }
    @EventListener
    public void handleDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        String userId = (String) accessor.getSessionAttributes().get("userId");
        if (userId != null) {
            redisTemplate.opsForSet().remove("online_users", userId);
            broadcastOnlineUsers();
        }
    }
    private void broadcastOnlineUsers() {
        Set<String> users = redisTemplate.opsForSet().members("online_users");
        messagingTemplate.convertAndSend("/topic/online-users", users);
    }
}
```

`SimpMessageSendingOperations`

- STOMP 메시지를 클라이언트에게 전송할 수 있게 도와주는 Spring 유틸리티 객체

`handleConnect`

- 웹소켓 연결이 성공했을 때 Spring이 자동으로 호출하는 메서드
- `ChannelInterceptor`에서 저장한 userId를 세션에서 꺼내고 Redis의 **Set**에 추가
- 모든 클라이언트에게 온라인 유저 목록 push

`handleDisconnect`

- 웹소켓 연결이 끊겼을 때 Spring이 자동으로 호출하는 메서드
- Redis의 **Set**에서 해당 유저 제거
- 모든 클라이언트에게 온라인 유저 목록 push
