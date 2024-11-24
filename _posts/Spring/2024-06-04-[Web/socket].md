웹소켓 기능을 개발하면서 나름 공부한 내용을 정리해 보았습니다!
궁금하거나 부연설명이 필요한 부분은 자유롭게 댓글 달아주세요!

# 웹소켓이란?

웹소켓(WebSocket)은 클라이언트와 서버 간의 실시간 양방향 통신을 가능하게 하는 프로토콜

기존의 HTTP 통신 방식은 클라이언트가 요청을 보내면 서버가 응답하는 요청-응답 모델을 따른다. 

반면, 웹소켓은 최초 연결 이후 클라이언트와 서버 간에 지속적인 연결을 유지하면서 상호 간에 데이터를 주고받을 수 있다.

이는 실시간 기능이 필요한 애플리케이션(채팅, 실시간 알림, 게임) 등에 유용하게 사용된다.

# 웹소켓의 특징

## 양방향 통신

클라이언트와 서버 모두 메시지를 보낼 수 있다.

## 지속적인 연결

HTTP와 달리 웹소켓은 연결이 한번 성립되면 끊기지 않고 계속 유지된다

## 낮은 오버헤드

HTTP처럼 매번 요청 헤더를 보내지 않고, 한 번 연결되면 지속적으로 데이터를 주고받을 수 있어 오버헤드가 적다.

## 지연 감소(실시간 데이터 전송)

서버와 클라이언트 간의 실시간 데이터 전송이 가능하다

http처럼 핸드쉐이크하면서 연결하고 연결 해제하는 과정이 생략되어서 데이터 전송시 지연(Latency)가 크게 감소한다(빨라진다)

# 웹소켓의 초기 연결

HTTP 프로토콜을 사용해 핸드셰이크를 해서 초기 연결을 설정한다

클라이언트가 HTTP 요청 헤더에 특별한 업그레이드 요청을 추가하여 웹소켓 프로토콜로 전환을 요청한다

서버가 이 요청을 수락하면 웹소켓 연결이 성립되고 이후부터는 HTTP가 아닌 웹소켓 프로토콜로 데이터가 전송된다

### 클라이언트의 요청

```java
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### 서버의 응답

```java
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

서버는 클라이언트의 요청을 확인하고, `Sec-WebSocket-Accept` 헤더에 특별한 값을 포함해 응답하여 웹소켓 연결을 설정한다

HTTP/1.1 101 Switching Protocols 응답 코드가 이 전환을 나타낸다

# 웹소켓 통신

웹소켓 연결 이후에는 TCP 연결을 기반으로 양방향 통신이 이루어진다.

클라이언트와 서버는 서로 메시지를 교환하며, 각 메시지는 프레임(frame) 단위로 전송된다. → 프레임 종류랑 다 예시들어봐

프레임은 텍스트, 바이너리 데이터, 제어 메시지 등을 포함할 수 있다. →하나하나 설명해봐

웹소켓은 기본적으로 자체적인 프레임 구조를 사용하여 데이터를 전송하지만 STOMP와 같은 상위 레벨 프로토콜을 사용할 수도 있다.

→ STOMP 랑 SockJS 설명을 여기다 추가해줘

## 웹소켓의 메시지 형식(프레임 구조)

웹소켓은 다음과 같은 프레임 구조를 가지고 데이터를 전송한다

| **FIN** | 메시지의 끝을 나타내는 플래그 |
| --- | --- |
| **RSV1, RSV2, RSV3** | 예약된 플래그(일반적으로 0으로 설정) |
| **OPCODE** | 메시지의 유형 (텍스트 메시지(1), 바이너리 메시지(2), 연결 종료(8), 핑(9), 퐁(10) 등) |
| **마스킹 키** | 클라이언트에서 서버로 전송되는 메시지에 대해 사용된다 |
| **페이로드 길이** | 전송되는 데이터의 크기 |
| **페이로드 데이터** | 실제 전송되는 데이터(텍스트 또는 바이너리) |

```java
import java.io.ByteArrayOutputStream;
import java.nio.charset.StandardCharsets;

public class WebSocketFrameExample {
    public static void main(String[] args) {
        String message = "Hello, WebSocket!"; // 실제 전송할 데이터
        byte[] frame = createTextFrame(message);

        // 프레임 출력
        for (byte b : frame) {
            System.out.printf("%02X ", b);
        }
    }

    public static byte[] createTextFrame(String message) {
        byte[] payloadData = message.getBytes(StandardCharsets.UTF_8);
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

        // FIN(1) + OPCODE(1) 설정
        byte finRsvOpcode = (byte) 0x81; // FIN flag + 텍스트 메시지 OPCODE 0x1
        outputStream.write(finRsvOpcode);

        // 페이로드 길이 설정
        int payloadLength = payloadData.length;
        if (payloadLength <= 125) {
            outputStream.write((byte) payloadLength); // 길이가 125 이하인 경우
        } else if (payloadLength <= 65535) {
            outputStream.write((byte) 126); // 길이가 126~65535 사이인 경우
            outputStream.write((payloadLength >> 8) & 0xFF); // 16비트 길이
            outputStream.write(payloadLength & 0xFF);
        } else {
            outputStream.write((byte) 127); // 길이가 65536 이상인 경우
            for (int i = 7; i >= 0; i--) {
                outputStream.write((payloadLength >> (8 * i)) & 0xFF); // 64비트 길이
            }
        }

        // 마스킹 키 설정 (4바이트, 예제에서는 단순히 0으로 설정)
        byte[] maskingKey = {0x00, 0x00, 0x00, 0x00};
        outputStream.write(maskingKey);

        // 페이로드 데이터 설정 (마스킹 키 적용 없음)
        outputStream.write(payloadData);

        return outputStream.toByteArray();
    }
}

```

