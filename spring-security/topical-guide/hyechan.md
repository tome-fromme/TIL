https://www.notion.so/topical-guide-0a6b5561b8934afaa2d4ffc5eb8ce125

### 인증과 접근제어

Authentication(인증)

- 나는 누구인가

Authorization(인가)

- 어떤 권한을 갖는가

스프링 시큐리티는 인증과 인가가 분리되어 디자인 되어있고 각각 전략이 다르고, 각각 확장해서 쓸 수 있도록 제공한다

### 인증(Authentication)

인증과정 주요전략이 담긴 인터페이스는 AuthenticationManager이고, 메소드 단 한 개만 가지고 있다.

```java
public interface **AuthenticationManager**{
 Authentication authenticate(Authentication authentication)
	throws AuthenticationException
}
```

Authentication 객체를 받고, Authentication 객체를 반환한다.

AuthenticationManager는 authenticate() 메서드를 통해 다음 중 한 가지를 할 수 있다.

1. 유효한 principal이면 Authentication을 리턴한다.
(principal은 Authentication authentication에 담겨있다?)
2. 유효하지 않으면 AuthenticationException을 throw 한다
3. 결정하지 못하면 null을 리턴한다

AuthenticationManager의 구현체는 ProviderManager이다.

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean
```

@Override로 구현하고 있다

---

```java
@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {

		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		int currentPosition = 0;
		int size = this.providers.size();

		// UsernamePasswordAuthenticationFilter가 요청 가로채서
		// UsernamePasswordAuthenticationToken(Authentication)객체를
		// AuthenticationManager - providerManager에게 넘겨준다
		// 바로 이 메서드

		// UsernamePasswordAuthenticationToken을 처리해줄
		// AuthenticationProvider 가 있는가?
		// AuthentictaionProvider들을 순회

		for (AuthenticationProvider provider : getProviders()) {
			// 지금 넘어온 토큰 너가 가진 AuthenticationProvider 목록으로 해결가능해?
			if (!provider.supports(toTest)) {
				continue; // 현재 반복문 끝내고 다음 반복문으로
			}
			//여기서도 지원하는지,
			//AuthenticationProvider

			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",
						provider.getClass().getSimpleName(), ++currentPosition, size));
			}
			try {
				result = provider.authenticate(authentication);
		// provider.authenticate(authentication)
		// authenticate를 통해 Authentication 리턴

    **// => 🌱
    // 내부적으로 List를 순회하며 supoorts메소드를 호출해
    // Authentication 객체를 처리할 수 있는지 확인하고,
    // 처리 가능하다면 해당 AuthenticationProvider 에게 인증 처리를 위임한다.**

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException | InternalAuthenticationServiceException ex) {
				prepareException(ex, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw ex;
			}
			catch (AuthenticationException ex) {
				lastException = ex;
			}
		}
		if (result == null && this.parent != null) {
			// Allow the parent to try.
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}
			catch (ProviderNotFoundException ex) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException ex) {
				parentException = ex;
				lastException = ex;
			}
		}
		if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}
			// If the parent AuthenticationManager was attempted and successful then it
			// will publish an AuthenticationSuccessEvent
			// This check prevents a duplicate AuthenticationSuccessEvent if the parent
			// AuthenticationManager already published it
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).
		if (lastException == null) {
			lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
					new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
		}
		// If the parent AuthenticationManager was attempted and failed then it will
		// publish an AbstractAuthenticationFailureEvent
		// This check prevents a duplicate AbstractAuthenticationFailureEvent if the
		// parent AuthenticationManager already published it
		if (parentException == null) {
			prepareException(lastException, authentication);
		}
		throw lastException;
	}
```

인증 준비 단계에서 만들어진 Authentication은 

`UsernamePasswordAuthenticationToken`이며, 

이 타입을 처리할 수 있는 AuthenticationProvider은 

`AbstractUserDetailsAuthenticationProvider`
 추상클래스를 상속하는 `DaoAuthenticationProvider`클래스이다

```
AbstractUserDetailsAuthenticationProvider
```

```java
@Override
public boolean supports(Class<?> authentication) {
return(UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
}
```

```java
@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				() -> this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));
		String username = determineUsername(authentication);
		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);
		if (user == null) {
			cacheWasUsed = false;
			try {
				user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			}
			catch (UsernameNotFoundException ex) {
				this.logger.debug("Failed to find user '" + username + "'");
				if (!this.hideUserNotFoundExceptions) {
					throw ex;
				}
				throw new BadCredentialsException(this.messages
						.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
			}
			Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
		}
		try {
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException ex) {
			if (!cacheWasUsed) {
				throw ex;
			}
			// There was a problem, so try again after checking
			// we're using latest data (i.e. not from the cache)
			cacheWasUsed = false;
			user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
			this.preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
		}
		this.postAuthenticationChecks.check(user);
		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}
		Object principalToReturn = user;
		if (this.forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}
		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
