---
title: SpringSecurity Architecture
author: Evil-Goblin
date: 2023-10-21 23:22:00 +0900
categories: [Spring, SpringSecurity]
tags: [Spring, SpringSecurity, Architecture]
---
## SpringSecurity Architecture Doc 정리

### SpringSecurity 는 크게 Authentication(인증) , Authorization(허가) , Protection Against Exploits(보호) 로 나뉜다.

### SpringSecurity 의 Servlet에 대한 지원은 Servlet Filter를 기반으로 한다.
- `Filter` 는 요청이 `Servlet` 에 도착하기 전에 동작한다.
  - `HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러`
- `Spring MVC` 는 `Servlet` 의 `Front Controller` 인 `Dispatcher Servlet` 위에서 동작한다.
  - `Filter` 는 `Spring MVC` 보다 먼저 동작한다.

## SpringSecurity 의 구성
### DelegatingFilterProxy
- `Servlet` 컨테이너는 자체 표준을 이용해 `Filter` 를 등록할 수 있지만 `Spring Bean` 을 통해 정의된 `Filter` 들은 등록되지 않는다.
- `DelegatingFilterProxy` 는 `Spring Context` 에 등록된 `Filter` `Bean` 을 실행시킨다.
  ![DelegatingFilterProxy01](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/7e1d65ec-117f-4d8e-82ba-4fea21b6a4b8)
  _Spring Context 가져온다._
  ![DelegatingFilterProxy02](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/29d8ca39-4556-46d1-9faa-e2b3d071b1a9)
  _Spring Context 에 등록된 Filter Bean 을 초기화 시킨다._
  ![DelegatingFilterProxy03](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/cc2f6a84-4ebb-4608-995d-39982c2ef44f)
  _초기화된 Filter Bean 을 실행시킨다. ( FilterChainProxy )_

### FilterChainProxy
- `SpringSecurity` 의 `Servlet` 에 대한 지원은 `FilterChainProxy` 를 통해 이루어진다.
- `SecurityFilterChain` 으로 등록한 `Filter` `Bean` 들을 실행시켜주는 역할을 한다.
- `Bean` 으로 등록되기 때문에 `DelegatingFilterProxy` 를 통해서 실행된다.
  ![FilterChainProxy01](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/7b87d6dc-efab-47bf-8aca-1dabaea90733)
  ![FilterChainProxy02](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/44bd9510-1f67-436d-9f8d-768629116b22)
  _ApplicationFilterChain 이 실행된 이후 DelegatingFilterProxy → FilterChainProxy 에 의해 SpringSecurity의 Filter 들이 실행된다._

### SecurityFilterChain
- `FilterChainProxy` 에 의해 실행되는 `Filter` `Bean` 이다.
- `SecurityFilterChain` 은 `Filter` 들의 리스트를 가진다.
- `Security Filter` 들은 `Bean` 으로 등록되지만 `DelegatingFilterProxy` 를 통해 실행되지 않는다.
  ![SecurityFilterChain01](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/d6af0484-694d-4f5a-bcbc-c8af7a6e728f)
  ![SecurityFilterChain02](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/48b76957-832c-44f1-8ee3-5ca93c7fba96)
#### SecurityFilter 가 Servlet 또는 DelegatingFilterProxy 를 통해 수행되지 않고 FilterChainProxy 를 통해 수행될 때의 장점
1. `Servlet` 의 최상단에서 수행될 수 있다.
   1. 로깅을 하기 적절하다.
2. `FilterChainProxy` 는 `SpringSecurity` 의 핵심이기 때문에 필수사항에 해당하는 작업을 수행할 수 있다.
   1. 메모리 누수 방지를 위한 `SecurityContext` 정리
   2. `HttpFireWall` 을 적용하여 공격으로부터 어플리케이션 보호
3. `Servlet` 컨테이너의 `Filter` 가 `URL` 만을 기반으로 호출되는 것과 달리 `RequestMatcher` 인터페이스를 사용하여 `HttpServletRequest` 의 모든 것을 기반으로 호출을 결정할 수 있다.
   1. 보다 호출 시기를 유연하게 결정할 수 있다.
  ![SecurityFilterChain03](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/79ff52c5-8fed-4cb7-9498-a3639e968867)
