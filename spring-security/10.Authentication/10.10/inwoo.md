## 10.10 Username/Password Authentication
사용자 이름과 비밀번호 검증은 사용자 인증할때 사용하는 방법중 하나이며, 종합적인 인증 방법을 지원한다.

1. 이름과 비밀번호를 읽는방법
 - 폼 로그인
 - 기본 인증
 - 다이제스트 인증

이름/비밀번호 조회 메커니즘은 지원하는 저장 메커니즘 중 어떤 것과도 조합하여 사용할 수 있다.

1. 인메모리 인증과 심플 스토리지
2. JDBC 인증과 관계형 데이터베이스
3. UserDetailService와 커스텀 데이터 스토어
4. LDAP 인증과 LDAP 스토리지

## 10.10-1 Form Login
스프링 시큐리티는 html폼 기반 사용자 이름/비밀번호 인증을 지원한다.

###절차
1. 사용자 인증 시도 시 UsernamePasswordauthenticationFilter 에서 사용자 정보를 확인한다.
2. AntPathRequestMatcher 에서 로그인 정보를 확인한다.
3.1 정보가 미 일치 시 다른 필터로 이동
3.2 정보가 일치 시 Authentication 에서 사용자가 입력한 로그인 값을 저장하여 인증 객체를 생성한다.
4. AuthenticationManager 에서 인증 처리를 하는데 AuthenticationProvider 에게 인증처리를 위임한다.
5.1 인증 실패 시 AuthenticationException 발생
5.2 인증 성공 시 권한 정보 및 인증 객체를 저장하여 AhthenticationManager 에게 전송한다.
6. AuthenticationManager 는 인증 객체를 Authentication 에게 반환한다.
7. Authentication 에서 인증 결과인 인증 객체를 Securitycontext 에 전송한다.
8. SecurityContext 에서 인증 객체를 저장한다
SuccessHandler 처리

### form 로그인 조건
1. /login 요청은 post로 전송해야 한다.
2. CSRF 토근을 포함해야 한다. (타임리프의 경우 자동으로 추가됨)
3. 사용자 이름은 username, 비밀번호는 password 파라미터로 명시해주어야 한다.
4. HTTP 파라미터 error가 있다면 사용자가 유효한 username, password를 제공하지 못했음을 나타낸다.
5. HTTP파라미터 logout이 있으면 사용자가 로그아웃에 성공한것이다.

## 10.10-2 Basic Authentication
HTTP 인증이 어떻게 동작되는지 알아보자