### 예시 데이터 출력

```java
"Hello, WebSocket!"
```

이 데이터를 전송한다면 다음과 같이 출력된다

```java
81 0E 00 00 00 00 48 65 6C 6C 6F 2C 20 57 65 62 53 6F 63 6B 65 74 21
```

- 81: FIN(1) + 텍스트 메시지 OPCODE(0x1)
- 0E: 페이로드 길이 (14바이트, "Hello, WebSocket!")
- 00 00 00 00: 마스킹 키 (4바이트)
- 48 65 6C 6C 6F 2C 20 57 65 62 53 6F 63 6B 65 74 21: "Hello, WebSocket!"의 UTF-8 인코딩된 값

## 웹소켓의 레이어

웹소켓은 응용 계층(Application Layer) 프로토콜로, OSI 7계층 모델에서 7계층에 속한다.

웹소켓이 TCP/IP 프로토콜 위에서 동작하며, 데이터를 어플리케이션 수준에서 처리된다는 뜻이다.

아래 Layer에 TCP (전송 계층) 프로토콜이 있어서 웹소켓 연결이 유지되는 동안에는 TCP 연결이 지속된다.

# 웹소켓의 연결 해제

### 1. 명시적 종료

클라이언트나 서버가 연결을 종료할 수 있다

일반적으로 close 프레임을 전송해서 이루어진다

이 프레임은 연결이 종료될 것임을 알리고, 종료 코드를 포함할 수 있다

```java
// 종료 프레임 예시
byte[] closeFrame = {(byte) 0x88, (byte) 0x00};  // FIN + OPCODE(8), payload length(0)
outputStream.write(closeFrame);
```

### 2. 네트워크 문제

클라이언트나 서버 간의 네트워크 연결이 끊어지면 웹소켓 연결도 끊어진다

### 3. 타임아웃

서버가 일정 시간동안 아무 메시지도 받지 못하면 연결을 종료한다
타임아웃 시간은 보통 서버 설정 파일에서 `keep-alive`나 `timeout`과 같은 속성으로 설정된다.

### 4. 프로토콜 위반

클라이언트나 서버가 웹소켓 프로토콜을 위반하면 연결이 종료될 수 있다

예 ) 잘못된 OPCODE 사용이나 유효하지 않은 프레임 형식

<aside>
❓

공부하다보니 웹소켓에서 특정 구독 주소로 메시지를 다 날리는데 그런건 내부적으로 어떻게 저장하는지 궁금해졌다

</aside>

# 웹소켓에서 SUBSCIBE한 메시지의 목적지들을 저장하는 방법 → 이거 예제코드를 하나씩 말해주라

## 1. Map을 사용하여 구독 관리

`Map` 자료구조를 활용하여, 구독 ID나 사용자 세션 ID를 키(key)로, 목적지(destination) 목록을 값(value)으로 저장
`Map<String, List<String>>` 형태로 세션 ID를 키로, 구독한 목적지들의 리스트를 값으로 저장
구독 요청이 들어오면 새로운 목적지를 추가하고, 구독 해제 요청이 들어오면 해당 목적지를 제거하는 방식으로 관리

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SubscriptionManager {
    private final Map<String, List<String>> subscriptions = new HashMap<>();

    // 구독 추가
    public void addSubscription(String sessionId, String destination) {
        subscriptions.computeIfAbsent(sessionId, k -> new ArrayList<>()).add(destination);
    }

    // 구독 해제
    public void removeSubscription(String sessionId, String destination) {
        if (subscriptions.containsKey(sessionId)) {
            subscriptions.get(sessionId).remove(destination);
            if (subscriptions.get(sessionId).isEmpty()) {
                subscriptions.remove(sessionId); // 구독 목록이 비어 있으면 제거
            }
        }
    }

    // 특정 세션의 구독 목록 조회
    public List<String> getSubscriptions(String sessionId) {
        return subscriptions.getOrDefault(sessionId, new ArrayList<>());
    }

    // 특정 목적지의 구독자 세션 ID 목록 조회
    public List<String> getSessionsForDestination(String destination) {
        List<String> sessions = new ArrayList<>();
        subscriptions.forEach((sessionId, destinations) -> {
            if (destinations.contains(destination)) {
                sessions.add(sessionId);
            }
        });
        return sessions;
    }
}

```

## 2. Spring의 SimpMessagingTemplate 사용

### SimpMessagingTemplate와 메시지 전송

`SimpMessagingTemplate`은 Spring에서 STOMP 메시지를 목적지(destination)에 따라 전송할 수 있도록 지원하는 템플릿
`SimpMessagingTemplate`은 `subscribe`된 특정 목적지로 메시지를 브로드캐스팅하는 데 활용되며, 구독 관리가 자동으로 이루어진다 
메시지 전송 시, `SimpMessagingTemplate`은 특정 목적지로 메시지를 전송하는 메서드를 제공(`convertAndSend(destination, payload)`).

메시지가 목적지에 따라 전송될 때, Spring은 이 메시지를 해당 목적지를 구독한 클라이언트에게 자동으로 전달

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue"); // 내부 메시지 브로커 활성화
        config.setApplicationDestinationPrefixes("/app"); // 메시지 라우팅용 전송 경로
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws") // 웹소켓 엔드포인트
                .setAllowedOrigins("*")
                .withSockJS(); // SockJS 폴백 사용
    }
}
```

