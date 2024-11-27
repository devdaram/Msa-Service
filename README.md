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


//코드 일부 수정발췌
//Gateway에 들어온 요청을 Authorization server로 보내 인증여부 확인을 받도록 하는 부분
private Mono<String> authenticationWithUserCredentialsToken(String accountId, ServerWebExchange exchange) {
	MultiValueMap<String, String> formData = new LinkedMultiValueMap<>();
	formData.add("accountId", accountId);
	formData.add("grant_type", GRANT_TYPE); //custom client credentials grant type

	log.info("Get user uuid ===> {}", accountId);

	return sslWebClient.baseUrl(authServerPath).build()
			.post()
			.uri("/oauth2/token")
			.header(HttpHeaders.CONTENT_TYPE, "application/x-www-form-urlencoded")
			.headers(header -> header.setBasicAuth(clientId, clientSecret))
			.bodyValue(formData)
			.retrieve()
			.onStatus(status -> status.is4xxClientError() || status.is5xxServerError(), clientResponse -> { //header값이 제대로 보내지지 않았을 경우 에러처리
				ProblemDetail detail =  ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Missing required header: " + HEADER_ACCOUNT_ID);
				detail.setTitle("User account Id does not exist.");
				detail.setInstance(exchange.getRequest().getURI());

				return (getVoidMono(exchange, detail).then(Mono.error(new RuntimeException("User account Id does not exist."))));
			})
			.bodyToMono(AuthResponse.class)
			.map(AuthResponse::accessToken);
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
     - RabbitMQ 동적 queue 할당 방식 + publisher 방식으로 진행 한 경험
     - Temporal 로 진행한 경험(보상로직)
 

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
                           @RequestBody Map<String, request> nameList);


/**
* RestClinet 를 실행하기 전에 interceptor를 실행할 수 있도록 하여 인증토큰 및 재시도 로직을 구현
*/

public class SdnClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
	//인증처리
	String authToken = authService.getDeviceAuthToken(deviceId);
	HttpHeaders headers = request.getHeaders();
	headers.setBasicAuth(authToken);

	return retryTemplate.execute(context -> { //Retry 시도

			ClientHttpResponse response = null;

			try {
				response = execution.execute(request, body);

			} catch (IOException e) {
				throw new AgentProcessException(e.getMessage());
			}

			HttpStatusCode code = response.getStatusCode();

			if(code.is2xxSuccessful()){ //성공이면 바로 return
				log.info("##### Request successfully done status code ... [{}]", response.getStatusCode().value());
				return response;

			} else if (isRetryableHttpStatus(code)) { //응답코드가 retry코드이면 retry 시도 (UNAUTHORIZED, FORBIDDEN, TOO_MANY_REQUESTS)
				log.warn("##### Request Fail. status code ... [{}]", response.getStatusCode().value());
				log.warn("##### [Retry]... >>> Called RetryTemplate.");
				throw new RetryableStatusCodeException(CLIENT, code.value(), response, request.getURI()); //retry 보냄

			} else {
				log.error("AgentProcessException : {}", response);
				return response;
			}

		}, recoveryContext -> { //지정된 횟수가 종료된 후 보상로직 실행
			log.error("##### Execute retry over. recovery process begin..");
			Throwable throwable = recoveryContext.getLastThrowable();

			if (!(throwable instanceof RetryableStatusCodeException retryException)) {
				log.info("throwable class name = {}", throwable.getClass().getName());
				if(throwable instanceof RuntimeException)
					// execute 에러가 아닌 401, 403 코드에대한 인증 처리 과정에서 발생하는 에러를 Runtime 받음
					throw new AgentProcessException(throwable.getMessage());
				throw new AgentProcessException(String.format("From request execute \"%s\" unexpect error occur.", request.getURI()));
			}

			HttpStatus httpStatus = extractLastErrorStatus(retryException.getMessage());

			//retry를 시도하고도 안되면 오류 난 status message로 return
			switch (httpStatus) {
				case UNAUTHORIZED       -> throw new HttpClientErrorException(HttpStatus.UNAUTHORIZED);
				case FORBIDDEN          -> throw new HttpClientErrorException(HttpStatus.FORBIDDEN);
				case TOO_MANY_REQUESTS  -> throw new HttpClientErrorException(HttpStatus.TOO_MANY_REQUESTS);
				default -> throw new RuntimeException(String.format("Request [%s] is timed out.", request.getURI()));
			}
		});
	}

}
```

```java
/**
* controller 에서 message를 publish
*/
 @Operation(summary = "생성")
    @PostMapping("/uri")
    public APIResponse<String> createTenant(@RequestBody @Valid Request request,
                                            @RequestHeader("account") String account)  {
        
        //exchange에 메세지 발행
        messagePublisher.publishMessage(account, request);

        log.info("#### Event publisher is starting ...! ");

        //메세지가 발행되었다고 응답
        return APIResponse.from("Successfully send message");
    }


