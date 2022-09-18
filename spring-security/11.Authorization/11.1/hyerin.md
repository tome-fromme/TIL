# 11.1. Authorization Architecture
## 11.1.1. Authorities
- 어떻게 모든 Authentication 구현체가 GrantedAuthority 객체 리스트를 저장하는지 설명
- GrantedAuthority 객체: principal에게 부여한 권한
- AuthenticationManager가 GrantedAuthority 객체를 Authentication 객체에 삽입하며 이후 권한을 결정할 때 
AccessDecisionManager가 GrantedAuthority를 읽어감
- GrantedAuthority는 메소드가 하나뿐인 인터페이스
````
String getAuthority();
````
- AccessDecisionManager에선 이 메소드로 GrantedAuthority를 명확한 String으로 조회할 수 있음
- GrantedAuthority가 값을 String으로 리턴하기 때문에 AccessDecisionManager 대부분 이를 쉽게 읽어갈 수 있음 
- GrantedAuthority를 명확하게 String으로 표현할 수 없다면 GrantedAuthority는 복잡한 케이스로 간주하고 getAuthority()에서는 null 리턴
- null을 리턴했다는 것은 AccessDecisionManager에 GrantedAuthority를 이해하기 위한 구체적인 코드가 있어야 한다는 뜻
- 스프링 시큐리티는 GrantedAuthority 구현체 SimpleGrantedAuthority를 제공
- 이 클래스는 사용자가 지정한 String을 GrantedAuthority로 변환해줌
- 시큐리티 아키텍처에 속한 모든 AuthenticationProvider는 Authentication 객체에 값을 채울 때 SimpleGrantedAuthority를 사용 

## 11.1.2. Pre-Invocation Handling
- 스프링 시큐리티는 method invocation 이나 웹 요청같은 보안 객체에 대한 접근을 제어하는 인터셉터를 제공
- 호출을 허용할지 말지를 결정하는 pre-invocation 결정은 AccessDecisionManager에서 내림

### AccessDecisionManager
- AccessDecisionManager는 AbstractSecurityInterceptor에서 호출하며 최종적인 접근 제어를 결정
- AccessDecisionManager는 세가지 메소드 있음
````
void decide(Authentication authentication, Object secureObject,
    Collection<ConfigAttribute> attrs) throws AccessDeniedException;

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
````
- AccessDecisionManager의 decide 메소드는 권한을 결정하기 위해 필요한 모든 정보를 건내 받음 
- 특히 보안 Object를 건내 받으면 실제 보안 객체를 실행할 때 넘긴 인자를 검사할 수 있음
- 접근을 거절할 경우엔 AccessDeniedException 던짐
- supports(ConfigAttribute) 메소드는 가동 시점에 AbstractSecurityInterceptor가 호출하고,
AccessDecisionManager가 건내받은 ConfigAttribute의 처리 가능 여부를 결정
- supports(Class) 메소드는 시큐리티 인터셉터 구현체가 호출하며 
설정해둔 AccessDecisionManager가 시큐리티 인터셉터가 제출할 보안 객체 타입을 지원하는지 확인

### Voting-Based AccessDecisionManager Implementations
- 인가와 관련한 모든 동작을 제어하고 싶으면 커스텀 AccessDecisionManager를 사용해도 되지만
스프링 시큐리티는 투표를 기반으로 동작하는 몇가지 AccessDecisionManager를 제공


<img src="https://godekdls.github.io/images/springsecurity/access-decision-voting.png"/>


- 투표 방식에선 권한을 결정할 때 일련의 AccessDecisionVoter 구현체에 의견을 물음
- 그 후, AccessDecisionManager가 투표 결과를 취합해서 AccessDeniedException을 던질지 말지 결정


- AccessDecisionVoter는 메소드를 세개 가진 인터페이스
````
int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attrs);

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
````
- 구현체는 AccessDecisionVoter의 static 필드 ACCESS_ABSTAIN, ACCESS_DENIED, ACCESS_GRANTED 중 하나를 의미하는 int 값 리턴
- 인가에 대해서 특별한 의견이 없을 때는 ACCESS_ABSTAIN 리턴
- 의견이 있다면 반드시 ACCESS_DENIED나 ACCESS_GRANTED를 리턴해야 함


- 스프링 시큐리티는 투표 결과를 집계하는 세가지 AccessDecisionManager 구현체 제공
  - ConsensusBased: 기권표를 제외한 투표 결과를 합산해 접근을 허가하거나 거절
  - AffirmativeBased: ACCESS_GRANTED가 하나라도 있으면 권한을 부여 (즉, 찬성표가 하나라도 있으면 거절표는 무시)
  - UnanimousBased: 기권을 제외한 모든 표가 만장일치로 ACCESS_GRANTED일 때만 접근을 허가. ACCESS_DENIED가 하나라도 있으면 접근 거부
  - 세가지 구현체는 모두 기권했을 때의 동작을 제어할 수 있는 파라미터 제공
- 다른 방식으로 투표 결과를 집계하는 커스텀 AccessDecisionManager를 구현해도 됨
- ex) AccessDecisionVoter의 투표에는 가중치를 두고 특정 voter의 거절표는 기각시킬 수도 있음