```java
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {
    private final SimpMessagingTemplate messagingTemplate;

    public ChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // 특정 목적지에 메시지 전송
    public void sendMessageToDestination(String destination, String message) {
        messagingTemplate.convertAndSend(destination, message);
    }
}
```

### Message Broker

`enableSimpleBroker()` 메서드를 통해 내장 브로커나 외부 브로커(RabbitMQ, ActiveMQ)를 설정할 수 있다.

이 브로커는 STOMP 프레임을 처리하고, 메시지의 목적지를 기반으로 라우팅한다.
Spring의 내장 브로커는 `/topic`과 `/queue` 등의 목적지 패턴을 미리 정의하여, 브로커가 메시지 경로를 해석하고 관리하도록 지원한다.

### SimpUserRegistry를 통한 세션과 구독 관리

`SimpUserRegistry`는 각 사용자의 세션 ID, 사용자 ID, 그리고 이들이 구독한 목적지를 트래킹하는 역할을 수행한다
클라이언트가 `SUBSCRIBE` 프레임을 통해 구독을 요청하면, Spring은 `SimpUserRegistry`에 이를 저장해두고, 메시지 전송 시 이 정보를 참조하여 각 사용자에게 구독된 메시지를 자동으로 라우팅한다

```java
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.stereotype.Service;

@Service
public class SubscriptionService {
    private final SimpUserRegistry userRegistry;

    public SubscriptionService(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    public void showActiveSubscriptions() {
        userRegistry.getUsers().forEach(user -> {
            System.out.println("User: " + user.getName());
            user.getSessions().forEach(session -> {
                session.getSubscriptions().forEach(subscription ->
                    System.out.println("  Subscription: " + subscription.getDestination())
                );
            });
        });
    }
}

```

### STOMP 핸들러와 컨트롤러 연동

STOMP 핸들러는 Spring의 `@MessageMapping`을 통해 컨트롤러 메서드로 메시지를 라우팅한다

이렇게 처리된 메시지는 내부 브로커를 통해 구독된 목적지로 전달된다

이 구조 덕분에 Spring에서는 개발자가 목적지 구독을 수동으로 관리할 필요 없이 자동으로 구독과 메시지 전송이 이루어진다.

## 3. SimpUserRegistry 활용

Spring WebSocket에서는 `SimpUserRegistry`를 통해 현재 연결된 사용자와 목적지 구독 정보를 조회할 수 있다. 이를 통해 특정 사용자 세션이 연결된 목적지 목록을 조회하여 관리할 수 있으며, 별도로 추가적인 Map 저장이 필요 없다.

목적지를 구독한 사용자를 관리하고, 해당 목적지에 맞춰 메시지를 전송하는 시스템을 구성하면 효과적으로 구독 기반의 메시징 시스템을 구축할 수 있다. SockJS와 STOMP를 통해 목적지별 메시징을 구현할 때 이러한 구독 관리가 유용하게 사용된다.

추가로 프로젝트에서 실제로 STOMP와 SockJS를 썼기 때문에 확실히 짚고 넘어가고자 찾아보았다

웹소켓의 상위레벨 프로토콜로는 STOMP와 SockJS가 있다.

# STOMP : Simple Text Oriented Messaging Protocol

메시지 기반의 프로토콜

브로커 기반의 메시지 전송을 위한 표준이다

웹소켓 위에서 구동할 수 있다

주로 Spring을 사용하는 애플리케이션에서 메시징에 사용된다

STOMP FRAME을 사용해서 텍스트 형식의 명령어로 전송된다

## STOMP FRAME의 종류들

### STOMP CONNECT FRAME

클라이언트가 서버에 연결을 시작할 때 사용하는 프레임
클라이언트가 STOMP 서버에 연결을 요청하고, 서버는 `CONNECTED` 프레임으로 응답

```java
CONNECT
accept-version:1.2
host:localhost

\0

```

- accept-version: 클라이언트가 지원하는 STOMP 버전
- host: 연결할 서버의 호스트

### STOMP CONNECTED FRAME

서버가 클라이언트의 연결 요청을 수락했음을 알리는 프레임

```java

CONNECTED
version:1.2

\0
```

- version: 연결에 사용된 STOMP 버전을 표시

### SEND FRAME

클라이언트가 서버에 메시지를 전송할 때 사용

이 프레임은 일반적으로 특정 목적지로 메시지를 보낼 때 사용된다

```java
SEND
destination:/app/chat.sendMessage
content-type:application/json

{"message": "Hello, world!"}
\0
```

- destination: 메시지가 전송될 목적지(예: 채팅방)
- content-type: 메시지의 형식을 지정(JSON 데이터는 `application/json`)
- payload: 전송할 실제 메시지 내용

### SUBSCRIBE FRAME

클라이언트가 특정 목적지의 메시지를 구독하기 위해 사용

클라이언트는 서버로부터 특정 주제의 메시지를 받는다

```java
SUBSCRIBE
id:sub-0
destination:/topic/chat

\0
```

