---
title: SpringSecurity Login 구현
author: Evil-Goblin
date: 2024-08-31 17:30:00 +0900
categories: [Spring, SpringSecurity]
tags: [Spring, SpringSecurity, Login]
---
## SpringSecurity6 에서의 로그인 구현
SpringSecurity 를 이용하여 로그인을 구현하는 방법들을 다뤄보고자 한다.  
무엇보다도 SpringSecurity5 와 달라진 부분에 대해서 다뤄볼까 한다.

### formLogin
SpringSecurity 의 설정으로 간편하게 formLogin 을 사용할 수 있다.

```java
@Configuration
public class SecurityConfig {
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
    httpSecurity
      .formLogin(form -> form
        .loginPage("/login")
        .usernameParameter("username")
        .passwordParameter("password")
        .successForwardUrl("/hello")
        .failureForwardUrl("/login?error")
        .permitAll());
    // ...
  }
}
```
- 다음과 같이 간단하게 설정이 가능하다.
- 아무 설정을 하지 않더라도 기본은 formLogin 으로 설정되며 "/login" url 에 SpringSecurity 의 기본 로그인 form 화면이 리턴된다.

```html
<form th:action="@{/login}" method="post">
    <div><label> User Name : <input type="text" name="username"/> </label></div>
    <div><label> Password: <input type="password" name="password"/> </label></div>
    <div><input type="submit" value="Sign In"/></div>
</form>
```
- usernameParameter, passwordParameter 에 해당하는 name 으로 로그인 요청을 보내도록 form 을 작성한다.

#### UsernamePasswordAuthenticationFilter
formLogin 은 `UsernamePasswordAuthenticationFilter` 에서 수행된다.

```java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
  @Override
  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
    throws AuthenticationException {
    // ...
    String username = obtainUsername(request);
    username = (username != null) ? username.trim() : "";
    String password = obtainPassword(request);
    password = (password != null) ? password : "";
    UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken.unauthenticated(username,
      password);
    // Allow subclasses to set the "details" property
    setDetails(request, authRequest);
    return this.getAuthenticationManager().authenticate(authRequest);
  }
}
```
- form 을 통해 전달받은 데이터를 기반으로 로그인을 시도한다.
- AuthenticationManager 의 동작 등은 생략하도록 하겠다.

```java
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
		Authentication authResult) throws IOException, ServletException {
	SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();
	context.setAuthentication(authResult);
	this.securityContextHolderStrategy.setContext(context);
	this.securityContextRepository.saveContext(context, request, response);
  // ...
}
```
- 중요한 부분은 로그인이 성공했다면 context 에 인증정보를 담고 SecurityContextRepository 에 context 를 저장한다는 것이다.
- httpBasic 또한 마찬가지로 로그인 성공시 context 를 생성하여 인증정보를 담고 SecurityContextRepository 에 저장한다.

### Business Code 에서 직접 구현
formLogin 은 form 을 이용하기 때문에 RequestBody 를 이용하는 경우는 formLogin 을 사용할 수 없다.  
Filter 에서는 Request 객체에 직접 접근하게 되기 때문에 RequestBody 의 내용을 직접 꺼내 인증을 수행하는 등을 수행하기 번거롭다.  
때문에 Business Code 에서 인증을 직접 구현하는 것이 더 편하다고 생각되어 구현해보게 되었다.

```java
public class LoginService {
  private final MemberService memberService;
  private final SecurityContextRepository securityContextRepository;
  public LoginResult login(LoginForm loginForm, HttpServletRequest request, HttpServletResponse response) {
    Member member = memberService.findByUsername(loginForm.getUsername());
    accountValidation(member, loginForm.getPassword());
    
    // applyAuthentication
    UserDetails userDetails = member.toUserDetails();
    Authentication token = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
    SecurityContext context = SecurityContextHolder.getContext();
    context.setAuthentication(token);
    securityContextRepository.saveContext(context, request, response);
  }
}
```
- 최대한 간략하게 휘갈겨 작성하였다.
- 중요한 것은 인가가 완료된 Authentication 토큰을 context 에 넣어주고 SecurityContextRepository 에도 직접 context 를 넣어준다는 사실이다.
- 기존 SpringSecurity5 와 다른 부분이 바로 이것이다.
- SpringSecurity5 를 사용하는 예제에서는 로그인을 직접 구현하는 경우 context 에 인가된 정보를 넣어주기만 하면 되었다.
- 무엇이 달라졌기에 SpringSecurity6 에서는 SecurityContextRepository 에 직접 넣어줘야 하는걸까?

#### SecurityContextPersistenceFilter 와 SecurityContextHolderFilter
두 필터는 모두 SecurityContext 를 관리하는 필터이다.  
SpringSecurity5 는 SecurityContextPersistenceFilter 를 사용하고, SpringSecurity6 는 SecurityContextHolderFilter 를 사용한다.  
SecurityContextRepository 로부터 context 를 로드하고 일종의 currentContext 에 넣어주는 역할을 한다.  
크게 다른 부분은 doFilter 이후의 동작이다.

