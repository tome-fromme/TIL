# 큰그림 Servlet Security

### 필터 리뷰
클라이언트는 어플리케이션 요청을 전송하고, 컨테이너는 서블릿과 여러 필터로 구성된 필터 체인을 만들어 URI path 기반으로 `HttpServletRequest`를 처리한다. 단일 HttpServletRequest, response 처리는 최대 한개의 서블릿이 담당하지만 필터는 여러개 사용할 수 있다. 필터는 보통 아래와 같이 사용한다.

- 다운스트림의 서블릿과 여러 필터의 실행을 막는다. 이 경우엔 보통 필터에서 `HttpServletResponse`를 작성
- 다운스트림에 있는 서블릿과 여러 필터로 HttpServletRequest, response를 수정한다.

필터는 필터체인 안에 있을경우 효력을 발휘한다.
사용예제
```
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    // 전처리
    chain.doFilter(request, response); // invoke the rest of the application
    // 후처리
}
```
필터는 다운스트림에 있는 나머지 필터와 서블릿에만 영향을 주기에 필터의 실행 순서는 엄청 중요하다.

### DelegatingFilterProxy
스프링 부트는 `DelegatingFilterProxy`라는 필터 구현체로 서블릿 컨테이너의 생명주기와 스프링의 어플리케이션 컨텍스트를 연결한다. 서블릿 컨테이너는 자체 표준을 사용해서 필터를 등록할 수 있지만, 스프링이 정의하는 빈은 인식하지 못한다. `DelegatingFilterProxy`는 표준 서블릿 컨테이너 메커니즘으로 등록할 수 있으면서도, 모든 처리를 필터를 구현한 스프링 빈으로 위임해준다.

### FilterChainProxy
스프링 시큐리티는 `FilterChainProxy`로 서블릿을 지원한다. `FiterChainProxy`는 스프링 시큐리티가 제공하는 특별한 Filter로, `SecurityFilterChain`을 통해 여러 필터 인스턴스로 위임할 수 있다. `FilterChainProxy`는 빈이기에 보통 `DelegatingFilterProxy`로 감싸져 있다.

### SeucirtyFilterChain
`FilterChainProxy`가 요청에 사용할 스프링 시큐리티의 필터들을 선택할 땐 `SecurityFilterChain`을 사용한다. `SecurityFilterChain`에 있는 보안 필터들은 전형적인 빈이지만, `DelegatingFilterProxy`가 아닌 `FilterChainProxy`로 등록한다. `FilterChainProxy`을 직접 서블릿 컨테이너에 등록하거나 `DelegatingFilterProxy`에 등록하면 좋은점으로는 스프링 시큐리티가 서블릿을 지원할 수 있는 시작점이 되어준다. 따라서 서블릿에 스프링 시큐리티를 적요하다 문제를 겪을경우 `FilterChainProxy`부터 디버그 포인트를 추가해 보는 것이 좋다.

`FilterChainProxy`는 스프링 시큐리티의 중심점이기에 필수로 여겨지는 작업을 수행할 수 있다. 예를들어 `SecurityContext`를 비워 메모리 릭을 방지할 수 있다. 스프링 시큐리티의 `HttpFirewall`을 적용해서 특정 공격 유형을 방어할 수 있다.

`SecurityFilterChain`을 어떨때 실행할지 유연하게 결정이 가능하다. 서블릿 컨테이너에선 URL로만 실행할 필터들을 결정하지만 `FilterChainproxy`는 `RequestMatcher` 인터페이스를 사용하면 `HttpServletRequest`에 있는 어떤것으로도 실행 여부를 결정할 수 있다.

### Security Filters
보안 필터는 `SecurityFilterChain API`를 사용해서 `FilterChainproxy`에 추가한다. 이땐 필터의 순서가 중요하다. (보통은 순서를 알지 않아도 된다)
- 필터의 순서
    1. ChannelProcessingFilter
    1. ConcurrentSessionFilter
    1. WebAsyncManagerIntegrationFilter
    1. SecurityContextPersistenceFilter
    1. HeaderWriterFilter
    1. CorsFilter
    1. CsrfFilter
    1. LogoutFilter
    1. OAuth2AuthorizationRequestRedirectFilter
    1. Saml2WebSsoAuthenticationRequestFilter
    1. X509AuthenticationFilter
    1. AbstractPreAuthenticatedProcessingFilter
    1. CasAuthenticationFilter
    1. OAuth2LoginAuthenticationFilter
    1. Saml2WebSsoAuthenticationFilter
    1. UsernamePasswordAuthenticationFilter
    1. ConcurrentSessionFilter
    1. OpenIDAuthenticationFilter
    1. DefaultLoginPageGeneratingFilter
    1. DefaultLogoutPageGeneratingFilter
    1. DigestAuthenticationFilter
    1. BearerTokenAuthenticationFilter
    1. BasicAuthenticationFilter
    1. RequestCacheAwareFilter
    1. SecurityContextHolderAwareRequestFilter
    1. JaasApiIntegrationFilter
    1. RememberMeAuthenticationFilter
    1. AnonymousAuthenticationFilter
    1. OAuth2AuthorizationCodeGrantFilter
    1. SessionManagementFilter
    1. ExceptionTranslationFilter
    1. FilterSecurityInterceptor
    1. SwitchUserFilter

### Handling Security Exceptions
`ExceptionTrasnlationFilter`는 AccessDeniedException`을 해석하고 `AuthenticationException`을 HTTP 응답으로 바꿔준다.
- `ExceptionTrasnlationFilter`는 `FilterChainProxy`에 하나의 보안 필터로 추가된다.