- id: 구독을 식별하는 고유 ID. 서버에서 특정 구독을 취소할 때도 사용된다.
- destination: 구독할 목적지, 예를 들어 채팅방의 주제(topic)이다.

### UNSUBSCRIBE FRAME

클라이언트가 특정 구독을 해지하기 위해 사용

```java
UNSUBSCRIBE
id:sub-0

\0
```

id: 해지할 구독의 ID. `SUBSCRIBE` 프레임에서 사용한 ID와 일치해야 한다

### DISCONNECT FRAME

클라이언트가 서버와의 연결을 종료할 때 사용하는 프레임

클라이언트가 더 이상 메시지를 주고받지 않겠다고 알리면서 연결을 안전하게 끊는 역할을 한다

```java
DISCONNECT

\0
```

### MESSAGE FRAME

서버가 클라이언트에게 구독된 목적지의 메시지를 전송할 때 사용하는 프레임

서버에서 클라이언트로 발송

```java

MESSAGE
subscription:sub-0
message-id:007
destination:/topic/chat
content-type:application/json

{"message": "New message from the chat"}
\0

```

- subscription: 클라이언트가 설정한 구독 ID
- message-id: 서버에서 발행한 메시지의 ID
- destination: 메시지가 발송된 목적지
- payload: 서버가 발송하는 메시지의 내용

### ACK (Acknowledgment) FRAME

클라이언트가 서버로부터 받은 메시지를 성공적으로 처리했음을 알리기 위해 전송

메시지의 확인 여부를 서버에 알릴 수 있다

```java
ACK
id:007

\0
```

id: 메시지의 ID. 서버 `MESSAGE` 프레임의 `message-id`와 일치해야 한다.

<aside>
❓

공부하다보니 사람들이 채팅을 받아서 몇명이 읽었는지 단톡 메시지 옆에 숫자로 표현하는것을 이걸로 하는건지 궁금해졌다

그러나 찾아보니 메시지를 안전하게 전달받았는지 여부를 확인하는 데 유용하지만, 읽은 사람의 수를 추적하는 용도로는 제한적이라 읽음 확인 기능을 구현하려면 다른 방식을 주로 사용한다고 한다.

