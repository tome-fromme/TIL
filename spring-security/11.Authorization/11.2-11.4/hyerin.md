# 11.2. Authorize HttpServletRequest with FilterSecurityInterceptor
- 서블릿 아키텍처와 구현체를 기반으로 서블릿 기반 어플리케이셔에서 권한을 부여하는 방법 설명


- FilterSecurityInterceptor는 HttpServletRequest를 사용해서 권한을 인가
- 이 인터셉터는 FilterChainProxy에 하나의 보안 필터로 추가됨
<img src="https://godekdls.github.io/images/springsecurity/filtersecurityinterceptor.png">
1. FilterSecurityInterceptor가 SecurityContextHolder에서 Authentication을 조회
2. FilterSecurityInterceptor가 넘겨받은 HttpServletRequest, HttpServletResponse, FilterChain으로 FilterInvocation을 만듦
3. FilterInvocation을 SecurityMetadataSource로 넘겨 ConfigAttribute 컬렉션을 가져옴
4. Authentication, FilterInvocation, ConfigAttribute 컬렉션을 AccessDecisionManager로 넘김
5. 인가를 거절하면 AccessDeniedException을 던짐. AccessDeniedException은 ExceptionTranslationFilter가 처리
6. 인가를 승인하면 FilterSecurityInterceptor는 일반적인 어플리케이션 프로세스를 실행할 수 있도록 FilterChain을 이어감


- 기본적으로 스프링 시큐리티에서 권한을 인가하려면 모든 요청을 인증해야 함
````
// java
protected void configure(HttpSecurity http) throws Exception {
    http
        // ...
        .authorizeRequests(authorize -> authorize
            .anyRequest().authenticated()
        );
}

// kotlin
fun configure(http: HttpSecurity) {
    http {
        // ...
        authorizeRequests {
            authorize(anyRequest, authenticated)
        }
    }
}
````
- 여러가지 규첵에 우선순위를 부여할 수도 있음
````
// java
protected void configure(HttpSecurity http) throws Exception {
    http
        // ...
        .authorizeRequests(authorize -> authorize // (1)
            .mvcMatchers("/resources/**", "/signup", "/about").permitAll() // (2)
            .mvcMatchers("/admin/**").hasRole("ADMIN") // (3)
            .mvcMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')") // (4)
            .anyRequest().denyAll() // (5)         
        );
}

// kotlin
fun configure(http: HttpSecurity) {
   http {
        authorizeRequests { // (1)
            authorize("/resources/**", permitAll) // (2)
            authorize("/signup", permitAll)
            authorize("/about", permitAll)

            authorize("/admin/**", hasRole("ADMIN")) // (3)
            authorize("/db/**", "hasRole('ADMIN') and hasRole('DBA')") // (4)
            authorize(anyRequest, denyAll) // (5)
        }
    }
}
````
- (1) 인가 조건을 여러개 지정. 각 규칙은 선언한 순서대로 적용
- (2) 모든 사용자가 접근할 수 있는 몇가지 URL 패턴을 지정
  - 구체적으로 말하면 "/resources"로 시작하는 URL이나, "/signup", "/about" 요청은 모든 사용자가 접근할 수 있음
- (3) "/admin"으로 시작하는 모든 요청은 "ROLE_ADMIN"을 가진 사용자로 제한. hasRole 메소드를 사용했기 때문에 "ROlE_" 프리픽스는 지정할 필요 없음
- (4) "/db/"로 시작하는 모든 요청은 "ROLE_ADMIN"과 "ROLE_DBA" 모두 필요. hasRole 메소드를 사용했기 때문에 "ROLE_" 프리픽스는 지정할 필요 없음
- (5) 위 조건에 충족하지 않는 다른 URL은 접근을 거절. 인증 조건을 누락하는 실수를 방지하는 좋은 전략임

# 11.3. Expression-Based Access Control
- 스프링 시큐리티 3.0부터 간단한 설정 속성과 voter로 접근 권한을 결정하는 방법 외에도, 스프링 EL표현식을 사용해 메커니즘을 구현할 수 있음
- 표현식 기반으로 접근을 제어할 땐 동일한 아키텍처를 사용하지만, 복잡한 Boolean 로직을 간단한 표현식 하나로 캡슐화 할 수 있음