```

boolean supports(Class<?> authentication);

Class<?>는 실제로는 Class<? extends Authentication> 이다.
(authentication() 메서드에 전달될 토큰 지원하고 있어요? 묻는 것)

ProviderManager는 일련의 AuthenticationProvider에게 위임함으로써 동일 어플리케이션 내 여러 다른 인증 메커니즘을 제공할 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9b4ad43d-0b83-4390-a32a-3715e08f930b/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ba7684fd-ec90-4e27-b94f-7424f8e24e9c/Untitled.png)

---

### Custimizing Authentication Managers

스프링 시큐리티는 일반적인 인증관리 특징을 빠르게 설정할 수 있게 헬퍼 제공
AuthenticationManagerBuilder를 많이 쓰고,
메모리,  JDBC, LDAP 사용자 정보를 셋팅하고 커스텀 UserDetailService를 추가할 수 있다.

AuthenticationManagerBuilder를 통해 인증 객체를 만들 수 있도록 제공해주고 있다.

AuthenticationManagerBuilder 쓰는 예

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  public initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
    builder.jdbcAuthentication().dataSource(dataSource)
			.withUser("dave").password("secret").roles("USER");
    //builder.inMemoryAuthentication()
		//	.withUser("dave").password("secret").roles("USER");

  }
}
```

만약 이렇게 한다면?

```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  DataSource dataSource;

  @Override
  public configure(AuthenticationManagerBuilder builder) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }
}
```

둘 다 어쨋든 인증 객체를 만드는건데

주입 받는 거 바꾼거니까 global 전역,
오버라이딩해서 사용하는데 빈으로 관리하는 게 아니어서 다른 빈에 @Autowired로 사용할 수 없다. local 지역으로 사용.

??활용??

### Authentication or AccessControl

인가처리

핵심 AccessDecisionManager(인터페이스)

```java
public interface AccessDecisionManager {

	void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
				throws AccessDeniedException, InsufficientAuthenticationException;
	boolean supports(ConfigAttribute attribute);
	boolean supports(Class<?> clazz);

}
```

이걸 구현하고 있는 abstract class AbstractAccessDecisionManager가 있고..
다시 상속하고 있는 구현체 3개 있따.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1c103a65-a6c2-4b02-acc4-0cd3cf9476e0/Untitled.png)

AccessDecisionVoter(인터페이스) 에게 기능들 위임.
(ProviderManager가 AuthenticationProviders들에게 위임하는 것과 유사)
~~? AuthenticationProvider는 다양한 토큰 처리를 위한 것?
그렇다면 AccessDecisionVoter를 사용하는 이유는?~~

```java
boolean supports(ConfigAttribute attribute);

boolean supports(Class<?> clazz);

int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
```

그리고 AccessDecisionVoter를 구현하고 있는 구현체들

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1d536419-7b6b-4bdf-a6a2-22d3582ecece/Untitled.png)

int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);

ConfigAttribute는 단 하나의 메소드만을 갖고 있는 인터페이스. 퍼미션 레벨을 판단할 수 있는 메타정보 제공
이 메소드는 접근권한 문자열 리턴 ROLE_ADMIN, ROLE_MANAGER ROLD_USER와 같이 사용자 role. ROLE_

```java
대게는 기본 AccessDecisionManager를 사용하고, 이것은AffirmativeBased  이다. (긍정기반/voter가 거부하지 않으면 허가되는)
```

만약 다른 표현식으로 범위 확장하려면 SecurityExpressionRoot 및 SecurityExpressionHandler의 커스터마이징이 필요하다

### Web Security

시큐리티? 필터활용해서 만든거다.
필터 Client 요청 -필터 - 필터 -필터 -서블렛

필터는 체인 형성하고 순서대로 실행된다 @Order로 사용할 수도 있다.

필터

```java
public class MyFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("필터2");
        chain.doFilter(request, response);
    }
}
```