[🧷웹소켓으로 채팅방 읽음 확인을 구현하는 방법](https://www.notion.so/f03efa9111894d83856ef4b5107ec158?pvs=21)

</aside>

### NACK (Negative Acknowledgment) FRAME

클라이언트가 서버로부터 받은 메시지를 처리하지 못했음을 알리기 위해 전송

이를 통해 서버는 메시지를 다시 처리하도록 할 수 있다.

# SockJS

SockJS는 웹소켓 폴백(fallback) 프로토콜이다.

웹소켓을 사용할 수 없는 환경에서도 비슷한 실시간 양방향 통신 기능을 제공하는 라이브러리이다

브라우저나 네트워크 설정으로 인해 웹소켓이 지원되지 않는 경우, SockJS는 HTTP를 사용해 폴백 방식으로 통신을 이어갈 수 있다.

SockJS는 주로 실시간 채팅, 알림, 협업 도구 등 실시간 연결이 필요한 애플리케이션에서 안정적인 연결을 보장하기 위해 사용된다.

## SockJS의 주요 기능

### 다양한 폴백 옵션 제공

SockJS는 웹소켓이 차단되거나 사용할 수 없는 경우 자동으로 롱 폴링(long polling), 스트리밍(streaming), XHR 등을 사용해 연결을 유지한다
이러한 폴백 방식은 실시간으로 데이터를 주고받기 위해 웹소켓과 비슷한 인터페이스를 제공

### 브라우저 및 네트워크 호환성 개선

특정 네트워크 방화벽이나 프록시 서버가 웹소켓 연결을 차단하는 경우가 많습니다. SockJS는 이를 해결하기 위해 다양한 폴백 방법을 지원하여 브라우저나 네트워크 환경의 제약을 뛰어넘어 실시간 연결을 유지할 수 있게 해준다

### 일관된 API 제공

SockJS는 웹소켓의 `send`, `close`, `onmessage`, `onclose`와 같은 API와 유사한 인터페이스를 제공해서 이를 통해 SockJS와 웹소켓을 쉽게 교체할 수 있도록 해준다

## SockJS의 작동 방식

SockJS는 웹소켓 연결이 불가능할 때 폴백 메커니즘을 통해 다음과 같은 방식으로 작동한다

### WebSocket

우선 웹소켓을 시도, 지원 가능하면 이를 사용해 연결을 유지

### XHR 스트리밍

웹소켓이 차단되면, XHR(비동기 HTTP 요청) 스트리밍 방식을 사용해 서버가 데이터를 클라이언트로 지속적으로 전송할 수 있게 한다

### Iframe 스트리밍

브라우저 호환성을 높이기 위해 iframe을 활용하여 지속적인 데이터를 전송하는 방식

### 롱 폴링

지속적으로 서버에 요청을 보내는 방식으로, 서버가 새 데이터를 보낼 준비가 되면 응답합니다. 폴링 방식은 효율적이진 않지만 대부분의 네트워크 환경에서 사용할 수 있다

## Spring에서 SockJS는 웹소켓과 STOMP와 함께 자주 사용된다

`StompEndpointRegistry`에 `withSockJS()` 메서드를 추가해 간단히 설정가능

```java

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-chat")
                .setAllowedOrigins("*")
                .withSockJS();  // SockJS 폴백 활성화
    }
}

```

위 설정을 통해 `/ws-chat` 경로로 웹소켓을 활성화하고, 브라우저나 네트워크에서 웹소켓을 지원하지 않으면 SockJS가 자동으로 폴백

## SockJS의 장점과 한계

### 장점

네트워크 환경에 상관없이 안정적인 연결을 유지할 수 있어, 웹소켓을 지원하지 않는 환경에서도 유용하게 사용할 수 있다.

### 단점

폴백 방식들은 웹소켓보다 효율성이 낮기 때문에, 실시간성이나 데이터 전송 속도 면에서 웹소켓보다 느릴 수 있다.

# 웹소켓이란?

웹소켓(WebSocket)은 클라이언트와 서버 간의 실시간 양방향 통신을 가능하게 하는 프로토콜

기존의 HTTP 통신 방식은 클라이언트가 요청을 보내면 서버가 응답하는 요청-응답 모델을 따른다.

반면, 웹소켓은 최초 연결 이후 클라이언트와 서버 간에 지속적인 연결을 유지하면서 상호 간에 데이터를 주고받을 수 있다.

이는 실시간 기능이 필요한 애플리케이션(채팅, 실시간 알림, 게임) 등에 유용하게 사용된다.

# 웹소켓의 특징

## 양방향 통신

클라이언트와 서버 모두 메시지를 보낼 수 있다.

## 지속적인 연결

HTTP와 달리 웹소켓은 연결이 한번 성립되면 끊기지 않고 계속 유지된다

## 낮은 오버헤드

HTTP처럼 매번 요청 헤더를 보내지 않고, 한 번 연결되면 지속적으로 데이터를 주고받을 수 있어 오버헤드가 적다.

## 지연 감소(실시간 데이터 전송)

서버와 클라이언트 간의 실시간 데이터 전송이 가능하다

http처럼 핸드쉐이크하면서 연결하고 연결 해제하는 과정이 생략되어서 데이터 전송시 지연(Latency)가 크게 감소한다(빨라진다)

# 웹소켓의 초기 연결

HTTP 프로토콜을 사용해 핸드셰이크를 해서 초기 연결을 설정한다

클라이언트가 HTTP 요청 헤더에 특별한 업그레이드 요청을 추가하여 웹소켓 프로토콜로 전환을 요청한다

서버가 이 요청을 수락하면 웹소켓 연결이 성립되고 이후부터는 HTTP가 아닌 웹소켓 프로토콜로 데이터가 전송된다

### 클라이언트의 요청

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

```

### 서버의 응답

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

```

서버는 클라이언트의 요청을 확인하고, Sec-WebSocket-Accept 헤더에 특별한 값을 포함해 응답하여 웹소켓 연결을 설정한다

HTTP/1.1 101 Switching Protocols 응답 코드가 이 전환을 나타낸다

# 웹소켓 통신

웹소켓 연결 이후에는 TCP 연결을 기반으로 양방향 통신이 이루어진다.

클라이언트와 서버는 서로 메시지를 교환하며, 각 메시지는 프레임(frame) 단위로 전송된다.

프레임은 텍스트, 바이너리 데이터, 제어 메시지 등을 포함할 수 있다.

웹소켓은 기본적으로 자체적인 프레임 구조를 사용하여 데이터를 전송하지만 STOMP와 같은 상위 레벨 프로토콜을 사용할 수도 있다.

## 웹소켓의 메시지 형식(프레임 구조)

웹소켓은 다음과 같은 프레임 구조를 가지고 데이터를 전송한다

FIN 메시지의 끝을 나타내는 플래그

| **RSV1, RSV2, RSV3** | 예약된 플래그(일반적으로 0으로 설정) |
| --- | --- |
| **OPCODE** | 메시지의 유형 (텍스트 메시지(1), 바이너리 메시지(2), 연결 종료(8), 핑(9), 퐁(10) 등) |
| **마스킹 키** | 클라이언트에서 서버로 전송되는 메시지에 대해 사용된다 |
| **페이로드 길이** | 전송되는 데이터의 크기 |
| **페이로드 데이터** | 실제 전송되는 데이터(텍스트 또는 바이너리) |

```
import java.io.ByteArrayOutputStream;
import java.nio.charset.StandardCharsets;

public class WebSocketFrameExample {
    public static void main(String[] args) {
        String message = "Hello, WebSocket!";// 실제 전송할 데이터byte[] frame = createTextFrame(message);

// 프레임 출력for (byte b : frame) {
            System.out.printf("%02X ", b);
        }
    }

    public static byte[] createTextFrame(String message) {
        byte[] payloadData = message.getBytes(StandardCharsets.UTF_8);
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

// FIN(1) + OPCODE(1) 설정byte finRsvOpcode = (byte) 0x81;// FIN flag + 텍스트 메시지 OPCODE 0x1
        outputStream.write(finRsvOpcode);

// 페이로드 길이 설정int payloadLength = payloadData.length;
        if (payloadLength <= 125) {
            outputStream.write((byte) payloadLength);// 길이가 125 이하인 경우
        } else if (payloadLength <= 65535) {
            outputStream.write((byte) 126);// 길이가 126~65535 사이인 경우
            outputStream.write((payloadLength >> 8) & 0xFF);// 16비트 길이
            outputStream.write(payloadLength & 0xFF);
        } else {
            outputStream.write((byte) 127);// 길이가 65536 이상인 경우for (int i = 7; i >= 0; i--) {
                outputStream.write((payloadLength >> (8 * i)) & 0xFF);// 64비트 길이
            }
        }

// 마스킹 키 설정 (4바이트, 예제에서는 단순히 0으로 설정)byte[] maskingKey = {0x00, 0x00, 0x00, 0x00};
        outputStream.write(maskingKey);

// 페이로드 데이터 설정 (마스킹 키 적용 없음)
        outputStream.write(payloadData);

        return outputStream.toByteArray();
    }
}

```

### 예시 데이터 출력

```
"Hello, WebSocket!"

```

이 데이터를 전송한다면 다음과 같이 출력된다

```
81 0E 00 00 00 00 48 65 6C 6C 6F 2C 20 57 65 62 53 6F 63 6B 65 74 21

```

- 81: FIN(1) + 텍스트 메시지 OPCODE(0x1)
- 0E: 페이로드 길이 (14바이트, "Hello, WebSocket!")
- 00 00 00 00: 마스킹 키 (4바이트)
- 48 65 6C 6C 6F 2C 20 57 65 62 53 6F 63 6B 65 74 21: "Hello, WebSocket!"의 UTF-8 인코딩된 값

## 웹소켓의 레이어

웹소켓은 응용 계층(Application Layer) 프로토콜로, OSI 7계층 모델에서 7계층에 속한다.

웹소켓이 TCP/IP 프로토콜 위에서 동작하며, 데이터를 어플리케이션 수준에서 처리된다는 뜻이다.

아래 Layer에 TCP (전송 계층) 프로토콜이 있어서 웹소켓 연결이 유지되는 동안에는 TCP 연결이 지속된다.

# 웹소켓의 연결 해제

### 1. 명시적 종료

클라이언트나 서버가 연결을 종료할 수 있다

일반적으로 close 프레임을 전송해서 이루어진다

이 프레임은 연결이 종료될 것임을 알리고, 종료 코드를 포함할 수 있다

```
// 종료 프레임 예시byte[] closeFrame = {(byte) 0x88, (byte) 0x00};// FIN + OPCODE(8), payload length(0)
outputStream.write(closeFrame);

```

### 2. 네트워크 문제

클라이언트나 서버 간의 네트워크 연결이 끊어지면 웹소켓 연결도 끊어진다

### 3. 타임아웃

서버가 일정 시간동안 아무 메시지도 받지 못하면 연결을 종료한다 타임아웃 시간은 보통 서버 설정 파일에서 keep-alive나 timeout과 같은 속성으로 설정된다.

### 4. 프로토콜 위반

클라이언트나 서버가 웹소켓 프로토콜을 위반하면 연결이 종료될 수 있다

예 ) 잘못된 OPCODE 사용이나 유효하지 않은 프레임 형식