## 11.3.1. Overview
- 스프링 시큐리티는 스프링 EL 표현식을 사용
- 표현식을 평가할 땐 평가 컨텍스트의 일부로 루트 객체를 사용
- 스프링 시큐리티는 웹과 메소드 시큐리티 전용 클래스를 루트 객체로 사용하기 때문에 별도의 내장 표현식을 사용할 수 있으며 현재 principal 등에 접근할 수 있음

### Common Built-In Expressions
- 표현식 루티 객체를 위한 베이스 클래스는 SecurityExpressionRoot
- 이 클래스는 웹, 메소드 시큐리티에서 공통적으로 사용할 수 있는 공통 표현식을 몇가지 제공

| Expression                                                           | Description                                                           |
|----------------------------------------------------------------------|-----------------------------------------------------------------------|
| hasRole(String role)                                                 | 현재 principal이 명시한 role을 가지고 있으면 true를 리턴                              |
| hasAnyRole(String… roles)                                            | 현재 principal이 명시한 role 중 하나라도 가지고 있으면 true를 리턴 (문자열 리스트를 콤마로 구분해서 전달) |
| hasAuthority(String authority)                                       | 현재 principal이 명시한 권한이 있으면 true를 리턴                                    |
| hasAnyAuthority(String… authorities)                                 | 현재 principal이 명시한 권한 중 하나라도 있으면 true를 리턴 (문자열 리스트를 콤마로 구분해서 전달)       |
| principal                                                            | 현재 사용자를 나타내는 principal 객체에 직접 접근할 수 있음                                |
| authentication                                                       | SecurityContext로 조회할 수 있는 현재 Authentication 객체에 직접 접근할 수 있음           |
| permitAll                                                            | 항상 true로 평가                                                           |
| denyAll                                                              | 항상 false로 평가                                                          |
| isAnonymous()                                                        | 현재 principal이 익명 사용자면 true를 리턴                                        |
| isRememberMe()                                                       | 현재 principal이 remember-me 사용자면 true를 리턴                               |
| isAuthenticated()                                                    | 사용자가 익명이 아니면 true를 리턴                                                 |
| isFullyAuthenticated()                                               | 사용자가 익명 사용자나 remember-me 사용자가 아니면 true를 리턴                            |
| hasPermission(Object target, Object permission)                      | 사용자가 target에 해당 permission 권한이 있으면 true를 리턴                           |
| hasPermission(Object targetId, String targetType, Object permission) | 사용자가 target에 해당 permission 권한이 있으면 true를 리턴                           |

## 11.3.2. Web Security Expressions
- URL별로 표현식을 적용하려면 먼저 <http> 요소의 use=expressions 속성을 true로 설정해야 함
- 이렇게 하면 스프링 시큐리티는 <intercept-url> 요소의 access 속성에 스프링 EL 표현식이 있음을 인지할 수 있음
- 표현식은 접근을 허용할지 말지를 Boolean으로 평가해야 함
````
<http>
    <intercept-url pattern="/admin*"
        access="hasRole('admin') and hasIpAddress('192.168.1.0/24')"/>
    ...
</http>
````
- 위에서 어플리케이션의 "admin" 영역은 (URL 패턴으로 정의함) "admin" 권한을 부여받은 사용자가 로컬 서브넷으로 접근할 때만 사용할 수 있도록 정의
- hasIpAddress 표현식은 웹 보안에서 사용할 수 있는 또다른 내장 표현식으로 WebSecurityExpressionRoot 클래스에 정의되어 있음
- 웹 접근 표현식을 평가할 땐 이 클래스 인스턴스를 루트 객체를 사용
- 또한 이 객체는 HttpServletRequest 객체를 request란 이름으로 직접 노출하므로 표현식에서 직접 request를 호출할 수도 있음
- 표현식을 사용하면 네임스페이스에 있는 AccessDecisionManager에 WebExpressionVoter가 추가됨
- 따라서 네임스페이스 없이 표현식을 사용한다면 이 중 하나를 설정에 직접 추가해야 함

### Referring to Beans in Web Security Expression
- 사용할 수 있는 표현식을 늘리고 싶다면 어떤 객체든지 스프링 빈으로 정의하면 쉽게 참조할 수 있음
````
// 예시
// webSecurity란 이름의 빈이 있고 이 빈의 메소드 시그니처는 아래와 같다고 가정
public class WebSecurity {
        public boolean check(Authentication authentication, HttpServletRequest request) {
                ...
        }
}

// 이 메소드는 다음과 같이 참조할 수 있음
<http>
    <intercept-url pattern="/user/**"
        access="@webSecurity.check(authentication,request)"/>
    ...