### RoleVoter
- 스프링 시큐리티가 제공하는 AccessDecisionVoter 중 가장 많이 사용
- RoleVoter는 설정 속성을 간단한 role 이름으로 취급하고 사용자가 해당 role을 할당받았으면 접근 허가에 투표
- prefix "ROLE_" 로 시작하는 ConfigAttribute이 있을 때 투표에 참여
- GrantedAuthority가 리턴하는 String이 (getAuthority() 메소드) 
"ROLE_" 로 시작하는 ConfigAttributes 중 하나라도 완전히 일치하면 찬성표 던짐
- "ROLE_"로 시작하는 ConfigAttribute와 일치하는게 하나도 없으면 반대표 던짐
- "ROLE_"로 시작하는 ConfigAttribute가 없으면 기권

### AuthenticatedVoter
- 익명 사용자, 완전히 인증된 사용자, remember-me로 인증한 사용자를 구분 할 수 있음
- 익명 사용자 접근을 위한 IS_AUTHENTICATED_ANONYMOUSLY 속성 처리

### Custom Voters
- 커스텀 AccessDecisionVoter로도 원하는 곳에 접근 제어 로직을 넣을 수 있음
- 보통 어플리케이션에 특화된 로직이나(비지니스 로직 관련) 보안 관리 로직을 구현하는 데 사용
- ex) 스프링 웹사이트에 있는 블로그 문서에서 설명하는 방법으로, 실시간으로 정지된 계정의 접근을 거절할 수 있음
  - https://spring.io/blog/2009/01/03/spring-security-customization-part-2-adjusting-secured-session-in-real-time

## 11.1.3. After Invocation Handling
- 보통 보안 객체를 실행하기 전에는 AbstractSecurityInterceptor가 AccessDecisionManager를 호출하는데, 
실제로는 보안 객체가 리턴하는 객체를 바꿔야 하는 어플리케이션도 있음
- 스프링 시큐리티는 ACL 기능과 통합되는 몇가지 구현체를 가진 편리한 훅을 제공

<img src="https://godekdls.github.io/images/springsecurity/after-invocation.png">


- AfterInvocationManager는 AfterInvocationProviderManager라는 구현체가 하나 있으며, 
이 구현체는 AfterInvocationProvider 리스트를 폴링
  - 폴링(polling): 하나의 장치(또는 프로그램)이 충돌 회피 또는 동기화 처리 등을 목적으로 
  다른 장치(또는 프로그램)의 상태를 주기적으로 검사하여 일정한 조건을 만족할 때 송수신 등의 자료처리를 하는 방식
- 각 AfterInvocationProvider는 리턴 객체를 수정하거나 AccessDeniedException를 던질 수 있음
  - 여러 provider가 객체를 수정할 수 있기 때문에 provider에 전달하는 객체는 리스트에 있는 이전 provider가 리턴한 객체
- AfterInvocationManager를 사용하더라도 MethodSecurityInterceptor의 AccessDecisionManager가 동적을 허용하려면 설정 속성 필요
- AccessDecisionManager 구현체 등 전형적인 스프링 시큐리티 설정을 사용했다면
특정 method invocation을 보호하기 위해 정의한 설정 속성이 없는 경우 모든 AccessDecisionVoter가 투표를 기권할 것
- AccessDecisionManager의 "allowIfAllAbstainDecisions" 프로퍼티가 false였다면 AccessDeniedException 던짐
  - 이 이슈를 피하려면 "allowIfAllAbstainDecisions"를 true로 설정하거나 (비추)
  - AccessDecisionVoter가 접근 허용에 투표할만한 설정 속성을 최소 한개는 사용. 보통 ROLE_USER나 ROLE_AUTHENTICATED 설정 속성 이용 (권장)

## 11.1.4 Hierarchical Roles
- role-hierarchy를 사용하면 특정 role이 다른 role을 포함하도록 설정할 수 있음
- 스프링 시큐리티의 RoleVoter를 확장한 RoleHierarchyVoter는 RoleHierarchy를 설정할 수 있으며,
이 값을 통해 사용자에게 할당할 모든 "reachable authorities"를 가져올 수 있음
````
<bean id="roleVoter" class="org.springframework.security.access.vote.RoleHierarchyVoter">
    <constructor-arg ref="roleHierarchy" />
</bean>
<bean id="roleHierarchy"
        class="org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl">
    <property name="hierarchy">
        <value>
            ROLE_ADMIN > ROLE_STAFF
            ROLE_STAFF > ROLE_USER
            ROLE_USER > ROLE_GUEST
        </value>
    </property>
</bean>
````
- ROLE_ADMIN ⇒ ROLE_STAFF ⇒ ROLE_USER ⇒ ROLE_GUEST
- AccessDecisionManager에 RoleHierarchyVoter를 설정하면,
제약조건을 평가할 때 ROLE_ADMIN으로 인증한 사용자는 마치 네가지 role를 모두 가진 것처럼 생동할 수 있음
- Role hierarchy는 어플리케이션의 접근 제어 설정을 단순화하고 사용자마다 할당할 권한을 줄일 수 있는 편리한 수단
- 좀 더 복잡한 요구사항이 있다면 어플리케이션에 필요한 접근 권한과 사용자에게 할당할 role을 매핑하는 로직을 정의해서 
사용자 정보를 로드할 때 이 둘을 변환하면 됨