> ❓공부하다보니 웹소켓에서 특정 구독 주소로 메시지를 다 날리는데 그런건 내부적으로 어떻게 저장하는지 궁금해졌다
> 

# 웹소켓에서 SUBSCIBE한 메시지의 목적지들을 저장하는 방법

## 1. Map을 사용하여 구독 관리

Map 자료구조를 활용하여, 구독 ID나 사용자 세션 ID를 키(key)로, 목적지(destination) 목록을 값(value)으로 저장 Map<String, List<String>> 형태로 세션 ID를 키로, 구독한 목적지들의 리스트를 값으로 저장 구독 요청이 들어오면 새로운 목적지를 추가하고, 구독 해제 요청이 들어오면 해당 목적지를 제거하는 방식으로 관리

```
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SubscriptionManager {
    private final Map<String, List<String>> subscriptions = new HashMap<>();

// 구독 추가public void addSubscription(String sessionId, String destination) {
        subscriptions.computeIfAbsent(sessionId, k -> new ArrayList<>()).add(destination);
    }

// 구독 해제public void removeSubscription(String sessionId, String destination) {
        if (subscriptions.containsKey(sessionId)) {
            subscriptions.get(sessionId).remove(destination);
            if (subscriptions.get(sessionId).isEmpty()) {
                subscriptions.remove(sessionId);// 구독 목록이 비어 있으면 제거
            }
        }
    }

// 특정 세션의 구독 목록 조회public List<String> getSubscriptions(String sessionId) {
        return subscriptions.getOrDefault(sessionId, new ArrayList<>());
    }

// 특정 목적지의 구독자 세션 ID 목록 조회public List<String> getSessionsForDestination(String destination) {
        List<String> sessions = new ArrayList<>();
        subscriptions.forEach((sessionId, destinations) -> {
            if (destinations.contains(destination)) {
                sessions.add(sessionId);
            }
        });
        return sessions;
    }
}

```

## 2. Spring의 SimpMessagingTemplate 사용

### SimpMessagingTemplate와 메시지 전송

SimpMessagingTemplate은 Spring에서 STOMP 메시지를 목적지(destination)에 따라 전송할 수 있도록 지원하는 템플릿 SimpMessagingTemplate은 subscribe된 특정 목적지로 메시지를 브로드캐스팅하는 데 활용되며, 구독 관리가 자동으로 이루어진다 메시지 전송 시, SimpMessagingTemplate은 특정 목적지로 메시지를 전송하는 메서드를 제공(convertAndSend(destination, payload)).