</http>

// 또는 자바 설정을 사용한다면
http
    .authorizeRequests(authorize -> authorize
        .antMatchers("/user/**").access("@webSecurity.check(authentication,request)")
        ...
    )
````
### Path Variables in Web Security Expressions
- URL에 있는 path variable을 참조해야 할 떄도 있음
````
// 예 - /user/{userId} 형식의 URL path에서 id를 가져와 사용자를 검색하는 RESTful 어플리케이션
// path variable도 간단하게 패턴에 지정해서 참조할 수 있음
// webSecurity란 이름의 빈이 있고, 이 빈의 메소드 시그니처는 아래와 같다고 가정
public class WebSecurity {
        public boolean checkUserId(Authentication authentication, int id) {
                ...
        }
}

// 이 메소드는 다음과 같이 참조할 수 있음
<http>
    <intercept-url pattern="/user/{userId}/**"
        access="@webSecurity.checkUserId(authentication,#userId)"/>
    ...
</http>

// 또는 자바 설정을 사용한다면
http
    .authorizeRequests(authorize -> authorize
        .antMatchers("/user/{userId}/**").access("@webSecurity.checkUserId(authentication,#userId)")
        ...
    );
````
- 위 두 설정 모두 URL이 매칭되면 checkUserId 메소드로 path variable을 (변환까지 해서) 전달
- 예를 들어 URL이 /user/123/resource 였다면 전달하는 id는 123

## 11.3.3. Method Security Expressions
- 메소드 시큐리티는 단순한 허가 또는 거절보다 조금 더 복잡한 규칙을 사용
- 스프링 시큐리티 3.0은 표현식을 종합적으로 지원하기 위한 새 어노테이션을 도입함

### @Pre and @Post Annotaions
- 네가지 어노테이션이 표현식 속성을 지원
- 사전/사후 권한 체크를 지원하며, 제출한 컬렉션 인자나 리턴한 값을 필터링할 수도 있음
- @PreAuthorize, @PreFilter, @PostAuthorize, @PostFilter
- 어노테이션 사용은 네임스페이스의 global-method-security 요소로 활성화
````
<global-method-security pre-post-annotations="enabled"/>
````

### Access Control using @PreAuthorize and @PostAuthorize
- 가장 유용한 어노테이션은 실제로 메소드를 실행할 수 있는지 없는지를 결정하는 @PreAuthorize
````
@PreAuthorize("hasRole('USER')")
public void create(Contact contact);
````
- 이 어노테이션은 "ROLE_USER" 권한이 있는 사용자만 접근할 수 있다는 뜻
````
@PreAuthorize("hasPermission(#contact, 'admin')")
public void deletePermission(Contact contact, Sid recipient, Permission permission);
````
- 여기선 현재 사용자가 실제로 주어진 연락처에 "admin" 권한이 있는지를 확인하기 위해 메소드 인자를 표현식 일부로 사용하고 있음
- 내장 표현식 hasPermission()은 어플리케이션 컨텍스트를 통해 스프링 시큐리티의 ACL 모듈로 연결되며 메소드 인자는 이름을 기준으로 표현식 변수로 사용할 수 있음


- 스프링 시큐리티가 메소드 인자를 리졸브하는 방법은 여러가지가 있음
- 스프링 시큐리티는 DefaultSecurityParameterNameDiscoverer를 사용해서 파라미터 이름을 발견
- 메소드의 단일 인자에 @PreAuthorize 어노테이션이 있는 경우 이 인자 값을 사용
- 이 어노테이션은 파라미터 이름에 관한 정보를 담지 않는 JDK 8 이전 버전으로 컴파일한 인터페이스에 유용
````
import org.springframework.security.access.method.P;
  
...
  
@PreAuthorize("#c.name == authentication.name")
public void doSomething(@P("c") Contact contact);
````
- 이 어노테이션을 사용하면 내부에선 어떤 어노테이션이든지 value 속성을 지원하도록 커스텀할 수 있는 AnnotationParameterNameDiscoverer를 사용
- 메소드 파라미터 중 한개라도 스프링 데이터의 @Param 어노테이션이 있으면 이 파라미터 값을 사용
- 이 어노테이션은 파라미터 이름에 관한 정보를 담지 않는 JDK 8 이전 버전으로 컴파일한 인터페이스에 유용
````
import org.springframework.data.repository.query.Param;
  
...
  
