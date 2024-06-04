---
layout: post
title:  "HTTPClient"
date:   2024-06-04 22:28:06 +0900
categories: spring
---

## HTTP Interface


Spring Framework를 사용하면 @HttpExchange 메서드가 있는 Java 인터페이스로 HTTP 서비스를 정의할 수 있다.

이 인터페이스를 HttpServiceProxyFactory에 전달하여 RestClient 또는 WebClient 와 같은 HTTP 클라이언트를 통해 요청을 처리하는 프록시를 생성할 수 있다. 

interface → `HttpServiceProxyFactory` → HTTP client로 요청들을 날리는 proxy를 생성

혹은 @Controller 에서 인터페이스를 구현하여 서버 요청 처리를 할 수도 있다

> **HTTP Client의 종류**
RestClient
WebClient
> 

## @HttpExchange method로 인터페이스를 생성하는 법

```bash
interface RepositoryService {

	@GetExchange("/repos/{owner}/{repo}")
	Repository getRepository(@PathVariable String owner, @PathVariable String repo);

	// more HTTP exchange methods...

}
```

저 메서드가 불리면 요청을 수행하는 proxy를 생성할 수 있다

For `RestClient`:

```bash
RestClient restClient = RestClient.builder().baseUrl("https://api.github.com/").build();
RestClientAdapter adapter = RestClientAdapter.create(restClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

RepositoryService service = factory.createClient(RepositoryService.class);
```

For `WebClient`:

```bash
WebClient webClient = WebClient.builder().baseUrl("https://api.github.com/").build();
WebClientAdapter adapter = WebClientAdapter.create(webClient);
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

RepositoryService service = factory.createClient(RepositoryService.class);
```

For `RestTemplate`:

```bash
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(new DefaultUriBuilderFactory("https://api.github.com/"));
RestTemplateAdapter adapter = RestTemplateAdapter.create(restTemplate);
HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();

RepositoryService service = factory.createClient(RepositoryService.class);
```

## RestClient, WebClient, RestTemplate의 차이

🔎를 알아보기 전에 개념을 짚고 넘어가자 !

블로킹(blocking)과 논블로킹(non-blocking)은 I/O 작업을 처리하는 두 가지 주요 방식

> **블로킹(blocking)**
요청한 작업이 완료될 때까지 대기
I/O 작업이 진행 중일 때, 스레드는 해당 작업이 완료될 때까지 차단된다
동기식 프로그래밍 모델에서 사용
> 

> **논블로킹(non-blocking)**
요청한 작업이 완료될 때까지 대기하지 않고 계속 다른 작업을 수행
I/O 작업이 진행 중일 때, 스레드는 다른 작업을 처리하거나 대기 상태
이벤트 기반의 시스템, 비동기식 프로그래밍 모델에서 주로 사용
> 

> **리액티브(Reactive) 프로그래밍**
비동기적인 이벤트 처리와 데이터 스트림에 중점을 둔 프로그래밍 패러다임
대용량 및 고성능 애플리케이션, 실시간 데이터 처리, 이벤트 기반 시스템 등 다양한 분야에서 유용
> 

### 리액티브 프로그래밍 모델의 특성

- **비동기성(Asynchrony) : I/O 작업이나 이벤트를 비동기적으로 처리**
    
    스레드가 작업을 수행하는 동안 다른 스레드가 대기하지 않고 다른 작업을 수행
    
- **데이터 스트림(Streams) : 리액티브 시스템은 데이터의 흐름을 중심으로 설계됨**
    
    이벤트 스트림이나 데이터 스트림에 대한 연산을 통해 데이터를 처리하고 반응
    
- **반응성(Responsiveness) : 리액티브 시스템은 대규모 및 복잡한 작업에도 빠르게 반응할 수 있는 능력을 갖추고 있다**
    
    비동기적인 처리와 데이터 스트림에 대한 실시간 처리를 통해 달성됨
    
- **탄력성(Resilience) : 장애 발생 시 복구할 수 있는 능력을 갖춤**
    
    에러 핸들링, 재시도 전략 등을 통해 구현됨
    

### Spring Framework의 리액티브 프로그래밍 요소

- **Webflux** : 리액티브 웹 애플리케이션을 개발하기 위한 모듈. 비동기적으로 HTTP 요청 및 응답을 처리한다
- **Project Reactor :** 리액티브 스트림 프레임워크, Publisher-Subscriber 모델을 기반으로 동작
- **WebClient :** 비동기적으로 웹 서비스와 상호작용하기 위한 클라이언트

### RestTemplate → 블로킹만 됨

스프링의 전통적인 HTTP 클라이언트

블로킹 모드에서 사용됨

동기식으로 HTTP요청을 처리함

서버 측 호출, Restful 웹 서비스와의 통합 등에 사용됨

Java의 **`java.net.HttpURLConnection`**이나 Apache HttpClient과 같은 내부 구현체를 사용하여 HTTP 요청을 수행
스프링 5.0부터는 주로 리액티브 프로그래밍 모델을 지원하는 WebClient를 사용하는 것이 권장된다

### WebClient → 리액티브만 됨

리액티브 스프링 애플리케이션에서 비동기식 HTTP 호출을 수행하는데 사용

기존의 RestTemplate보다 더 많은 유연성을 제공

리액티브 프로그래밍 모델에 적합함

논블로킹 I/O를 사용하여 요청 및 응답을 처리하므로 성능이 향상될 수 있다

### RestClient → 리액티브와 블로킹 둘 다 됨

Spring Framework 5.0에서 도입된 새로운 클라이언트

리액티브 및 블로킹 모드 지원

리액티브 스프링에서는 **`org.springframework.web.reactive.function.client.WebClient`**를 사용하여 비동기식 HTTP 요청을 처리
블로킹 스프링에서는 **`org.springframework.web.client.RestTemplate`**과 같은 블로킹 HTTP 클라이언트를 사용
리액티브 프로그래밍을 위한 WebClient와 블로킹 프로그래밍을 위한 RestTemplate의 기능을 대체하는 새로운 클라이언트

```bash
package kpring.core.auth.client

interface AuthClient {
    @PostExchange("/api/v1/validation")
    fun validateToken(
        @RequestHeader("Authorization") token: String,
    ): ResponseEntity<TokenValidationResponse>
}
```

```bash
package kpring.chat.config

@Configuration
class ClientConfig {
    @Value("\${auth.url}")
    private val authUrl: String? = null

    @Bean
    fun authClient(): AuthClient {
        val restClient = RestClient.builder()
            .baseUrl(authUrl!!)
            .build()
        val adapter = RestClientAdapter.create(restClient)
        val factory = HttpServiceProxyFactory.builderFor(adapter).build()
        return factory.createClient(AuthClient::class.java)
    }
}
```

```bash
package kpring.chat.api.v1

@RestController
@RequestMapping("/api/v1")
class ChatContoller(
    private val chatService: ChatService, val authClient: AuthClient
) {

    @PostMapping("/chat")
    fun createChat(
        @Validated @RequestBody request: CreateChatRequest, @RequestHeader("Authorization") token: String
    ): ResponseEntity<*> {
        val tokenResponse = authClient.validateToken(token)
        val userId = tokenResponse.body?.userId ?: throw GlobalException(ErrorCode.INVALID_TOKEN_BODY)

        val result = chatService.createChat(request, userId)
        return ResponseEntity.ok().body(result)
    }
}
```

원래는 authClient 객체를 Controller에 주입을 하려 해도 구현체가 없어서 되지 않는데 저 Config 파일을 설정해주면 저걸로 생성이 되어서 자동으로 주입된다!