1. 권한이 없는 리소스 /private에 인증되지 않은 요청을 보낸다.
2. 스프링 시큐리티의 FilterSecurityInterceptor에서 AccessDeniedException을 날려 인증되지 않은 요청을 거절했음을 알린다.
3. 인증되지 않은 사용자임으로 ExceptionTranslationfilter에서 인증을 시작한다. 설정한 AuthenticationEntryPoint는 BasicAuthenticationEntryPoint 인스턴스로 WWW-Authenticate 헤더를 전송한다. (이때 클라이언트가 기존 요청을 다시 전송할 수 있으므로 RequestCache는 보통 요청을 저장하지 않는 NullRequestcache를 사용한다.

클라이언트는 WWW-Authentication 헤더를 받으면 username과 password로 재시도해야 한다는 것을 알고있다. 다음은 username과 password를 처리하는 플로우다.

1. 사용자가 username과 password를 제출하면 usernamePasswordAuthenticationFilter는 HttpServletRequest에서 이 값을 추출해 Authentication 유형 중 하나인 usernamePasswordAuthenticationToken을 만든다.
2. UsernamePasswordAuthenticationToken을 AuthenticationManager로 넘겨 인증한다.
3-1. 인증 실패 시 SecurityContextHolder를 비운 뒤 RememberMeServices.loginFail을 실행한다. (remember me 설정이 없으면 동작을 하지 않음)
그 뒤 AuthenticationEntryPoint를 실행해서 WWW-Authenticate 전송을 한다.
3-2. 인증 성공 시 SecurityContextHolder에 Authentication을 세팅하고 RememberMeServices.loginSuccess를 실행한다. 
(마찬가지로 remember me 설정이 없으면 동작을 하지 않음) 그 뒤 BasicAuthenticationFilter에서 FilterChain.doFilter(req, resp)를 호출해서 나머지 애플리케이션 로직을 실행한다.

## 10.10-3 Digest Authentication
다이제스트 인증은 안전하지 않기에 최신 애플리케이션에서는 단방향 적응형 비밀번호 해쉬를 사용해서 credential을 저장해야 한다.

## 10.10-4 In-Memory Authentication
스프링 시큐리티의 InmemoreyUserDetailService는 메모리 기반으로 username/password를 인증하는 userDetailService 구현체다. InMemoreyUserDetailManager는 UserDetailManager 인터페이스도 구현했기 때문에 UserDetail를 관리할 수 있다. username/password를 읽어 인증하도록 설정하면 시큐리티는 UserDetail를 통해 인증한다.

## 10.10-5 JDBC Authentication
시큐리티의 JdbcDaoImpl은 JDBC 기반으로 username/password 인증하는 UserDetailService 구현체이다. JdbcDaoImpl을 상속한 JdbcUserdetailsManager는 userDetailsManager인터페이스도 구현했기에 UserDetails를 관리할 수 있다. username/password를 읽어 인증하도록 설정하면 시큐리티는 UserDetails를 통해 인증한다.

(단 해당 방법을 이용하기 위해서는 조건에 맞는 스키마가 필요하다)

## 10.10-6 UserDetails
UserDetails는 UserDetailsService가 반환하는 값으로 DaoAuthenticationProvider가 UserDetails를 인증하고, 이 UserDetails를 principal로 가진 Authentication을 반환한다.

## 10.10-7 UserDetailsService
UserDetasilService는 DaoAuthenticationProvider가 username/password로 인증할 때 필요한 username, password와 다른 속성을 조회할 때 사용한다. (인메모리와 JDBC 기반 두 개 모두의 구현체가 있음)
 - 커스텀 인증을 정의하기 위해서는 커스텀 UserDetailService를 빈으로 만들면된다.

## 10.10-8 PasswordEncoder
서블릿에서 시큐리티를 사용하면 PassowrdEncoder를 통합해 비밀번호를 안전하게 저장할 수 있다. 시큐리티가 사용하는 PasswordEncoder 구현체를 커스텀 하려면 PasswordEncoder 빈을 정의해야 한다.

## 10.10-9 DaoAuthenticationProvider
DaoAuthenticationProvider는 UserDetailService와 PasswordEncoder로 username/password를 인증하는 AuthenticationProvider 구현체다.

###동작과정
1. Username, Password조회하는 인증 Filter에서 UsernamePasswordAuthenticationtoken을 AuthenticationManager로 넘긴다. (AuthenticationManager는 ProviderManager가 구현하고 잇다.)
2. 이 ProviderManager는 DaoAuthenticationProvider를 AuthenticationProvider로 사용하도록 설정돼 있다.
3. DaoAuthenticationProvider는 UserDetailsService에서 UserDetails를 조회한다.
4. 그 다음 DaoAuthenticationProvider는 이전 단계에서 얻은 UserDetails에 있는 비밀번호를 PasswordEncoder로 검증한다. (위에서 말한 애플리케이션에서 위험한 다이제스트 암호화가 아님)
5. 인증에 성공하면 UsernamePasswordAuthenticationToken타입의 Authentication을 반환하며, 이 객체는 UserDetailsService가 반환한 UserDetails를 principal로 가지고 있다. 결국에 반환한 UsernamePasswordAuthenticationToken은 인증 Filter에서 SecurityContextHolder로 세팅된다.

10.10-10 LDAP Authentication
LDAP은 조직에서 사용자 정보를 관리하기 위한 중앙 저장소와 인증 서비스로 많이 사용한다. 어플리케이션 사용자의 role을 저장할 때도 사용할 수 있다.

시큐리티의 LDAP 기반 인증은 username/password로 인증하도록 설정했을때 사용한다. 하지만 username/password로 인증하더라도 UserDetailService와 통합되지 않는다. LDAP 서버는 bind 인증에서 비밀번호를 반환하지 않기에 애플리케이션에서 비밀번호를 인증할 수 없기 때문이다.

LDAP 서버의 설정 시나리오는 다양하므로 시큐리티는 전체 설정을 바꿀 수 있는 LDAP provider를 제공한다. 인증을 처리할 때와 role을 조회할 때는 별도의 전략 인터페이스를 사용하며, 다양한 상황에서 설정할 수 있는 디폴트 구현체를 제공한다.

만약 LDAP 서버를 설정했다면 시큐리티에서도 사용자 인증시 사용할 LDAP 서버를 가리키도록 해야한다. (JDBC의 DataSource와 동일하게 LDAP의 ContextSource를 구성하면 된다.)

Using BInd Authentication
Bind인증은 LDAP에서 가장 많이 사용하는 사용자 인증 메커니즘이며, LDAP서버로 사용자 credentital을 전송하고, 서버에서 이를 인증한다.
 - 장점 : 사용자 정보 (i.e. pwd)를 클라이언트에 노출할 필요가 없어 유출 가능성이 낮음

*ie password가 IE에서 저장하는 비밀번호를 의미하나?