@PreAuthorize("#n == authentication.name")
Contact findContactByName(@Param("n") String name);
````
- 이 어노테이션을 사용하면 내부에선 어떤 어노테이션이든지 value 속성을 지원하도록 커스텀할 수 있는 AnnotationParameterNameDiscoverer를 사용


- -parameters 인자를 사용해서 JDK 8로 소스 코드를 컴파일하고 스프링 4+를 사용했다면, 표준 JDK 리플렉션 API로 파라미터 이름을 찾음
- 클래스와 인터페이스 둘 모두에서 동작함


- debug symbol을 포함해서 컴파일했다면, 이 debug symbol을 사용해서 파라미터 이름을 찾음
- 인터페이스는 파라미터 이름과 관련한 디버그 정보가 없으므로 동작하지 않음
- 인터페이스를 사용한다면 어노테이션이나 JDK 8 방식을 사용해야 함


- 표현식 내에선 모든 스프링 EL 기능을 사용할 수 있으므로 인자의 프로퍼티에도 접근할 수 있음
````
// 특정 메소드를 username이 연락처의 이름과 일치하는 사용자에게만 허가하고 싶을경우
@PreAuthorize("#contact.name == authentication.name")
public void doSomething(Contact contact);
````
- 위에선 또 다른 내장 표현식 authentication을 사용했으며, 이는 보안 컨텍스트에 저장된 Authentication을 나타냄
- principal 표현식을 사용하면 "principal" 프로퍼티에 직접 접근할 수도 있음
- principal은 보통 UserDetails 인스턴스이므로 principal.username이나 principal.enabled 같은 표현식을 사용하면 됨


- 일반적이진 않지만 메소드를 실행한 다음에 접근 제어를 확인하고 싶을 수도 있음
- 이땐 @PostAuthorize 어노테이션을 사용하면 됨
- 메소드가 리턴한 값에 접근하려면 표현식에 내장된 이름 returnObject를 사용

### Filtering using @PreFilter and @PostFilter
- 스프링 시큐리티는 컬렉션이나 배열 필터링을 지원하며, 이제는 표현식으로 구현할 수 있음
- 메소드가 리턴한 값에 가장 많이 사용
````
@PreAuthorize("hasRole('USER')")
@PostFilter("hasPermission(filterObject, 'read') or hasPermission(filterObject, 'admin')")
public List<Contact> getAll();
````
- @PostFilter 어노테이션을 사용하면 스프링 시큐리티는 리턴된 컬렉션을 순회해서 표현식의 결과가 false인 모든 요소를 제거
- filterObject는 컬렉션의 현재 객체를 참조
- @PreFilter를 사용하면 메소드 호출 전에 필터링할 수도 있지만 흔한 요구사항은 아님
- 기본 문법은 동일하지만 컬렉션 타입 인자가 둘 이상이라면 어노테이셔의 filterTarget 프로퍼티에 인자 이름을 지정해야 함


- 필터링은 데이터 조회 쿼리를 튜닝하는 용도가 아니라는 점을 명심해야 함
- 사이즈가 큰 컬렉션을 필터링하고 많은 엔트리를 제거하는 것은 비효율적

### Built-In Expressions
- 시큐리티에 특화된 내장 표현식도 있음
- filterTarget과 returnValue도 간단하게 사용할 수 있지만, hasPermission() 표현식을 사용하면 좀 더 자세히 살펴볼 수 있음

### The PermissionEvaluator Interface
- hasPermission() 표현식은 PermissionEvaluator 인스턴스로 위임
- 이 인터페이스는 표현식 시스템과 스프링 시큐리티의 ACL 시스템을 연결하기 위한 것으로, 추상적이 permission 기반으로 도메인 객체에 인가 조건을 지정할 수 있음
- ACL 모듈에 직접적인 의존성은 없으므로 필요하다면 다른 구현체로 바꿀 수 있음
- 이 인터페이스에는 두가지 메소드가 있음
````
boolean hasPermission(Authentication authentication, Object targetDomainObject,
                            Object permission);

boolean hasPermission(Authentication authentication, Serializable targetId,
                            String targetType, Object permission);
