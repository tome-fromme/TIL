## 10.11 Session Management
http 세션 관련 기능은 SessionManagementFilter와 이 필터가 위임하는 SessionAuthenticationStrategy 인터페이스가 처리한다.  
전형적으로 *session-fixation 공격을 방어하고, 세션 타임아웃을 감지하고, 인증된 사용자가 동시에 열 수 있는 세션 수를 제한하는 등에 사용한다

*)컴퓨터 네트워크 보안에서 세션 고정 공격은 한 사람이 다른 사람의 세션 식별자를 수정할 수있는 시스템의 취약성을 악용하려고합니다. 대부분의 세션 고정 공격은 웹 기반이며 대부분은 URL 또는 POST 데이터에서 허용되는 세션 식별자에 의존합니다.

## 10.11.1 Detecting Timeouts
시큐리티는 유효하지 않은 세션 ID를 제출하면 이를 적절한 URL로 리다이렉트할 수 있다.

이 메커니즘에서 세션 타임아웃도 설정한 경우 로그아웃한 사용자가 브라우저를 닫지 않고 다시 로그인할 때 에러로 오인할 수 있다. 세션을 무효화할 때 세션 쿠키를 비우지 않으면 로그아웃하더라도 같은 쿠키를 제출하기 때문이다. (로그아웃 시 로그아웃 핸들러에서 JESESSION 쿠키를 제거할 수 있음)

##10.11.2 Concurrent Session Controller
시큐리티에서는 리스터를 추가하면, 같은 사용자가 여러번 로긍니할 수 없도록 제한할 수 있다.

##10.11.3 Session Fixation Attack Protection
사이트에 접근하여 세션 생성 뒤 다른 사용자가 해당 세션에 로그인 하도록 유도한다(세션 식별자를 파라미터로 가지고 있는 링크를 보내는 식으로) 시큐리티는 사용자가 로그인하면 자동으로 새로운 세션을 만들거나, 세션 ID를 바꿔서 이 공격을 방어한다. 방어할 필요 없거나 다른 요구사항과 충돌된다면 속성값의 설정을 바꿀 수 있다.

1. none : 기존 세션 유지
2. newSession : 새 세션을 만들고 기존 세션 데이터를 복사하지 않는다.
3. migrateSession : 새 세션을 만들고 기존 세션 속성을 새로운 세션에 옮긴다.
4. changeSessionId : 새 세션을 만들지 않고 대신 서블릿 컨테이너가 제공하는 방식으로 session fixation 공격을 방어한다.

## 10.11.4 SessionManagementFilter
SessionManagementFilter는 SecurityContextRepository 컨텐츠를 SecurityContextHolder에 있는 현재 컨텐츠와 비교해서 현재 요청을 처리하는 동안 사용자를 인증했는지 확인한다. (보통 pre-authentication이나 remember-me같은 상호작용이 없는 인증에서 사용) 필터는 repository인증 컨텍스트가 있으면 아물너 처리도 하지 않는다. 반대로 reposiotry 인증 컨텍스트가 없고 스레드 로컬에 익명이 아닌 Authentication 객체가 있다면, 이전 필터에서 인증한 것으로 간주한다. 이땐 설정해둔 SessionAuthenticationStrategy를 실행한다.

## 10.11.5 SessionAuthenticationStrategy
SessionManagementFilter, AbstractAuthenticationprocessingFilter 둘다 사용하므로 커스텀 폼 로그인 클래스를 만드는 등의 상황에선 둘 모두에 주입해주어야 한다.
디폴트 SessionFixationProtectionStrategy를 사용한다면, HttpSessionBindingListnener를 구현한 빈을 세션에 저장하면 제대로 동작하지 않을 수 있으며, 스프링 세션 스코프 빈도 마찬가지다.

## 10.11.6 Concurrency Control
시큐리티는 사용자가 같은 애플리케이션에 동시에 인증할 수 있는 횟수를 제한할 수 있다. 많은 ISV는 이를 사용해서 라이센스를 관리하며, 네트워크 관리자들은 이를 통해 사람들이 로그인 이름을 공유한 것을 막는다. (동시 세션을 제어하기 위해서는 설정을 추가해주어야 한다.)

# 10.12. Remember-me Authentication
*remember-me == r-me

## 10.12.1 Overview
remember-me 또는 persistent-login 인증은 인증 주체의 식별자를 기억하고 여러 세션에 사용할 수 있는 웹사이트를 말한다. 보통 브라우저에서 전송한 쿠키를 이후 세션에서 감지하고 자동으로 로그인하는 식으로 동작한다. 시큐리티는 이를 위한 훅과 두가지 r-me 구현체를 제공한다.
하나는 해시처리한 쿠키로 토근을 유지하고, 다른 하나는 DB등 영구 스토리지 메커니즘으로 토근을 저장한다.

두 구현체 모두 UserDetailsService가 있어야 한다. 만약 이를 사용하지 않는 provider를 쓴다면 애플리케이션 컨텍스트에 UserDetailsService빈을 추가 해주어야 한다.

## 10.12.2 Simple Hash-Based Token Aproach
이 방법은 해쉬를 사용해서 r-me 전략을 구현한다. 쿠키 자체는 인증 상호작용에 성공하면 브라우저가 보내는 값이다. 따라서 r-me 토큰은 명시한 기간에만 유효하며, 사용자 이름이나 비번, 키가 변경되면 더 이상 유효하지 않다. 특히 r-me 토큰은 유출되면 만료되기 전까지 모든 user agent에서 사용할 수 있다는 보안 이슈가 있다. (다이제스트 인증과 동일한 이슈) 사용자가 토큰이 탈취되었음을 알 수 있다면 즉시 비번을 변경하여 모든 remember-me 토큰을 무효화 시킬 수 있다. 보안이 더 중요한 서비스라면 다음 섹션에 있는 방법을 사용해야 한다. 아니면 쓰지 말던가...

1. Persistent Token Approach
- DB SQL로 생성된 persistent-logins 테이블이 있어야함
2. r-me interfaces and implementations
- r-me 는 UsernamePasswordAuthenticationFilter와 같이 사용하며, AbstractAuthenticationProcessingFilter 클래스에 있는 훅으로 구현한다. BasicAuthenticationFilter와 같이 사용할 수 있다. 훅에선 적당할 때 RememberMeService구현체를 실행한다.
3. PersistentTokenBasedRememberMeServices
- 토큰을 저장할 PersistentTokenRepository를 추가로 설정해야 하며, 테스트 전용 InMemoryTokenRepositoryImpl과 DB에 토큰을 저장하는 JdbcTokenRepositoryImpl 2개가 있다.

마찬가지로 r-me 서비스 활성화를 위하여 해당 설정한 클래스들을 빈으로 추가해주어야 한다.

## 10.13 OpenID Support
네임스페이스는 OpenID 로그인을 지원하며 해당 로그인을 인증하려면 myopenid.com 사이트로 로그인할 수 있어야 한다.

## 10.14 AnonymousAuthentiation









