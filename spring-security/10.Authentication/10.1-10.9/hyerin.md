# 10.1. SecurityContextHolder
- 스프링 시큐리티의 인증 모델 중심은 SecurityContext를 가지고 있는 SecurityContextHolder
<img src="https://godekdls.github.io/images/springsecurity/securitycontextholder.png">
- 스프링 시큐리티로 인증한 사용자의 상세 정보 저장
- SecurityContextHolder에 어떻게 값을 넣는지는 상관하지 않음. 값이 있다면 현재 인증된 사용자 정보로 사용
- 사용자가 인증됐음을 나타내는 가장 쉬운 방법은 SecurityContextHolder를 직접 설정하는 것
````
// SecurityContextHolder 설정
SecurityContext context = SecurityContextHolder.createEmptyContext(); // (1)
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER"); // (2)
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context); // (3)
````
1. 비어있는 SecurityContext 생성. 멀티스레드 경쟁상태를 피하기 위해서는
   SecurityContextHolder.getContext().setAuthentication(authentication) 를 사용하면 안되고 새로운 SecurityContext 인스턴스를 생성해야 함
2. 새로운 Authentication 객체 생성. Authentication 구현체라면 모두 SecurityContext를 담을 수 있음
   - 프로덕션 환경에서는 UsernamePasswordAuthenticationToken(userDetails, password, authorities)
3. SecurityContextHolder에 SecurityContext를 설정해줌. 스프링 시큐리티는 이 정보를 authorization에 사용
- 인증 주체에 대한 정보를 얻으려면 SecurityContextHolder에 접근하면 됨

# 10.2. SecurityContext
- SecurityContext는 SecurityContextHolder로 접근 가능
- SecurityContext는 Authentication 객제를 가지고 있음

# 10.3. Authentication
- Authentication은 스프링 시큐리티 내에서 두가지 주요한 목적을 가짐
  - 인증에 사용할 사용자의 credential 제공. 이 상황에선 isAuthenticated()은 false 리턴
  - 현재 인증된 사용자를 나타냄. Authentication은 SecurityContext에서 가져올 수 있음
- Authentication이 포함하고 있는 것
  - principal: 사용자를 식별. username/password로 인증할때는 UserDetails의 인스턴스
  - credentials: 주로 비밀번호. 유출 방지를 위해 사용자 인증 후 비움
  - authorities: 사용자에게 부여한 권한은 GrantedAuthority로 추상화. ex) role, scope

# 10.4. GrantedAuthority
- 사용자에게 부여한 권한은 GrantedAuthority로 추상화. ex) role, scope
- GrantedAuthority는 Authentication.getAuthorities() 메소드로 얻을 수 있음
- 이 메소드는 GrantedAuthority 객체의 Collection을 제공
- GrantedAuthority는 말 그대로 인증 주체(principal)에게 부여된 권한 (권한은 보통 역할 role을 뜻함)
- user/password 인증방식 사용 시 보통 UserDetailsService가 GrantedAuthority 제공
- GrantedAuthority는 일반적으로 어플리케이션 전반에 걸친 권한을 의미

# 10.5. AuthenticationManager
- 스프링 시큐리티 필터의 인증 수행 방식을 결정하는 API
- AuthenticationManager가 리턴한 Authentication은 AuthenticationManager를 호출한 컨트롤러(스프링 시큐리티의 필터)에 의해 SecurityContextHolder에 설정됨
- 스프링 시큐리티 필터를 사용하지 않는 경우 SecurityContextHolder를 직접 설정할 수 있으며 AuthenticationManager를 사용할 필요 없음
- AuthenticationManager 구현체는 어떤 것을 사용해도 상관 없지만, 가장 많이 사용하는 구현체는 ProviderManager

# 10.6. ProviderManager
<img src="https://docs.spring.io/spring-security/site/docs/5.3.2.RELEASE/reference/html5/images/servlet/authentication/architecture/providermanager.png">
<img src="https://docs.spring.io/spring-security/site/docs/5.3.2.RELEASE/reference/html5/images/servlet/authentication/architecture/providermanager-parent.png">
<img src="https://docs.spring.io/spring-security/site/docs/5.3.2.RELEASE/reference/html5/images/servlet/authentication/architecture/providermanagers-parent.png">

# 10.7. AuthenticationProvider
- 여러개의 AuthenticationProvider는 ProviderManager에 주입될 수 있음
- 각각의 AuthenticationProvider는 각각 다른 인증을 수행
- ex) DaoAuthenticationProvider는 username/password 기반 인증을 JwtAuthenticationProvider는 JWT token 인증을 지원

# 10.8. Request Credentials with AuthenticationEntryPoint
- 클라이언트의 credential을 요청하는 HTTP 응답을 보낼 때 사용
- 클라이언트가 인증을 요청할 때 credential을 같이 보내는 경우에는 credential 요청 HTTP 응답을 만들 필요 없음
- 클라이언트가 접근권한이 없는 리소스에 인증되니 않는 요청을 보내는 경우, AuthenticationEntryPoint 구현체가 클라이언트에 credential 요청
- AuthenticationEntryPoint는 로그인페이지로 리다이렉트하거나 WWW-Authenticate 헤더로 응답하는 일 담당

# 10.9. AbstractAuthenticationProcessingFilter