```java
// SecurityContextPersistenceFilter.java
try {
  this.securityContextHolderStrategy.setContext(contextBeforeChainExecution);
  chain.doFilter(holder.getRequest(), holder.getResponse());
}
finally {
  SecurityContext contextAfterChainExecution = this.securityContextHolderStrategy.getContext();
  // Crucial removal of SecurityContextHolder contents before anything else.
  this.securityContextHolderStrategy.clearContext();
  this.repo.saveContext(contextAfterChainExecution, holder.getRequest(), holder.getResponse());
  request.removeAttribute(FILTER_APPLIED);
}
```
- finally 부분에서 세팅해줬던 context 를 비워주게 된다.
- 하지만 이전 formLogin 과 같이 SecurityContextRepository 에 save 를 해주는 로직이 있다.
- 때문에 context 에만 인가된 정보를 넣어주면 SecurityContextRepository 에 저장이 된다.

```java
// SecurityContextHolderFilter.java
try {
  this.securityContextHolderStrategy.setDeferredContext(deferredContext);
  chain.doFilter(request, response);
}
finally {
  this.securityContextHolderStrategy.clearContext();
  request.removeAttribute(FILTER_APPLIED);
}
```
- 반대로 SecurityContextHolderFilter 의 경우 finally 부분에서 SecurityContextRepository 에 저장되지 않는다.
- 때문에 직접 SecurityContextRepository 에 저장해주어야 한다.
- 명시적 저장을 위해서 SpringSecurity 설정 또한 추가해주어야 한다.

```java
@Configuration
public class SecurityConfig {
  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
    httpSecurity
      .securityContext(securityContext -> {
        securityContext.securityContextRepository(delegatingSecurityContextRepository());
        securityContext.requireExplicitSave(true);
      });
    // ...
  }
}
```
- ContextRepository 와 requireExplicitSave 를 지정하여 명시적으로 저장한다는 설정을 하여야 한다.

#### Session 에 직접 저장
Session 을 이용하여 로그인 정보를 저장하는 경우 바로 Session 에 넣어볼 수도 있을 것 같다.

```java
public class LoginService {
  private final MemberService memberService;
  private final HttpSession httpSession;
  public LoginResult login(LoginForm loginForm, HttpServletRequest request, HttpServletResponse response) {
    Member member = memberService.findByUsername(loginForm.getUsername());
    accountValidation(member, loginForm.getPassword());
    
    // applyAuthentication
    UserDetails userDetails = member.toUserDetails();
    Authentication token = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
    SecurityContext context = SecurityContextHolder.getContext();
    context.setAuthentication(token);
    httpSession.setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, context);
  }
}
```
- Session 에 직접적으로 context 를 넣어주어 구현이 가능하다.
- 하지만 이 방법은 spring-session-jdbc 와 함께 사용하게 되면 아쉬운 부분이 생길 수 있다.
  - 다른 글에서 포스팅하도록 하겠다.

#### 변경된 이유
[SpringSecurityDocs](https://docs.spring.io/spring-security/reference/6.0/migration/servlet/session-management.html)  
```text
Spring Security 5의 기본 동작은 SecurityContextPersistenceFilter 를 사용하여 보안 컨텍스트가 자동으로 보안 컨텍스트 리포지토리에 저장되도록 하는 것입니다.
저장 작업은 HttpServletResponse 가 커밋되기 직전과 SecurityContextPersistenceFilter 직전에 수행되어야 합니다.
안타깝게도 요청이 완료되기 전(즉, HttpServletResponse 를 커밋하기 직전)에 SecurityContext 의 자동 지속이 수행되면 사용자를 놀라게 할 수 있습니다.
또한 저장이 필요한지 여부를 판단하기 위해 상태를 추적하는 것도 복잡하여 때때로 SecurityContextRepository(즉, HttpSession)에 불필요한 쓰기가 발생할 수 있습니다.

Spring Security 6의 기본 동작은 SecurityContextHolderFilter 가 보안 컨텍스트 저장소에서 보안 컨텍스트만 읽고 이를 보안 컨텍스트 홀더에 채우는 것입니다.
이제 보안 컨텍스트를 요청 간에 유지하려면 사용자가 명시적으로 보안 컨텍스트를 보안 컨텍스트 리포지토리와 함께 저장해야 합니다.
이렇게 하면 필요할 때만 보안 컨텍스트 저장소(예: HttpSession)에 쓰기만 하면 되기 때문에 모호성이 제거되고 성능이 향상됩니다.
```

## 결론
- SpringSecurity6 를 이용하여 로그인을 직접 구현하는 경우 명시적으로 ContextRepository 에 저장해주어야 한다.