메시지가 목적지에 따라 전송될 때, Spring은 이 메시지를 해당 목적지를 구독한 클라이언트에게 자동으로 전달

```
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue"); // 내부 메시지 브로커 활성화
        config.setApplicationDestinationPrefixes("/app"); // 메시지 라우팅용 전송 경로
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws") // 웹소켓 엔드포인트
                .setAllowedOrigins("*")
                .withSockJS(); // SockJS 폴백 사용
    }
}

```

```
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {
    private final SimpMessagingTemplate messagingTemplate;

    public ChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

// 특정 목적지에 메시지 전송public void sendMessageToDestination(String destination, String message) {
        messagingTemplate.convertAndSend(destination, message);
    }
}

```

### Message Broker

enableSimpleBroker() 메서드를 통해 내장 브로커나 외부 브로커(RabbitMQ, ActiveMQ)를 설정할 수 있다.

이 브로커는 STOMP 프레임을 처리하고, 메시지의 목적지를 기반으로 라우팅한다. Spring의 내장 브로커는 /topic과 /queue 등의 목적지 패턴을 미리 정의하여, 브로커가 메시지 경로를 해석하고 관리하도록 지원한다.

### SimpUserRegistry를 통한 세션과 구독 관리

SimpUserRegistry는 각 사용자의 세션 ID, 사용자 ID, 그리고 이들이 구독한 목적지를 트래킹하는 역할을 수행한다 클라이언트가 SUBSCRIBE 프레임을 통해 구독을 요청하면, Spring은 SimpUserRegistry에 이를 저장해두고, 메시지 전송 시 이 정보를 참조하여 각 사용자에게 구독된 메시지를 자동으로 라우팅한다

```
import org.springframework.messaging.simp.user.SimpUserRegistry;
import org.springframework.stereotype.Service;

@Service
public class SubscriptionService {
    private final SimpUserRegistry userRegistry;

    public SubscriptionService(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    public void showActiveSubscriptions() {
        userRegistry.getUsers().forEach(user -> {
            System.out.println("User: " + user.getName());
            user.getSessions().forEach(session -> {
                session.getSubscriptions().forEach(subscription ->
                    System.out.println("  Subscription: " + subscription.getDestination())
                );
            });
        });
    }
}

```

### STOMP 핸들러와 컨트롤러 연동

STOMP 핸들러는 Spring의 @MessageMapping을 통해 컨트롤러 메서드로 메시지를 라우팅한다

이렇게 처리된 메시지는 내부 브로커를 통해 구독된 목적지로 전달된다

이 구조 덕분에 Spring에서는 개발자가 목적지 구독을 수동으로 관리할 필요 없이 자동으로 구독과 메시지 전송이 이루어진다.

## 3. SimpUserRegistry 활용

Spring WebSocket에서는 SimpUserRegistry를 통해 현재 연결된 사용자와 목적지 구독 정보를 조회할 수 있다. 이를 통해 특정 사용자 세션이 연결된 목적지 목록을 조회하여 관리할 수 있으며, 별도로 추가적인 Map 저장이 필요 없다.

목적지를 구독한 사용자를 관리하고, 해당 목적지에 맞춰 메시지를 전송하는 시스템을 구성하면 효과적으로 구독 기반의 메시징 시스템을 구축할 수 있다. SockJS와 STOMP를 통해 목적지별 메시징을 구현할 때 이러한 구독 관리가 유용하게 사용된다.

추가로 프로젝트에서 실제로 STOMP와 SockJS를 썼기 때문에 확실히 짚고 넘어가고자 찾아보았다

웹소켓의 상위레벨 프로토콜로는 STOMP와 SockJS가 있다.

# STOMP : Simple Text Oriented Messaging Protocol

메시지 기반의 프로토콜

브로커 기반의 메시지 전송을 위한 표준이다

웹소켓 위에서 구동할 수 있다

주로 Spring을 사용하는 애플리케이션에서 메시징에 사용된다

STOMP FRAME을 사용해서 텍스트 형식의 명령어로 전송된다

## STOMP FRAME의 종류들

### STOMP CONNECT FRAME

클라이언트가 서버에 연결을 시작할 때 사용하는 프레임 클라이언트가 STOMP 서버에 연결을 요청하고, 서버는 CONNECTED 프레임으로 응답

```
CONNECT
accept-version:1.2
host:localhost

\\0

```

- accept-version: 클라이언트가 지원하는 STOMP 버전
- host: 연결할 서버의 호스트

### STOMP CONNECTED FRAME

서버가 클라이언트의 연결 요청을 수락했음을 알리는 프레임

```
CONNECTED
version:1.2

\\0

```

- version: 연결에 사용된 STOMP 버전을 표시

### SEND FRAME

클라이언트가 서버에 메시지를 전송할 때 사용

이 프레임은 일반적으로 특정 목적지로 메시지를 보낼 때 사용된다

```
SEND
destination:/app/chat.sendMessage
content-type:application/json

{"message": "Hello, world!"}
\\0

```

- destination: 메시지가 전송될 목적지(예: 채팅방)
- content-type: 메시지의 형식을 지정(JSON 데이터는 application/json)
- payload: 전송할 실제 메시지 내용

### SUBSCRIBE FRAME

클라이언트가 특정 목적지의 메시지를 구독하기 위해 사용

클라이언트는 서버로부터 특정 주제의 메시지를 받는다

```
SUBSCRIBE
id:sub-0
destination:/topic/chat

\\0

```

