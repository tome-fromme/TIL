# 10.10. Username/Password Authentication
- 사용자 이름과 비밀번호 검증은 사용자를 인증할 떄 가장 많이 사용하는 방법 중 하나
- 스프링 시큐리티는 HttpServletRequest에서 이름과 비밀번호를 읽을 수 있는 메커니즘을 기본으로 제공
  - 폼 로그인
  - 기본 인증
  - 다이제스트 인증

## 10.10.1. Form Login
- 스프링 시큐리티는 html 폼 기반 사용자 이름/비밀번호 인증을 지원
<img src="https://godekdls.github.io/images/springsecurity/loginurlauthenticationentrypoint.png">
1. 먼저 사용자가 권한이 없는 리소스 /private 에 인증되니 않은 요청을 보냄
2. 스프링 시큐리티의 FilterSecurityInterceptor에서 AccessDeniedException을 던져 인증되지 않은 요청을 거절했음을 알림
3. 인증되지 않은 사용자이므로 ExceptionTranslationFilter에서 인증을 시작하고, 설정한 AuthenticationEntryPoint로 로그인 페이지로의 리다이렉트 응답을 전송. AunthenticationEntryPoint는 대부분 LoginUrlAuthenticationEntryPoint 인스턴스
4. 그러면 브라우저는 리다이렉트된 로그인 페이지를 요청
5. 어플리케이션에서 로그인 페이지를 렌더링

- username과 password를 제출하면 UsernamePasswordAuthenticationFilter가 이 값을 인증
- UsernamePasswordAuthenticationFilter는 AbstractAuthenticationProcessingFilter를 상속

<img src="https://godekdls.github.io/images/springsecurity/usernamepasswordauthenticationfilter.png">

1. 사용자가 username과 password를 제출하면 UsernamePasswordAuthenticationFilter는 HttpSevletRequest에서 이 값을 추출해 Authentication 유형 중 하나인 UsernamePasswordAuthenticationToken을 만듦
2. UsernamePasswordAuthenticationToken을 AuthenticationManager로 넘겨 인증. AuthenticationManager 상세 동작은 사용자 정보를 저장한 방식에 따라 다름
3. 인증에 실패
   1. SecutiryContextHolder를 비움
   2. RememverMeSevices.loginFail을 실행. remeber me를 설정하지 않았다면 아무 동작도 하지 않음
   3. AuthenticationFailureHandler를 실행
4. 인증에 성공
   1. SessionAuthenticationStrategy에 새로 로그인했음을 통보
   2. SecurityContextHolder에 Authentication을 세팅
   3. RememverMeServices.loginSuccess를 실행. remeber me를 설정하지 않았다면 아무 동작도 하지 않음
   4. ApplicationEventPublisher는 InteractiveAuthenticationSuccessEvent를 발생
   5. AuthenticationSuccessHandler를 실행. 보통 로그인 페이지로 리다이렉트 할 때는 SimplUrlAuthenticationSuccessHandler가 ExceptionTranslationFilter에 저장된 요청으로 리다이렉트

- 스프링 시큐리티에선 폼 로그인이 디폴트로 활성화
- 하지만 서블릿 기반 설정을 사용한다면 폼 기반 로그인을 명시해야 함
```
protected void configure(HttpSecurity http) {
    http
        // ...
        .formLogin(withDefaults());
}
```

- 스프링 시큐리티 설정에 로그인 페이지를 명시했다면 페이지 렌더링을 직접 구현해야 함
- 디폴트 HTML 폼 핵심 규칙
  - /login에 post 요청 보내야 함
  - CSRF 토큰을 포함해야 하며, 타임리프에서는 자동으로 추가
  - 사용자 이름은 username 파라미터로 명시해야 함
  - 비밀번호는 password 파라미터로 명시
  - HTTP 파라미터 error가 있으면 사용자가 유효한 username/password를 제공하지 못했음을 나타냄
  - HTTP 파라미터 logout이 있으면 사용자가 로그아웃에 성공한 것을 나타냄

- 스프링 MVC를 사용한다면 GET/login 요청을 직접 만든 로그인 템플릿으로 매핑하는 컨트롤러가 필요
```
@Controller
class LoginController {
    @GetMapping("/login")
    String login() {
        return "login";
    }
}
```

## 10.10.2. Basic Authentication
- 스프링 시큐리티에서 HTTP 기본 인증
<img src="https://godekdls.github.io/images/springsecurity/basicauthenticationentrypoint.png">

1. 먼저 사용자가 권한이 없는 리소스 /private에 인증되지 않은 요청을 보냄
2. 스프링 시큐리티의 FilterSecurityInterceptor에서 AccessDeniedException을 던져 인증되지 않은 요청을 거절했음을 알림
3. 인증되지 않은 사용자이므로 ExceptionTranslationFilter에서 인증을 시작. 설정한 AuthenticationEntryPoint는 basixAuthenticationEntryPoint 인스턴스로, WWW-Authentication 헤더를 전송. 이때는 클라이언트가 기존 요청을 다시 전송할 수 있으므로 RequestCache는 보통 요청을 저장하지 않는 NullRequestCache를 사용

<img src="https://godekdls.github.io/images/springsecurity/basicauthenticationfilter.png">