```java
@Bean
    public FilterRegistrationBean<MyFilter> filter(){
        FilterRegistrationBean<MyFilter> bean = new FilterRegistrationBean<>(new MyFilter());
        bean.addUrlPatterns("/*");
        bean.setOrder(1); // 낮은 번호 필터가 먼저 실행된다.
        return bean;
    }
```

스프링 시큐리티는 하나의 필터로 설치되고, 필터체인프록시로 완성된다. 그리고 그 안에 특별한 역할을 추가하는 필터들이 있다

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b1a4571-18bd-4c45-81c4-4596c170c803/Untitled.png)

모든 요청에 적용된다

순수 스프링 부트 에는 보통 6개의 필터가 있다.
첫번 째는 /css/**, /images/** 정적 리소스 패턴
오류 뷰 /error 무시하기 위한 것

마지막 꺼는 /** 모든 경로에 매치 되며, 인증, 권한부여, 예외처리, 세션처리, 헤더쓰기 등을 위한 로직을 퐇마한다. 여기 11개의 필터가 세부적으로 있지만 알 필요 없다

### Creating and Customizing Filter Chains

기본 폴백 필터 체인 (/**)은 SecurityProperties.BASIC_AUTH_ORDER로 정해진 순서를 갖는다 security.basic.enabled = false로 설정을 끌수도 있고, 더 낮은 순위의 규칙을 정의할 수 있다. @Bean추가하고 @Order 쓰면 된다

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
     ...;
  }
}
```

이 빈은 스프링 시큐리티에 새로운 필터를 추가하고 폴백 이전의 우선순위를 갖는다

ui요청은 로그인 페이지로 리다이렉트 하는 쿠기 기반 인증처리로,
API요청은 404 응답을 리턴하는 토큰 기반 인증처리로 만들 수도 있다

### Request Matching for Dispatch and Authorization

보안 필터 체인(동일하게 WebSecurityConfigurererAdapter)은 http 요청을 처리할지를 결정하는 request matcher를 가지고 있다

특정 필터 체인이 결정되면 다른 체인은 적용되지 않는다. 추가적으로 matcher 설정하면 세분화된 권한 제어할 수 있다.

```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/foo/**")
      .authorizeRequests()
        .antMatchers("/foo/bar").hasRole("BAR")
        .antMatchers("/foo/spam").hasRole("SPAM")
        .anyRequest().isAuthenticated();
  }
}
```

### Combining Appliation Security Rules with Actuator Roles

스프링 액추에이터 : 운영 중인 애플리케이션 HTTP나 JMX를 이용해서 모니터링하고 관리할 수 있게 기능을 제공한다

스프링 시큐리티 어플리케이션에 액츄에이터 추가하면 자동으로 액츄에이터 추가필터체인 생긴다 (스프링 시큐리이테 스프링 액추에이터도 쓸 수 있게 스프링부트가 만들어져있다는 말인 것 같다)

기본 SecurityProeprties폴백 필터보다 5 작은 순서(ManagementServerProperties.BASIC_AUTH_ORDER)로 적용되므로 폴백 전에 참조된다.
엑츄에이터 기본보안설정을 선호하는 경우는 액츄에이터보다 +1 로 순서 설정.

@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)

### Method Security

메서드에도 보안 줄 수 있다.

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(securedEnabled = true)
public class SampleSecureApplication {
}
```

@EnableGlobalMethodSecurity(securedEnabled = true)
적어서 메서드에서 작동하도록 만든다음에

```java
@Service
public class MyService {

  @Secured("ROLE_USER")
  public String secure() {
    return "Hello Security";
  }

}
```

@PreAuthorize 및 @PostAuthorize 어노테이션

도 사용할 수 있다

### Working With Threads

스프링 시큐리티는 근본적으로 thread bound이다

현재의 인증된 principal을 다양한 downstream consumer들이 이용할 수 있게 해야하기 때문이다

1인 1개로 쓴다는 말인건가?

무수한 HTTP 요청으로 쓰레드가 여러 개 생겨도 ContextHolder에서 꺼낸 인증정보들이 고이지 않게 된다

### Processing Secure Methods Asynchronously

SecurityContext는 쓰레드 바운드이므로

보안 메서드를 호출하는 백그라운드를 수행하려는 경우

@Async를 사용하면 컨텍스트가 전달되는 지 확인해야 한다

Runnable과 Callable에 대해 좀 더 쉬운 헬퍼를 제공한다