- id: 구독을 식별하는 고유 ID. 서버에서 특정 구독을 취소할 때도 사용된다.
- destination: 구독할 목적지, 예를 들어 채팅방의 주제(topic)이다.

### UNSUBSCRIBE FRAME

클라이언트가 특정 구독을 해지하기 위해 사용

```
UNSUBSCRIBE
id:sub-0

\\0

```

id: 해지할 구독의 ID. SUBSCRIBE 프레임에서 사용한 ID와 일치해야 한다

### DISCONNECT FRAME

클라이언트가 서버와의 연결을 종료할 때 사용하는 프레임

클라이언트가 더 이상 메시지를 주고받지 않겠다고 알리면서 연결을 안전하게 끊는 역할을 한다

```
DISCONNECT

\\0

```

### MESSAGE FRAME

서버가 클라이언트에게 구독된 목적지의 메시지를 전송할 때 사용하는 프레임

서버에서 클라이언트로 발송

```
MESSAGE
subscription:sub-0
message-id:007
destination:/topic/chat
content-type:application/json

{"message": "New message from the chat"}
\\0

```

- subscription: 클라이언트가 설정한 구독 ID
- message-id: 서버에서 발행한 메시지의 ID
- destination: 메시지가 발송된 목적지
- payload: 서버가 발송하는 메시지의 내용

### ACK (Acknowledgment) FRAME

클라이언트가 서버로부터 받은 메시지를 성공적으로 처리했음을 알리기 위해 전송

메시지의 확인 여부를 서버에 알릴 수 있다

```
ACK
id:007

\\0

```

id: 메시지의 ID. 서버 MESSAGE 프레임의 message-id와 일치해야 한다.

> ❓
> 

### NACK (Negative Acknowledgment) FRAME

클라이언트가 서버로부터 받은 메시지를 처리하지 못했음을 알리기 위해 전송

이를 통해 서버는 메시지를 다시 처리하도록 할 수 있다.

# SockJS

SockJS는 웹소켓 폴백(fallback) 프로토콜이다.

웹소켓을 사용할 수 없는 환경에서도 비슷한 실시간 양방향 통신 기능을 제공하는 라이브러리이다

브라우저나 네트워크 설정으로 인해 웹소켓이 지원되지 않는 경우, SockJS는 HTTP를 사용해 폴백 방식으로 통신을 이어갈 수 있다.

SockJS는 주로 실시간 채팅, 알림, 협업 도구 등 실시간 연결이 필요한 애플리케이션에서 안정적인 연결을 보장하기 위해 사용된다.

## SockJS의 주요 기능

### 다양한 폴백 옵션 제공

SockJS는 웹소켓이 차단되거나 사용할 수 없는 경우 자동으로 롱 폴링(long polling), 스트리밍(streaming), XHR 등을 사용해 연결을 유지한다 이러한 폴백 방식은 실시간으로 데이터를 주고받기 위해 웹소켓과 비슷한 인터페이스를 제공

### 브라우저 및 네트워크 호환성 개선

특정 네트워크 방화벽이나 프록시 서버가 웹소켓 연결을 차단하는 경우가 많습니다. SockJS는 이를 해결하기 위해 다양한 폴백 방법을 지원하여 브라우저나 네트워크 환경의 제약을 뛰어넘어 실시간 연결을 유지할 수 있게 해준다

### 일관된 API 제공

SockJS는 웹소켓의 send, close, onmessage, onclose와 같은 API와 유사한 인터페이스를 제공해서 이를 통해 SockJS와 웹소켓을 쉽게 교체할 수 있도록 해준다

## SockJS의 작동 방식

SockJS는 웹소켓 연결이 불가능할 때 폴백 메커니즘을 통해 다음과 같은 방식으로 작동한다

### WebSocket

우선 웹소켓을 시도, 지원 가능하면 이를 사용해 연결을 유지

### XHR 스트리밍

웹소켓이 차단되면, XHR(비동기 HTTP 요청) 스트리밍 방식을 사용해 서버가 데이터를 클라이언트로 지속적으로 전송할 수 있게 한다

### Iframe 스트리밍

브라우저 호환성을 높이기 위해 iframe을 활용하여 지속적인 데이터를 전송하는 방식

### 롱 폴링

지속적으로 서버에 요청을 보내는 방식으로, 서버가 새 데이터를 보낼 준비가 되면 응답합니다. 폴링 방식은 효율적이진 않지만 대부분의 네트워크 환경에서 사용할 수 있다

## Spring에서 SockJS는 웹소켓과 STOMP와 함께 자주 사용된다

StompEndpointRegistry에 withSockJS() 메서드를 추가해 간단히 설정가능

```
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-chat")
                .setAllowedOrigins("*")
                .withSockJS();  // SockJS 폴백 활성화
    }
}

```

위 설정을 통해 /ws-chat 경로로 웹소켓을 활성화하고, 브라우저나 네트워크에서 웹소켓을 지원하지 않으면 SockJS가 자동으로 폴백

## SockJS의 장점과 한계

### 장점

네트워크 환경에 상관없이 안정적인 연결을 유지할 수 있어, 웹소켓을 지원하지 않는 환경에서도 유용하게 사용할 수 있다.

### 단점

폴백 방식들은 웹소켓보다 효율성이 낮기 때문에, 실시간성이나 데이터 전송 속도 면에서 웹소켓보다 느릴 수 있다.
