---
title: Difference between SpringBootTest and WebMvcTest during SpringSecurityTest
author: Evil-Goblin
date: 2023-10-23 02:25:00 +0900
categories: [Spring, SpringSecurity]
tags: [Spring, SpringSecurity, Test, SpringBootTest, WebMvcTest]
math: true
mermaid: true
---
# SpringSecurityTest에서 SpringBootTest를 사용하는 경우와 WebMvcTest를 사용하는 경우의 차이점

## Test 환경
- `SpringBoot 3.1.3`
- `SpringSecurity 6.0.11` , `SpringSecurity 6.0.12`
- `SpringSecurity Configure`
  ```java
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
      .authorizeHttpRequests(request -> request
        .requestMatchers("/", "home").permitAll()
        .anyRequest().authenticated()
      )
      .formLogin(form -> form
        .loginPage("/login")
        .permitAll()
      )
      .logout(LogoutConfigurer::permitAll);

    return http.build();
  }
  ```

## 단순 기능의 차이
- `SpringBootTest`
  - `SpringBoot Application` 의 전체 `Context` 를 로드한다.
    - 로드하는데 시간이 걸리기 때문에 테스트가 비교적 오래걸린다.
- `WebMvcTest`
  - 웹 관련 `Bean` 만 로드된다.
    - 테스트가 비교적 빠르게 수행된다.
    - `@Controller` , `@ControllerAdvice` , `@JsonComponent` , `Filter` , `WebMvcConfigurer` , `HandlerMethodArgumentResolver` 등

## SpringSecurityTest 시 발생하는 차이점
- `WebMvcTest` 의 경우 `loginPage` 설정이 적용되지 않는다.
- 때문에 권한이 없는 페이지에 접근시 `redirect` 처리가 되지 않는다.
### WebMvcTest
![SpringSecurityTest01](https://github.com/Evil-Goblin/springio-guide/assets/74400861/33cb942d-2705-4343-9451-a49cb641c1b8){: .normal }
![SpringSecurityTest02](https://github.com/Evil-Goblin/springio-guide/assets/74400861/a44e8400-8b75-4072-8c68-99baf6e36c7d){: .normal }
### SpringBootTest
![SpringSecurityTest03](https://github.com/Evil-Goblin/springio-guide/assets/74400861/141fddd3-1499-40cb-ae61-2f65db09cce5){: .normal }
![SpringSecurityTest04](https://github.com/Evil-Goblin/springio-guide/assets/74400861/6e83754a-8c90-490c-ba96-fba5ae17ecd2){: .normal }

## 차이점 발생의 원인
- 당연하게도 `SpringBootTest` 와 `WebMvcTest` 의 차이에 있고 이미 위에 기술하였다.
- 위에 기술한 대로 `SpringBootTest` 시에는 전체 컨텍스트가 로드된다.
  - 등록한 모든 `Bean` 들이 등록된다.
  - `WebMvcTest` 는 제한적인 `Bean` 만이 등록된다.

### 권한이 없는 페이지에 대해서 `401` `status code` 를 반환하는 설정은 `httpBasic` 설정이다.
- 하지만 테스트에 사용한 설정에는 `httpBasic` 설정이 없다.
- 다시말해 `SpringBootTest` 는 직접 등록한 `SecurityFilterChain` `Bean` 을 이용하여 `SpringSecurity` 가 설정된다.
- 반대로 `WebMvcTest` 는 직접 등록한 `SecurityFilterChain` 이 아닌 `Default` 설정이 사용된다.
  ![DefaultSpringSecurityConfigure01](https://github.com/Evil-Goblin/springio-guide/assets/74400861/80349a9a-7ae6-4056-b96d-1f8a0019d886)
  _`SpringBootTest` 의 콜스택_
  ![DefaultSpringSecurityConfigure02](https://github.com/Evil-Goblin/springio-guide/assets/74400861/4d2c699e-f8b4-40c4-878b-2d0fb571e475)
  _`WebMvcTest` 의 콜스택_

## 결론
- `WebMvcTest` 는 `@EnableWebSecurity` 가 로드되지 않기 때문에 `DefaultWebSecurity` 가 로드되어 기동된다.

### 추가 정보
#### LoginPage로 redirect 또는 401 status 를 반환하는 과정
- `ExceptionTranslationFilter` 에서 `AuthenticationException` 에러가 발생했을 때 등록된 `AuthenticationEntryPoint` 를 수행한다.
  - `SecurityFilterChain` 의 설정마다 `configurer` 가 축적된다.
  - `httpBasic` 설정은 `HttpBasicConfigurer` 를 추가한다.
  - `HttpBasicConfigurer` 는 `AuthenticationEntryPoint` 으로 `HttpStatusEntryPoint` 를 가지고 있다.
  - 때문에 `httpBasic` 설정시 `HttpStatusEntryPoint` 가 등록되어 `AuthenticationException` 발생시 `401` `status code` 를 반환한다.
    ![HttpBasicConfigurer](https://github.com/Evil-Goblin/springio-guide/assets/74400861/7be6fb49-1e2f-42d5-896e-443ad8a6356e)
  - 반면 `httpBasic` 을 설정하지 않을 시 `FormLoginConfigurer` 에 의해 `LoginUrlAuthenticationEntryPoint` 가 등록된다.
    ![FormLoginConfigurer](https://github.com/Evil-Goblin/springio-guide/assets/74400861/a4f23e67-17cc-4d53-b114-fc581e014f60)
- `AuthenticationEntryPoint` 는 `commence` 메소드를 가진 `interface` 이다.
  - `AuthenticationEntryPoint` 의 구현체들은 `commence` 를 구현하여 각자의 동작을 한다.
    ![HttpStatusEntryPoint](https://github.com/Evil-Goblin/springio-guide/assets/74400861/732bab77-a5de-4b6a-8b62-bd6ea7b693d1)
    _`HttpStatusEntryPoint` 는 자신에게 설정된 `status code` 를 작성하는 동작을 한다._
    ![LoginUrlAuthenticationEntryPoint](https://github.com/Evil-Goblin/springio-guide/assets/74400861/f7640833-1d1f-4e0e-950f-5d48ac30e973)
    _`LoginUrlAuthenticationEntryPoint` 는 설정된 경로로 `redirect` 를 수행한다._
