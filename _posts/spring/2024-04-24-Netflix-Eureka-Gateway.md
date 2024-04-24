---
title: "MSA 구성을 위한 Netflix Eureka와 Spring Cloud Gateway"
categories:
    spring
tags:
    spring
    MSA
date:
    2024-04-24 22:59:19+0900
toc:
    true
toc_sticky:
    true
---


# Spring Cloud Netflix Eureka
## Netflix Eureka란?
- 넷플릭스의 MSA로의 전환 과정에서 얻은 노하우를 Spring Cloud에 기부한 오픈 소스
- Service Discovery의 역할을 한다.
  - 마이크로 서비스의 IP와 포트 번호를 저장, 관리하는 시스템
## 구현 방법
> 아직 기초를 배우는 단계이기에 변경될 수 있음.
1. `spring-cloud-starter-netflix-eureka-server` 의존성 추가
```java
dependencies {
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

2. Main Application 파일에 `@EnableEurekaServer` 어노테이션 추가
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}

}
```
3. application.yml 설정
```yml
server:
  port: 8761

spring:
  application:
    name: eureka

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
    response-cache-update-interval-ms: 5000
    eviction-interval-timer-in-ms: 10000
```
- eureka.client.register-with-eureka
  - 레지스트리에 자신을 등록할지에 대한 여부 (디폴트 true)
- eureka.client.fetch-registry
  - 레지스트리에 있는 정보를 가져올지에 대한 여부 (디폴트 true)
- eureka.server.response-cache-update-interval-ms
  - Eureka Server의 캐싱 업데이트 주기
- eviction-interval-timer-in-ms
  - 클라이언트로부터 하트비트가 계속 수신 되는지 점검 (디폴트 60초)

# Spring Cloud Gateway
## Spring Cloud Gateway란?
- spring Cloud 생태계의 API Gateway를 제공해주는 역할
- 클라이언트의 요청을 다른 서비스 서버로 라우팅시키는 라우터의 역할
- 분산되어 존재하는 모든 마이크로 서비스의 엔드포인트를 **단일화**

## 구현 방법

### 각 마이크로 서비스 (테스트를 위한 기본 형태)
1. 의존성 추가
```java
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```
2. RestController 생성
   - 기존의 모놀리식 아키텍처와 같은 컨트롤러를 만든다.
```java
@RestController
@RequestMapping("/first-service")
@Slf4j
public class FirstServiceController {

	Environment env;

	@Autowired
	public FirstServiceController(Environment env) {
		this.env = env;
	}

	@GetMapping("/welcome")
	public String welcome() {
		return "첫번째 서비스예요";
	}

	@GetMapping("/message")
	public String message(@RequestHeader("first-request") String header) {
		System.out.println(header);
		return "메시지 서비스";
	}

	@GetMapping("/check")
	public String check(HttpServletRequest request){
		log.info("Server port = {}", request.getServerPort());
		return String.format("Hi, there. This is a message from First Service on PORT %s", env.getProperty("local.server.port"));
	}
}
```
3. application.yml 설정
```yml
server:
  port: 0

spring:
  application:
    name: first-service

eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
```
- port 번호를 0번으로 줌으로써 랜덤으로 포트 번호 할당
  - 중복을 신경쓰지 않아도 됨.
  - instance-id를 통해 eureka에서 찾음.

### 게이트웨이

1. 의존성 추가
```java
dependencies {
  implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
  implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
  compileOnly 'org.projectlombok:lombok'
  annotationProcessor 'org.projectlombok:lombok'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
  ```
1. application.yml 설정
```yml
server:
  port: 8000

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8761/eureka

spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      routes:
        - id: fisrt-service
          uri: lb://FIRST-SERVICE
          predicates:
            - Path=/first-service/**
          filters:
            - AddRequestHeader=first-request, first-request-header2
            - AddResponseHeader=first-response, first-response-header2
        - id: second-service
          uri: lb://SECOND-SERVICE
          predicates:
            - Path=/second-service/**
          filters:
            - AddRequestHeader=second-request, second-request-header2
            - AddResponseHeader=second-response, second-response-header2
```
- uri
  - lb를 통해 로드 밸런싱
- predicates
  - 주어진 요청이 조건을 충족하는지 확인하는 요소