````
- 첫번째 인자를 (Authentication 객체) 제공하지 않은 경우만 제외하면 모두 가능한 표현식으로 매핑됨
- 첫번째 메소드는 접근을 제어하는 도메인 객체를 이미 로드한 경우에 사용
- 현재 사용자가 이 객체에 주어진 permission이 있다면 표현식은 true를 리턴
- 두번째 메소드는 객체를 로드하진 않았지만 그 식별자를 알 때 사용
- 올바른 ACL permission을 로드하려면 도메인 객체에 대한 추상적인 "type" 지정자도 필요
- 보통은 도메인 객체의 자바 클래스를 사용하지만, permission을 로드하는 방식과 일치하기만 하면 꼭 그래야만 한다는 법은 없음


- hasPermission() 표현식을 사용하려면 어플리케이션 컨텍스트에 PermissionEvaluator를 명시해야함
````
<security:global-method-security pre-post-annotations="enabled">
<security:expression-handler ref="expressionHandler"/>
</security:global-method-security>

<bean id="expressionHandler" class=
"org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler">
    <property name="permissionEvaluator" ref="myPermissionEvaluator"/>
</bean>
````
- myPermissionEvaluator는 PermissionEvaluator를 구현한 빈
- 보통 ACL 모듈에 있는 구현체 AclPermissionEvaluator를 사용함

### Method Security Meta Annotations
- 메소드 시큐리티에 메타 어노테이션을 사용하면 코드를 좀 더 가독성 있는 코드로 만들 수 있음
- 똑같은 복잡한 표현식을 코드 전체에 반복하고 있다면 특히 유용
````
@PreAuthorize("#contact.name == authentication.name")
````
- 이 코드를 여기저기 반복하는 대신 이 코드 대신 사용할 메타 어노테이션을 만들 수 있음
````
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("#contact.name == authentication.name")
public @interface ContactPermission {}
````
- 메타 어노테이션은 스프링 시큐리티의 모든 메소드 시큐리티 어노테이션에 사용할 수 있음
- 스펙 준수를 위해 JSR-250 어노테이션은 메타 어노테이션을 지원하지 않음


# 11.4. Secure Object Implementations
## 11.4.1 AOP Allianxe (MethodInvocation) Security Interceptor
- 스프링 시큐리티 2.0 이전엔 MethodInvocation을 보호하려면 꽤 많은 보일러 플레이트 설정이 필요했음
- 현재 권장하는 메소드 시큐리티 설정 방법은 네임스페이스 설정
- 네임스페이스를 사용하면 메소드 시큐리티를 위한 빈들이 자동으로 설정되기 때문에 정말로 구현체를 알 필요가 없음


- 메소드 시큐리티는 MethodInvocation을 보호해주는 MethodSecurityInterceptor로 구현
- 설정 방법에 따라 인터셉터를 특정한 빈 하나에서만 사용할 수도 있고 인터셉터 하나를 여러 빈이 공유할 수도 있음
- 인터셉터는 MethodSecurityMetadataSource 인스턴스를 사용해서 특정 method invocation에 적용할 설정 속성을 가져옴
- MapBasedMethodSecurityMetadataSource로는 메소드 이름을 (와일드카드 지원) 키로 갖는 설정 속성을 저장하며, 내부적으로 <intercept-methods>나 <protect-point> 요소로 어플리케이션에 해당 속성을 정의했을 때 사용
- 다른 구현체는 어노테이션 기반 설정에서 사용


### Explicit MethodSecurityInterceptor Configuration
- 어플리케이션 컨텍스트에 MethodSecurityInterceptor를 직접 설정해서 스프링 AOP의 프록시 메커니즘과 함께 사용할 수도 있음
````
<bean id="bankManagerSecurity" class=
    "org.springframework.security.access.intercept.aopalliance.MethodSecurityInterceptor">
<property name="authenticationManager" ref="authenticationManager"/>
<property name="accessDecisionManager" ref="accessDecisionManager"/>
<property name="afterInvocationManager" ref="afterInvocationManager"/>
<property name="securityMetadataSource">
    <sec:method-security-metadata-source>
    <sec:protect method="com.mycompany.BankManager.delete*" access="ROLE_SUPERVISOR"/>
    <sec:protect method="com.mycompany.BankManager.getBalance" access="ROLE_TELLER,ROLE_SUPERVISOR"/>
    </sec:method-security-metadata-source>
</property>
</bean>
````

