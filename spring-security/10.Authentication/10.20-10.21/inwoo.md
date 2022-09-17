## 10.20 Handling Logouts

### 10.20.1 Logout Java Configuration
WebSecurityConfigurerAdapter를 사용했다면 자동으로 로그아웃 기능을 추가한다. (디폴트는 "/logout" URL 접근 시 자동으로 로그아웃 시킨다.)

### 10.20.3 LogoutHandler
로그인과 마찬가지로 로그아웃 시 동작할 핸들러를 설정할 수 있다. 단 예외를 던지면 안된다.

### 10.20.4 LogoutSuccessHandler
SimpleUrlLogoutSuccessHandler, HttpStatusReturningLogoutSuccessHandler 2개의 구현체를 제공하며, 로그아웃에 성공한 다음 LogoutFilter에서 호출하며, 적절한 곳으로 리다이렉트 또는 포워딩시키는 용도로 사용한다. 단  LogoutHandler와 다르게 예외를 던질 수 있음

## 10.21 Authentication Events
인증에 성공하거나 실패할때 각각 AuthenticationSuccessEvent, AuthenticationFailureEvent가 발생한다.

이 이벤트를 수신하려면 Publisher가 있어야 하며, 기본적인
DefaultAuthenticationEventPublisher가 기본적으로 수 많은 이벤트 발생 시 AuthentictaionfailureEvent를 발행한다.

이 이벤트를 수신하려면 먼저 AuthenticationEventPublisher를 설정해야 한다.
- Publisher는 예외가 정확하게 일치해야 이벤트를 발행한다. (즉 이 예외 클래스를 상속한 클래스에선 이벤트를 발생시키지 않는다.)