1. 사용자가 username과 password를 제출하면 UsernamePasswordAuthenticationFilter는 HttpServletRequest에서 이 값을 추출해 Authentication 유형 중 하나인 UsernamePasswordAuthenticationToken을 만듦
2. UsernamePasswordAuthenticationToken을 AuthenticationManger로 넘겨 인증. AuthenticationManager 상세 동작은 사용자 정보를 저장한 방식에 따라 다름
3. 인증에 실패
    1. SecutiryContextHolder를 비움
    2. RememverMeSevices.loginFail을 실행. remeber me를 설정하지 않았다면 아무 동작도 하지 않음
    3. AuthenticationFailureHandler를 실행해서 WWW-Authentication 전송을 트리거
4. 인증에 성공
   1. SecurityContextHolder에 Authentication을 세팅
   2. RememberMeServices.loginSuccess을 실행. remember me를 설정하지 않았다면 아무 동작도 하지 않음
   3. BasicAuthenticationFilter 에서 FilterChain.doFilter(request, response)를 호출해서 나머지 어플리케이션 로직을 실행

- 스프링 시큐리티에선 HTTP 기본 인증을 디폴트로 활성화
- 하지만 서블릿 기반 설정을 하나라도 사용하고 있다면 HTTP Basic을 명시해야 함
````
protected void configure(HttpSecurity http) {
    http
        // ...
        .httpBasic(withDefaults());
}
````

## 10.10.3. Digest Authentication
- 다이제스트 인증은 안전하지 않으므로 최신 어플리케이션에선 사용하지 말아야 함
- 비밀번호를 일반 텍스트나 암호화 형식 또는 MD5 형식으로 저장해야 한다는게 가장 큰 문제
- 이 저장 형식은 전부 안전하지 않음
- 그대신 다이제스트에선 지원하지 않는 단방향 적응형 비밀번호를 해시를 사용해서 credential을 저장해야 함

## 10.10.4. In-Memory Authentication
- 스프링 시큐리티의 InMemoryUserDetailManager는 메모리 기반으로 username/password를 인증하는 UserDetailsService 구현체
- InMemoryUserDetailManager는 UserDetailManager 인터페이스도 구현했기 때문에 UserDetail를 관리할 수 있음
- username/password를 읽어 인증하도록 설정하면 스프링 시큐리티는 UserDetails를 통해 인

````
@Bean
public UserDetailsService users() {
    UserDetails user = User.builder()
        .username("user")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER")
        .build();
    UserDetails admin = User.builder()
        .username("admin")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER", "ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user, admin);
}
````

## 10.10.5. JDBC Authentication
- 스프링 시큐리티의 JdbcDaoImpl은 JDBC 기반으로 username/password를 인증하는 UserDetailService 구현체
- JdbcDaoImpl을 상속한 JdbcUserDetailManager는 UserDetailManager 인터페이스도 구현했기 때문에 UserDetails를 관리할 수 있음
- username/password를 읽어 인증하도록 설정하면 스프링 시큐리티는 UserDetails를 통해 인증

## 10.10.6. UserDetails
- UserDetails는 userDetailsService가 리턴하는 값
- DaoAuthenticationProvider가 userDetails를 인증하고, 이 UserDetails를 principal로 가진 Authentication을 리턴

## 10.10.7. UserDetailService
- UserDetailServicesms DaoAutehnticationProvider가 username/password로 인증할 때 필요한 username, password와 다른 속성을 조회할 때 사용
- 스프링 시큐리티가 제공하는 UserDetailService는 인메모리와 JDBC 기반 구현체가 있음
- 커스텀 인증을 정의하려면 커스텀 UserDetailService를 빈으로 만들면 됨
````
@Bean
CustomUserDetailsService customUserDetailsService() {
    return new CustomUserDetailsService();
}
````

## 10.10.8. PasswordEncoder
- 서블릿에서 스프링 시큐리티를 사용하면 PasswordEncoder를 통합해 비밀번호를 안전하게 저장할 수 있음
- 스프링 시큐리티가 사용하는 PasswordEncoder 구현체를 커스텀하려면 PasswordEncoder 빈을 정의

## 10.10.9. DaoAutehnticationProvider
- DaoAutehnticationProvider는 UserDetailService와 PasswordEncoder로 username/password를 인증하는 AuthenticationProvider 구현체
<img src="https://godekdls.github.io/images/springsecurity/daoauthenticationprovider.png">

1. Username & Password를 조회하는 인증 filter에서 UsernamePasswordAuthenticationToken을 AuthenticationManager로 넘김. AuthenticationManager는 ProviderManager가 구현
2. 이 ProviderManager는 DaoAutehnticationProvider를 AuthenticationProvider로 사용하도록 설정되어 있음
3. DaoAutehnticationProvider는 UserDetailServices에서 UserDetails를 조회
4. 그 다음 DaoAuthenticationProvider는 이전 단계에서 얻은 UserDetails에 있는 비밀번호를 PasswordEncoder로 검증
5. 인증에 성공하면 UsernamePasswordAuthenticationToken 타입의 Authentication을 반환하며, 이 객체는 UserDetailsService가 리턴한 UserDetails를 principal로 가지고 있음. 결국에 리턴한 UsernamePasswordAuthenticationToken은 인증 filter에서 SecurityContextHolder로 세팅