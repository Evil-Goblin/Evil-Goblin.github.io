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
   - 로깅을 하기 적절하다.
2. `FilterChainProxy` 는 `SpringSecurity` 의 핵심이기 때문에 필수사항에 해당하는 작업을 수행할 수 있다.
   - 메모리 누수 방지를 위한 `SecurityContext` 정리
   - `HttpFireWall` 을 적용하여 공격으로부터 어플리케이션 보호
3. `Servlet` 컨테이너의 `Filter` 가 `URL` 만을 기반으로 호출되는 것과 달리 `RequestMatcher` 인터페이스를 사용하여 `HttpServletRequest` 의 모든 것을 기반으로 호출을 결정할 수 있다.
   - 보다 호출 시기를 유연하게 결정할 수 있다.
  ![SecurityFilterChain03](https://github.com/Evil-Goblin/spring-lecture/assets/74400861/79ff52c5-8fed-4cb7-9498-a3639e968867)

### Security Filters
- `SecurityFilterChain` 에 의해 `FilterChainProxy` 등록된다.
- 인증 , 권한 부여 등의 용도로 사용된다.
- 특정 순서로 실행되어 적시에 호출이 보장된다.
  - 인증을 수행하는 필터는 권한 부여 필터보다 먼저 호출되어야 한다.
  - 필터의 호출 순서는 `FilterOrderRegistration.java` 에서 확인할 수 있다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(Customizer.withDefaults())
            .authorizeHttpRequests(authorize -> authorize
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

}
```
- 각각의 체이닝은 다음의 필터들을 등록시킨다.
  - `HttpSecurity::csrf` : `CsrfFilter`
  - `HttpSecurity::formLogin` : `UsernamePasswordAuthenticationFilter`
  - `HttpSecurity::httpBasic` : `BasicAuthenticationFilter`
  - `HttpSecurity::authorizeHttpRequests` : `AuthorizationFilter`

- 설정된 필터들은 다음과 같은 순서로 수행된다.
  1. `CSRF` 공격으로부터 보호하기 위해 `CsrfFilter` 가 호출된다.
  2. 요청을 인증( `Authentication` ) 하기 위한 인증 필터가 호출된다. ( `UsernamePasswordAuthenticationFilter` , `BasicAuthenticationFilter` )
  3. 권한 부여를 위한 `AuthorizationFilter` 가 호출된다.

#### Printing the Security Filters
```text
INFO 13948 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Will secure any request with [
org.springframework.security.web.session.DisableEncodeUrlFilter@7ed3df3b, 
org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@465b38e6, 
org.springframework.security.web.context.SecurityContextHolderFilter@575c3e9b, 
org.springframework.security.web.header.HeaderWriterFilter@9e54c59, 
org.springframework.security.web.csrf.CsrfFilter@4ecd00b5, 
org.springframework.security.web.authentication.logout.LogoutFilter@39ffda4a, 
org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@3f598450, 
org.springframework.security.web.savedrequest.RequestCacheAwareFilter@73c3cd09, 
org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@24a2e565, 
org.springframework.security.web.authentication.AnonymousAuthenticationFilter@4b960b5b, 
org.springframework.security.web.access.ExceptionTranslationFilter@d535a3d, 
org.springframework.security.web.access.intercept.AuthorizationFilter@7102ac3e]
```
- 필터 목록은 `INFO` 레벨의 로그로 출력된다.
- 필터의 적용여부를 확인할 수 있다.
- 추가 로깅 설정을 통해 동작의 확인 또한 가능하다.

#### Adding a Custom Filter to the Filter Chain
```java
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        String tenantId = request.getHeader("X-Tenant-Id"); // 1
        boolean hasAccess = isUserAllowed(tenantId); // 2
        if (hasAccess) {
            filterChain.doFilter(request, response); // 3
            return;
        }
        throw new AccessDeniedException("Access denied"); // 4
    }

}
```
- 추가할 사용자 정의 필터를 생성한다.
  - `1` : 헤더에서 `X-Tenant-Id` 를 가져온다.
  - `2` : 허용 여부를 결정한다.
  - `3` : 허용시 다음 필터를 수행한다.
  - `4` : 불허시 `AccessDeniedException` 를 발생시킨다.

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ...
        .addFilterBefore(new TenantFilter(), AuthorizationFilter.class);
    return http.build();
}
```
- `HttpSecurity::addFilterBefore` 메소드를 통해 `AuthorizationFilter` 동작 전에 동작할 수 있도록 배치한다.
- `HttpSecurity::addFilterAfter` 메소드를 통해 특정 필터 뒤에 필터를 추가할 수 있다.
- `HttpSecurity::addFilterAt` 메소드를 통해 필터 체인의 특정 필터 위치에 필터를 추가할 수 있다.
  - 필터를 등록할 때 `@Component` 를 이용하거나 `Bean` 으로 등록하면 `SpringBoot` 에 의해 자동으로 등록이 되어버리기 때문에 `SpringSecurity` , `Container` 로서 각각 `Filter` 가 수행되어 버리기 때문에 주의 해야한다.
  - 만약 의존성 주입을 활용하고자 `Bean`으로 등록하고는 싶지만 중복 호출은 피하고 싶다면 `FilterRegistrationBean` 을 등록하고 활성화 속성을 `false` 로 설정하면 `SpringBoot Container` 에 자동 등록되지 않도록 할 수 있다.
  ```java
  @Bean
  public FilterRegistrationBean<TenantFilter> tenantFilterRegistration(TenantFilter filter) {
      FilterRegistrationBean<TenantFilter> registration = new FilterRegistrationBean<>(filter);
      registration.setEnabled(false);
      return registration;
  }
  ```
