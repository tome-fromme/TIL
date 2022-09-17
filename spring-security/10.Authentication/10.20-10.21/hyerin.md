# 10.20. Handling Logout
## 10.20.1. Logout Java Configuration
- WebSecurityConfigurerAdapter를 사용했다면 자동으로 로그아웃 기능을 추가함
- 디폴트로는 /logout URL에 접근했을 때 다음과 같이 사용자를 로그아웃 시킴
  - HTTP 세션 무효화
  - 관련한 모든 RememberMe 인증 제거
  - /login?logout 으로 리다이렉트


- 로그아웃 커스텀 옵션
````
protected void configure(HttpSecurity http) throws Exception {
    http
        .logout(logout -> logout                              // (1)
            .logoutUrl("/my/logout")                          // (2)
            .logoutSuccessUrl("/my/index")                    // (3)
            .logoutSuccessHandler(logoutSuccessHandler)       // (4)
            .invalidateHttpSession(true)                      // (5)
            .addLogoutHandler(logoutHandler)                  // (6)
            .deleteCookies(cookieNamesToClear)                // (7)
        )
        ...
}
````
1. 로그아웃 기능 제공. WebSecurityConfigurerAdapter를 사용하면 자동으로 적용
2. 로그아웃 일으킬 URL. 디폴트는 /logout. CSRF 방어를 사용하고 있다면(디폴트) POST 요청이어야 함
3. 로그아웃 이후 리다이렉트 할 URL. 디폴트는 /login?logout
4. 커스텀 LogoutSuccessHandler를 지정하면 logoutSuccessUrl()은 무시함
5. 로그아웃 할 때 HttpSession을 무효화시킬지 여부 나타냄. 디폴트는 true. 내부에선 SecurityContextLogoutHandler에 설정
6. LogoutHandler를 추가. SecurityContextLogoutHandler는 기본적으로 마지막 LogoutHandler로 추가됨
7. 로그아웃에 성공하면 삭제할 쿠키 이름을 여러개 지정할 수 있음. 직접 CookieClearingLogoutHandler를 추가하는 것과 동일


- 일반적으로 로그아웃 기능을 커스텀할 땐 LogoutHandler나 LogoutSuccessHandler 구현체 (혹은 둘 다) 추가하는 식으로 구현
- 스프링 시큐리티 설정 API를 사용하면 다양한 공통 시나리오 내부에서 이 핸들러 사용

## 10.20.2 Logout XML Configuration
- logout요소는 특정 URL로 이동하는 식으로 로그아웃을 지원
- 기본 로그아웃 URL은 /logout이지만 logout-url 속성에 다른 값을 설정할 수 있음

## 10.20.3 LogoutHandler
- 일반적으로 LogoutHandler 구현체는 로그아웃 처리에 함여하는 클래스를 나타냄
- 보통 필요한 clean-up 동작을 수행하기 때문에 예외를 던지면 안됨 (??)
- 스프링 시큐리티는 다양한 구현체 제공
  - PersistentTokenBasedRememberMeServices
  - TokenBasedRememberMeServices
  - CookieClearingLogoutHandler
  - CsrfLogoutHandler
  - SecurityContextLogoutHandler
  - HeaderWriterLogoutHandler
- 설정 API에 LogoutHandler 구현체를 직접 추가할 수도 있지만 내부에서 대신 LogoutHandler 구현체를 적용해주는 간단 메소드도 제공
- ex) deleteCookies() 메소드로 로그아웃에 성공하면 삭제할 쿠키를 하나 이상 지정할 수 있음 (CookieClearingLogoutHandler를 추가하는것 보다 간단)

## 10.20.4 LogoutSuccessHandler
- LogoutSuccessHandler는 로그아웃에 성공한 다음 LogoutFilter에서 호출하며 적절한 곳으로 리다이렉트 또는 포워딩시키는 용도로 사용
- 이 인터페이스는 LogoutHandler와 거의 동일하지만 예외를 던질 수 있음
- 스프링 시큐리티가 제공하는 구현체
  - SimpleUrlLogoutSuccessHandler
  - HttpStatusReturningLogoutSuccessHandler