/**
* Message를 publish하는 부분
*/
public void publishMessage(String account, Request request){
	//...생략

	dynamicListenerService.startListener(user.zoneName());
	rabbitTemplate.convertAndSend(user.zoneName(), Event.of(jobId, methodName, user, request));
}


/**
* 큐 생성
*/
public void createQueueForZone(String queueName) {
	Queue queue = new Queue(queueName, true, false, false);
	Binding binding = new Binding(
			queueName,
			Binding.DestinationType.QUEUE,
			RabbitMQConfig.EXCHANGE,
			RabbitMQConfig.ROUTING_KEY,
			null
	);
	rabbitAdmin.declareQueue(queue);
	rabbitAdmin.declareBinding(binding);
	registerListenerForQueue(queueName);
}

/**
* 동적으로 생성 된 큐에 listener를 설정해주는 부분
*/
public void registerListenerForQueue(String queueName) {
	SimpleRabbitListenerEndpoint endpoint = new SimpleRabbitListenerEndpoint();

	endpoint.setId(queueName);
	endpoint.setQueueNames(queueName);
	endpoint.setMessageListener(new MessageListenerAdapter(new Object() {
		public Response handleMessage(final Event event) throws JsonProcessingException {
			log.info("Received message from dynamic queue: " + queueName + " | Message: " + event.requestBody());
			
			processMessage(event);
		}
	},  "handleMessage"));

	adapter.setMessageConverter(new Jackson2JsonMessageConverter()); // JSON 변환기 설정
	endpoint.setMessageListener(adapter);
	rabbitListenerEndpointRegistry.registerListenerContainer(endpoint, listenerContainerFactory, true);
	rabbitListenerEndpointRegistry.start();
}


/**
* 발행된 메세지를 Consume 하는 부분
*/
public void processMessage(Event event){
	try {
		SdnEntity entity = hostService.getAndUseAvailableDevice(event.user().account()); //3. 사용가능한 호스트가 있는지 확인 , 있으면 사용하겠다고 마크(DB 업데이트)
		if (entity == null)
			throw new UnavailableDeviceException(HttpStatus.NOT_FOUND, "사용 가능한 장치가 없습니다."); //사용할 수 있는 장비가 없음
		log.info("Event publishing .... : {}", event.methodName());

		thisService.createSomthing(event, AvailableDevice.of(entity.getHost(), entity.getPort()));

	}catch (Exception e){
		log.error("Error : {}", e.getMessage());
	}
}


/**
* listener start
*/
public void startListener(String listenerId) {
	MessageListenerContainer container = rabbitListenerEndpointRegistry.getListenerContainer(listenerId);
	if (container != null && !container.isRunning()) {
		container.start(); // 리스너 컨테이너 시작
		System.out.println("Listener with id " + listenerId + " started.");
	} else {
		System.out.println("Listener with id " + listenerId + " is already running or not found.");
	}
}
```


-------------
 <h3> 4. Authorization Service</h3>
     
     - MSA 내부적으로 인증절차 구조를 만들기 위한 서비스 모듈
     - Client credentials 과 Custom Client credentials 방법을 사용하였음
     
 

```java

//코드 수정 발췌
//SecurityConfig에서 custom client credential 방법을 적용하기 위해 [OAuth2AccountIdClientAuthenticationConverter] 설정하는 부분
authorizationServerConfigurer
	.tokenEndpoint(oAuth2TokenEndpointConfigurer ->
		oAuth2TokenEndpointConfigurer
				.accessTokenRequestConverter(
						new DelegatingAuthenticationConverter(Arrays.asList(
								new OAuth2AccountIdClientAuthenticationConverter(),
								new OAuth2ClientCredentialsAuthenticationConverter()
						))
				)
	);


//custom provider를 적용해주는 부분
private void addOAuth2AuthorizationCodeAuthenticationProvider(HttpSecurity http) {
	OAuth2AuthorizationService authorizationService = http.getSharedObject(OAuth2AuthorizationService.class);
	OAuth2TokenGenerator<? extends OAuth2Token> tokenGenerator = http.getSharedObject(OAuth2TokenGenerator.class);

	OAuth2AccountIdClientAuthenticationProvider userClientAuthenticationProvider =
			new OAuth2AccountIdClientAuthenticationProvider(authorizationService, tokenGenerator, accountService);

	// This will add new authentication provider in the list of existing authentication providers.
	http.authenticationProvider(userClientAuthenticationProvider);
}
```


