<h3>💻 사용 언어 환경</h3>
  
    1. Java 21
    2. Springboot 3.3.1

<h1>✨ 맡은 업무 설명</h1>
<h3> 1. Gateway Service</h3>
  
     - MSA 구조에서 사용되는 Gateway Service 개발

```java
//코드 일부 수정발췌
//토큰을 헤더에 추가하기 위한 Globalfilter 설정
@Override
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		return getAccessToken(
					exchange.getRequest().getHeaders().get(HEADER_ID).getFirst())
				.flatMap(accessToken -> {
					log.debug("AccessToken ===> {}",accessToken);

					exchange.getRequest().mutate()
							.header(HttpHeaders.AUTHORIZATION, "Bearer " + accessToken)
							.build();

					log.debug("Response ===> {}",exchange.getResponse());
					return chain.filter(exchange);
				});
	}
```

```xml
#코드 일부 수정발췌
#Routing 설정

spring:
  main:
    web-application-type: reactive

  cloud:
    gateway:
      httpclient :
        ssl :
          useInsecureTrustManager : true #로컬테스트용 설정
      routes:
        - id : device-service
          uri : http://device-service:31010
          predicates :
            - Path=/v1/device/**
```
     
 -----------------      
 <h3> 2. API Adapter </h3>
    
     - 장비에게 요청을 보내 받아온 응답을 다른 서비스에게로 전달하는 모듈
     - RabbitMQ로 요청이 들어오는 것을 HTTP로 변경해주는 기능
     - 비동기 서비스를 지원하기 위한 Spring webflux 사용

```java
//코드 일부 수정발췌
//RabbitMQ 리스너로 메세지를 받아오는 메소드
@RabbitListener(queues = RabbitConfig.RECEIVE_QUEUE_NAME)
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = RabbitConfig.RECEIVE_QUEUE_NAME, durable = "true"),
            exchange = @Exchange(value = RabbitConfig.RECEIVE_EXCHANGE_NAME, type = "direct"),
            key = RabbitConfig.RECEIVE_ROUTING_KEY
    ))
    public void receiveMessage(final org.springframework.amqp.core.Message message) throws JsonProcessingException {
        byte[] payloadBytes = message.getBody();
        String uri = message.getMessageProperties().getHeader("uri");
        Payload payload = objectMapper.readValue(new String(payloadBytes, StandardCharsets.UTF_8), Payload.class);

        log.info("message => {} {}", message, payload);
        processAndForward(payload, uri);
    }


//다른 서비스로 전달하는 메소드
 public void processAndForward(Payload payload, String uri, String accessToken) {
        loadBalancedWebClientBuilder
                .build()
                .method(newMethod)
                .uri(uri) // api gateway로 가는 주소
                .headers(header -> header.setBearerAuth(accessToken))
                .contentType(MediaType.APPLICATION_JSON) // 요청의 Content-Type 설정
                .accept(MediaType.APPLICATION_JSON) // 응답의 Content-Type 설정
                .header("Token", payload.token())
                .bodyValue(payload.data())//payload를 body로
                .retrieve()
                .toEntity(Message.class)
                .publishOn(Schedulers.boundedElastic())
                .subscribe(response -> {
                    log.info("Received response: {}", response);
                    sender(response);
                }, error -> {
                    if (error instanceof WebClientResponseException ex) {
						            if(ex.getStatusCode().is4xxClientError() || ex.getStatusCode().is5xxServerError()){
                            log.warn("Received {} response: {}", ex.getStatusCode(), ex.getMessage());
                            ConvertMQ convertMQ = new ConvertMQ(ex.getHeaders(), new Message("1", ex.getMessage()));
                            rabbitTemplate.convertAndSend(RabbitConfig.SENDER_EXCHANGE_NAME, RabbitConfig.SENDER_ROUTING_KEY, convertMQ);
                        }
                    }
                });
    }

```
     
-----------------
 <h3> 3. 장비연동 Service</h3>
     
     - 장비에게 요청을 보내 받아온 응답을 다른 서비스에게로 전달하는 모듈
     - 특징 : feign 대신 RestClient사용 (Spring support 이슈)
 

```java
//RestClient를 사용하기 위한 config 파일 일부 수정발췌
 @LoadBalanced
    @Bean
    public RestClient.Builder sslRestClientBuilder(HttpComponentsClientHttpRequestFactory httpRequestFactory) {
        return RestClient.builder().apply(restClientSsl.fromBundle("nsm-ssl"));
    }

    @Bean
    public NsoClient nsoClient(RestClient.Builder sslRestClientBuilder, authService authService, RetryTemplate retryTemplate) {
        return HttpServiceProxyFactory
                .builder()
                .exchangeAdapter(RestClientAdapter.create(sslRestClientBuilder.clone()
                        .requestInterceptor(new NsoClientHttpRequestInterceptor(authService, retryTemplate))
                        .build()))
                .build()
                .createClient(DeviceClient.class);
    }
```

```java

//장비에게 직접 요청을 내리는 코드를 일부 수정 발췌
@HttpExchange("https://service")
public interface DeviceClient {

    @PostExchange("/")
    Response createDevice (@RequestHeader(LOADBALANCER_TARGET_HOST_HEADER) String host,
                           @RequestHeader(LOADBALANCER_TARGET_PORT_HEADER) int port,
                           @RequestHeader(LOADBALANCER_TARGET_IS_SECURE_HEADER) boolean secure,
                           @RequestHeader("Device-Id") String deviceId,
                           @RequestBody Map<String, clientRequest> nameList);
```



<img width="1194" alt="image" src="https://github.com/user-attachments/assets/2ab1063e-2ea4-45ea-893c-9cd44724539c">