- 직접 SimpleUrlLogoutSuccessHandler를 지정하지 않아도 logoutSuccessUrl() 메소드로 설정할 수 있음
- 이 메소드 내부에서 SimpleUrlLogoutSuccessHandler를 세팅
- 로그아웃되면 설정해둔 URL로 리다이렉트. 디폴트는 /login?logout
- REST API를 사용한다면 HttpStatusReturningLogoutSuccessHandler 추천
- 이 LogoutSuccessHandler는 로그아웃에 성공하면 URL로 리다이렉트하는 대신 반환할 HTTP 상태코드를 지정할 수 있음
- 상태코드를 지정하지 않으면 디폴트로 200 return

# 10.21. Authentication Events
- 인증에 성공하거나 실패할 때마다 각각 AuthenticationSuccessEvent와 AuthenticationFailureEvent가 발생
- 이 이벤트를 수신하려면 먼저 AuthenticationEventPublisher를 설정해야 함
- 스프링 시큐리티의 DefaultAuthenticationEventPublisher 로도 충분
````
@Bean
public AuthenticationEventPublisher authenticationEventPublisher
        (ApplicationEventPublisher applicationEventPublisher) {
    return new DefaultAuthenticationEventPublisher(applicationEventPublisher);
}
````
- 이렇게 해주면 @EventListener를 사용할 수 있음
````
@Component
public class AuthenticationEvents {
    @EventListener
    public void onSuccess(AuthenticationSuccessEvent success) {
        // ...
    }

    @EventListener
    public void onFailure(AuthenticationFailureEvent failures) {
        // ...
    }
}
````
- AuthenticationSuccessHandler, AuthenticationFailureHandler와 유사하지만 리스너는 서블릿 API와는 독립적으로 사용할 수 있음
## 10.21.1. Adding Exception Mappings
- publisher는 exception이 정확하게 일치해야 해당 이벤트를 발행
- 아래와 같은 이벤트 발생 시 AuthenticationFailureEvent 발행

| Exception | Event |
|-----------|-------|
| BadCredentialsException      | AuthenticationFailureBadCredentialsEvent |
| UsernameNotFoundException      | AuthenticationFailureBadCredentialsEvent |
| AccountExpiredException      | AuthenticationFailureExpiredEvent  |
|ProviderNotFoundException|AuthenticationFailureProviderNotFoundEvent|
|DisabledException|AuthenticationFailureDisabledEvent|
|LockedException|AuthenticationFailureLockedEvent|
|AuthenticationServiceException|AuthenticationFailureServiceExceptionEvent|
|CredentialsExpiredException|AuthenticationFailureCredentialsExpiredEvent|
|InvalidBearerTokenException|AuthenticationFailureBadCredentialsEvent|

- 예외 클래스를 상속한 클래스에선 이벤트를 발생시키지 않음
- 하위 클래스를 매핑하고 싶다면 publisher의 setAdditionalExceptionMappings 메소드로 매핑을 추가해야 함
````
@Bean
public AuthenticationEventPublisher authenticationEventPublisher
        (ApplicationEventPublisher applicationEventPublisher) {
    Map<Class<? extends AuthenticationException>,
        Class<? extends AuthenticationFailureEvent>> mapping =
            Collections.singletonMap(FooException.class, FooEvent.class);
    AuthenticationEventPublisher authenticationEventPublisher =
        new DefaultAuthenticationEventPublisher(applicationEventPublisher);
    authenticationEventPublisher.setAdditionalExceptionMappings(mapping);
    return authenticationEventPublisher;
}
````

## 10.21.2. Default Event
- AuthenticationException 이 발생할 때마다 실행할 catch-all 이벤트를 설정할 수 있음
````
@Bean
public AuthenticationEventPublisher authenticationEventPublisher
        (ApplicationEventPublisher applicationEventPublisher) {
    AuthenticationEventPublisher authenticationEventPublisher =
        new DefaultAuthenticationEventPublisher(applicationEventPublisher);
    authenticationEventPublisher.setDefaultAuthenticationFailureEvent
        (GenericAuthenticationFailureEvent.class);
    return authenticationEventPublisher;
}
````