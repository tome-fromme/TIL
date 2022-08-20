## 10.1 SecurityContextHolder
스프링 시큐리티의 인증 모델 중심에는 `SecurityContextHolder`가 있다. 이 홀더는 SecurityContext를 가지고 있으며, 이 안에는 시큐리티를 통하여 인증한 사용자의 정보를 저장한다. 기본적으로 `SecurityContextHolder`는 ThreadLocal을 사용해서 정보를 저장하기에 메소드에 직접 SecurityContext를 넘기지 않아도 동일한 스레드라면 항상 SecurityContext에 접근할 수 있다. 기존 principal 요청을 처리한 다음에 비워주는것을 잊지만 않으면 ThreadLocal를 사용해도 안전하다. 그렇기에 스프링 시큐리티 FilterChainProxy는 항상 SecurityContext를 비워준다.

## 10.2 SecurityContext
시큐리티를 통하여 인증한 사용자의 정보인 Authentication 객체를 가지고 있다.

## 10.3 Authentication
현재 인증된 사용자를 나타냄

## 10.4 GrantedAuthority
사용자에게 부여한 권한은 `GrantedAuthority`로 추상화 한다. 권한은 보통 역할로 구분되며 이런 역할은 이후에 웹 인가, 메소드 인가, 도메인 객체 인가에서 사용한다. 스프링 시큐리티의 다른 코드에선 role을 해석하고 필요한 권한을 확인한다. - 이름/비밀번호 기반 인증을 사용한다면 보통 UserDetailService가 GrantedAuthority를 로드한다.

## 10.5 AuthenticationManager
`AuthenticationManager`는 스프링 시큐리티 필터의 인증 수행 방식을 정의하는 API이다. 매니저가 반환한 권한정보를 시큐리티 컨텍스트 홀더에 설정하는 건 `AuthenticationManager`를 호출한 객체가 담당한다. 만약 시큐리티 필터를 사용하지 않는다면 직접 SecurityContextHolder를 설정하면 된다. (가장 많은 구현체는 PrividerManager이다.)

## 10.6 ProviderManager
AuthenticationManager 구현체로 동작을  AuthenticationProvider List에 위임한다. 모든 Authenticationprovider는 인증을 성공, 실패, 아니면 결정을 내릴 수 없을때 다운스트림에 있는 Authenticationprovider가 결정하도록 만들 수 있다. 만약 설정한 Authenticationprovider가 **전부다 인증**을 하지 못한다면 AuthenticationException의 하위클래스인 ProviderNotFoundException이 발생하면서 실패한다.

보통은 AuthenticationProvider마다 각자가 맡은인증을 수행하는 법을 알고있다. 예를들어 하나는 이름/비번을 검증하고, 하나는 SAML 인증을 담당할 때 인증 유형마다 AuthenticationProvider가 있기에 빈 하나만 외부로 노출하면서 여러 인증을 지원할 수 있다.

여러 ProviderManager 인스턴스에 동일한 부모 (AuthenticationManager)를 공유하는것도 가능하다.

## 10.7 Authenticationprovider
ProviderManager엔 AuthenticationProvider를 여러개 주입할 수 있다.
각각 Authenticationprovider마다 당담하는 인증 유형이 다르다.

## 10.8 Request Credentials With AuthenticationEntryPoint
AuthenticationEntryPoint는 클라이언트의 credential을 요청하는 HTTP 응답을 보낼때 사용한다.

## 10.9 AbstractAuthenticationProcessingFilter
사용자의 credential을 인증하기 위한 베이스 필터이다. 시큐리티는 보통 AuthenticationEntryPoint로 credential을 요청한다. 그러고 나면 AbstractAuthenticationProcessingFilter는 제출한 모든 인증 요청을 처리할 수 있다.

