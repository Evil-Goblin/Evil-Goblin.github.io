---
title: AuthenticationPrincipal Annotation 을 활용한 로그인 정보 가져오기
author: Evil-Goblin
date: 2024-08-31 20:30:00 +0900
categories: [Spring, SpringSecurity]
tags: [Spring, SpringSecurity, Annotation]
---
## AuthenticationPrincipal Annotation
```java
@Target({ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AuthenticationPrincipal {
    boolean errorOnInvalidType() default false;

    String expression() default "";
}
```
- SpringSecurity 에서 제공하는 annotation 이다.
- 기본적으로는 저장된 Authentication 의 principal 을 가져온다.
- expression 을 통해서 다른 값을 가져올 수도 있다.

### AuthenticationPrincipalArgumentResolver
- AuthenticationPrincipal 어노테이션에 대응하는 ArgumentResolver 이다.

```java
public final class AuthenticationPrincipalArgumentResolver implements HandlerMethodArgumentResolver {
  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    return findMethodAnnotation(AuthenticationPrincipal.class, parameter) != null;
  }

  @Override
  public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
    Authentication authentication = this.securityContextHolderStrategy.getContext().getAuthentication();
    if (authentication == null) {
      return null;
    }
    Object principal = authentication.getPrincipal();
    AuthenticationPrincipal annotation = findMethodAnnotation(AuthenticationPrincipal.class, parameter);
    String expressionToParse = annotation.expression();
    if (StringUtils.hasLength(expressionToParse)) {
      StandardEvaluationContext context = new StandardEvaluationContext();
      context.setRootObject(principal);
      context.setVariable("this", principal);
      context.setBeanResolver(this.beanResolver);
      Expression expression = this.parser.parseExpression(expressionToParse);
      principal = expression.getValue(context);
    }
    if (principal != null && !ClassUtils.isAssignable(parameter.getParameterType(), principal.getClass())) {
      if (annotation.errorOnInvalidType()) {
        throw new ClassCastException(principal + " is not assignable to " + parameter.getParameterType());
      }
      return null;
    }
    return principal;
  }
}
```
- SecurityContext 에서 Authentication 객체를 꺼내 Principal 을 리턴한다.
- 만약 expression 이 있다면 expression 을 통해 다른 값을 꺼낼 수 있다.

### 예시
```java
public class CustomUserUserDetails extends User {
     // ...
     public CustomUser getCustomUser() {
         return customUser;
     }
 }
```
- Principal 로 넣을 UserDetails 의 커스텀 객체이다.
- expression 을 이용해 CustomUserUserDetails 가 아닌 CustomUser 을 꺼내려고 한다. 

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : customUser")
public @interface CurrentUser {
}
```
- expression 을 이용해서 CustomUser 를 꺼내도록한다.
- 만약 인증정보가 없다면 익명 사용자인 AnonymousUser 가 발급되기 때문에 CustomUser 를 가지고 있지 않아 우리가 원하는 동작을 하지 않는다.
- 때문에 익명사용자인 경우 null 을 반환하도록 한다.

```java
@Controller
public class UserController {
  @GetMapping("/hello")
  public String hello(@CurrentUser CustomUser user) {
    // ...
  }
}
```
- 받아올 매개변수에 AuthenticationPrincipal 을 메타어노테이션으로 사용하는 CurrentUser 어노테이션을 사용함으로서 객체를 받아온다.
- 하지만 익명사용자인 경우 null 이 전달될 수 있기 때문에 null 체크가 필수일 것 같다.

## 그래서 쓸만한가?
이 글을 작성한 이유가 되겠다.  
이거 결국 쓸만한가?  
ArgumentResolver 를 제공해준다 라는 느낌일 뿐 실제로 사용하기 보다는 직접 구현하는 것이 더 좋을 것 같다는 생각이 강하게 든다.  
우선 nullable 한 값이 넘어오는 것이 매우 마음에 들지 않는다.  
nullable 하지 않게 작성할 수도 있지만 익명 사용자와 같은 예외상황이 Controller 까지 넘어오는 것이 매우 아쉬운 부분이다.  
때문에 ArgumentResolve 과정에서 발생하는 예외의 경우 SpringSecurity 의 AuthenticationException 을 상속하는 커스텀 예외를 던지도록하여 AuthenticationEntryPoint 에서 통합하여 처리하는 것이 제일 깔끔하지 않을까 생각된다.  

```java
// CurrentUser.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface CurrentUser {
}

// CurrentUserArgumentResolver.java
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(CurrentUser.class);
  }

  @Override
  public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, 
                                NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    if (authentication == null || !authentication.isAuthenticated()) {
      throw new UnauthorizedException();
    }
    // ...
  }
}

// UnauthorizedAuthenticationEntryPoint.java
public class UnauthorizedAuthenticationEntryPoint implements AuthenticationEntryPoint {

  private final UnauthorizedException unauthorizedException = new UnauthorizedException();
  private final ObjectMapper objectMapper;

  @Override
  public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
    CustomAuthenticationException ex = extractAuthenticationException(authException);
    ErrorResponse errorResponse = ErrorResponse.builder()
      .code(ex.getStatusCode())
      .message(ex.getMessage())
      .validation(ex.getValidation())
      .build();

    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
    response.setStatus(ex.getStatusCode());
    objectMapper.writeValue(response.getWriter(), errorResponse);
  }

  private CustomAuthenticationException extractAuthenticationException(AuthenticationException authException) {
    if (authException instanceof CustomAuthenticationException) {
      return (CustomAuthenticationException) authException;
    }
    return unauthorizedException;
  }
}
```
- 많이 생략하였지만 이와 같이 예외 자체도 직접 관리하여 하나의 로직으로 관리될 수 있도록 하는 것이 훨씬 좋아보인다.
- 이로인해 Controller 에 넘어오는 매개변수는 신뢰할 수 있는 값만을 넘길 수 있게 된다.

## 결론
처음부터 커스텀을 만들어 사용하는 것부터 시작해서 그런지 제공해주는 기능이 아쉽게 느껴진다.  
결국 서비스에 대응되는 확장성이 중요하기 때문에 일관된 정책을 지키기 위한 커스텀이 더 좋을 것 같다.