## 11.4.2. AspectJ (JoinPoint) Security Interceptor
- AspectJ 보안 인터셉터는 앞에서 설명한 AOPP Alliance 보안 인터셉터와 매우 유사
- AspectJSecurityInterceptor: AspectJ 인터셉터
- 프록시를 통해 인터셉터를 구성할 때 스프링 어플리케이션 컨텍스트에 의존하는 AOP Alliance 보안 인터셉터와는 달리, 
AspectJSecurityInterceptor는 AspectJ 컴파일러를 통해 구성됨
- 어플리케이션 하나에 보안 인터셉터 두 종류를 모두 사용하는게 그렇게 드문일은 아님
- 보통 AspectJSecurityInterceptor로 도메인 객체 인스턴스 보안을 처리하고, AOP Alliance MethodSecurityInterceptor로 서비스 레이어 보안을 처리
````
// 스프링 어플리케이션 컨텍스트에 AspectJSecurityInterceptor를 설정하는 방법

<bean id="bankManagerSecurity" class=
    "org.springframework.security.access.intercept.aspectj.AspectJMethodSecurityInterceptor">
<property name="authenticationManager" ref="authenticationManager"/>
<property name="accessDecisionManager" ref="accessDecisionManager"/>
<property name="afterInvocationManager" ref="afterInvocationManager"/>
<property name="securityMetadataSource">
    <sec:method-security-metadata-source>
    <sec:protect method="com.mycompany.BankManager.delete*" access="ROLE_SUPERVISOR"/>
    <sec:protect method="com.mycompany.BankManager.getBalance" access="ROLE_TELLER,ROLE_SUPERVISOR"/>
    </sec:method-security-metadata-source>
</property>
</bean>
````
- 클래스 명만 제외하면 AspectJSecurityInterceptor는 AOP Alliance 보안 인터셉터와 완전히 동일
- SecurityMetadataSource는 AOP 라이브러리에 있는 클래스가 아닌 java.lang.reflect.Method로 동작하기 때문에 두 인터셉터에 같은 securityMetadataSource를 공유하는 것도 가능
- 접근 권한을 결정할 땐 관련 AOP 라이브러리 전용 invocation을 (i.e. MethodInvocation 또는 JoinPoint) 사용하기 때문에 다양한 추가 기준을 고려해서 결정할 수 있음 (메소드 인자 등)


- AspectJ aspect 정의
````
package org.springframework.security.samples.aspectj;

import org.springframework.security.access.intercept.aspectj.AspectJSecurityInterceptor;
import org.springframework.security.access.intercept.aspectj.AspectJCallback;
import org.springframework.beans.factory.InitializingBean;

public aspect DomainObjectInstanceSecurityAspect implements InitializingBean {

    private AspectJSecurityInterceptor securityInterceptor;

    pointcut domainObjectInstanceExecution(): target(PersistableEntity)
        && execution(public * *(..)) && !within(DomainObjectInstanceSecurityAspect);

    Object around(): domainObjectInstanceExecution() {
        if (this.securityInterceptor == null) {
            return proceed();
        }

        AspectJCallback callback = new AspectJCallback() {
            public Object proceedWithObject() {
                return proceed();
            }
        };

        return this.securityInterceptor.invoke(thisJoinPoint, callback);
    }

    public AspectJSecurityInterceptor getSecurityInterceptor() {
        return securityInterceptor;
    }

    public void setSecurityInterceptor(AspectJSecurityInterceptor securityInterceptor) {
        this.securityInterceptor = securityInterceptor;
    }

    public void afterPropertiesSet() throws Exception {
        if (this.securityInterceptor == null)
            throw new IllegalArgumentException("securityInterceptor required");
        }
    }
}
````
- 위에서 보안 인터셉터는 모든 PersistableEntity 인스턴스에 적용되며 PersistableEntity는 위에 나타나있지 않지만 추상클래스 (원하는 다른 클래스나 pointcut 표현식을 사용해도 됨)
- AspectJCallback에서 proceed(); 구문은 around() 본문 내에 있을 때만 특별한 의미를 갖기 때문에 필요
- AspectJSecurityInterceptor는 타겟 객체를 계속 이어가려면 이 익명 AspectJCallback 클래스를 실행함


- 스프링이 이 aspect를 로드하고 AspectJSecurityInterceptor와 연결할 수 있도록 설정을 추가해야함
````
<bean id="domainObjectInstanceSecurityAspect"
    class="security.samples.aspectj.DomainObjectInstanceSecurityAspect"
    factory-method="aspectOf">
<property name="securityInterceptor" ref="bankManagerSecurity"/>
</bean>
````
- 이제 어플리케이션 내 어디든지 적합하다고 생각하는 방법으로 빈을 만들 수 있으며, 그 빈에는 시큐리티 인터셉터가 적용될 것