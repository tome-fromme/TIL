# Authorities
GrantedAuthority 객체는 principal에게 부여한 권한을 의미한다. AuthenticationManager가 GrantedAuthority 객체를 Authentication 객체에 삽입하며, 이후 권한을 정할때 AccessDecisionManager가 GrantedAuthority를 읽어가며, 이를 읽어가기 쉽게 String으로 반환을 한다. 만약 String으로 반환을 할 수 없는 복잡한 케이스일 경우 null을 반환한다.

### Pre, After - Invocation Handling
시큐리티는 method invocation이나 웹 요청같은 보안객체에 대한 접근을 제어하는 인터셉터를 제공한다. 호출을 허용할지 말지를 결정하는 pre-invocation 결정은 AccessDecissionManager에서 내린다.

*AccessDecisionManager*는 (3가지 메서드 존재) AbstractSecurityInterceptor에서 호출하며, 최정족인 접근 제어를 결정한다.

1. decide 메서드는 권한을 결정하기 위한 필요한 정보를 건내받는 메서드 접근 거절 시 AccessDeniedException 발생
2. supports(ConfigAttribute) 메서드는 기동 시점에 AbstractSecurityInterceptor가 호출하며, AccessDecisionManager가 건내받은 ConfigAttribute의 처리 가능 여부를 결정한다.
3. supports(Class)메서드는 시큐리티 인터셉터 구현체가 호출되며, 설정해둔 AccessDecisionManager가 시큐리티 인터셉터가 제출할 보안 객체 타입을 지원하는지 확인한다.

AfterInvocationManager도 AfterInvocationProviderManager라는 구현차가 하나 있으며, 이 구현체는 AfterInvocationProvider 리스트를 폴링한다. 각 AfterInvocationProvider는 리턴 객체를 수정하거나 AccessdeniedException을 던질 수 있다.

### Hierarchical Roles
여기서 사용하는 계층 구조로는
ROLE_ADMIN => ROLE_STAFF => ROLE_USER => ROLE_GUEST 로 4가지 역할이 있으며, AccessDecisionManager에 RoleHierarchyvoter를 설정하면 제약조건 확인 시 ROLE_ADMIN은 하위 모든 role을 가진것 처럼 행동할 수 있다.

# 11.2 Authorize HttpServletRequestWith FilterSecurityInterCeptor

해당 섹션은 서블릿 아키텍처와 구현체를 기반 어플리케이션에서 권한을 부여하는 방법을 다룸

FilterSecurityInterceptor는 HttpServletRequest를 사용해서 권한을 인가한다. 이 인터셉터는 FilterChainProxy에 하나의 보안 필터로 추가된다.

동작과정
1. FilterSecurityInterceptor가 SecurityContextHolder에서 AUthentication을 조회
2. FilterSecurityInterceptor가 넘겨받은 HttpServletReqeust, Response, FilterChain으로 FilterInvocation을 생성
3. FilterInvocation을 SecurityMetadataSource로 넘겨 ConfigAttribute 컬렉션을 가져온다,
4. 마지막으로 Authentciation, FilterInvocation, ConfigAttribute 컬렉션을 AccessDecisionManager로 넘긴다
- 인가 거절 시 AccessDeniedExcetpion 발생
- 인가 승인 시 FilterChain을 이어간다.

# 11.3 Expression-Based Access Control
시큐리티 3.0부터 간단한 설정 속성과 voter로 접근 권한을 결정하는 방법 외에도, 스프링 EL표현식을 사용해 인가 메커니즘을 구현할 수 있다.

## 11.3.3 Method Security Expressions
사전/사후 권한 체크를 지원하며, 제출한 컬렉션 인자나 리턴한 값을 필터링 할 수 있는 4가지 어노테이션 표션식 속성을 지원한다.

1. @PreAuthorize, @PostAuthorize : 실제로 메서드를 실행할 수 있는지 없는지 결정한다 (ex. 만약 USER 가 ADMIN 권한 메서드를 실행하려고함)
2. @PreFilter, @PostFilter : 스프링 시큐리티는 리턴된 컬렉션을 순회하여 표현식 결과가 false인 모든 요소를 제거한다. filterObject는 컬렉션의 현재 객체를 참조한다.

```
인가 및 요청 관련은 보통 1차로 Config파일에서 세팅
```

# 11.4 Secure Object Implementations

## 11.4.1 AOP Alliance (MethodInvocation) Security Interceptor
현재 권장하는 메서드 시큐리티 설정방법은 네임스페이스 설정이다. 네임스페이스 설정 시 메서드 시큐리티들 위한 빈들이 자동으로 설정되기 떄문에 구현체를 알 필요가 없어진다.

메서드 시큐리티는 MethodInvocation을 보호해주는 MethodSecurityInterceptor로 구현한다. 설정 방법에 따라 인터셉터를 특정한 빈 하나만 사용할 수 있고, 인터셉터 하나에 여러개 빈을 공유할 수 있다.

## 11.4.2 AspectJ (JoinPoint) Security Interceptor
인터셉터 이름은 AspectJSecurityInterceptor이다. 프록시를 통해 인터셉터를 구성할 떄 스프링 애플리케이션 컨텍스트에 의존하는 AOP Alliance 보안 인터셉터와 달리 AspectJSecurityInterceptor는 AspectJ컴파일러를 통해 구성된다.

애플리케이션 하나에 보안 인터셉터 두 종류를 모두 사용하는 게 그렇게 드문일은 아니다. 보통 AspectJSecurityInterceptor로 도메인 객체 인스턴스 보안을 처리하고, AOP Alliance MethodSecurityInterceptor로 서비스 레이어 보안을 처리한다.


