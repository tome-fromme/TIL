# Spring Security Topical guide

## Spring Security
- Spring 기반의 애플리케이선의 보안(인증, 인가)을 담당하는 스프링 하위 프레임워크
- 인증과 인가에 대한 부분을 filter 흐름에 따라 처리

## 인증 vs 인가
- 인증 (Authentication): 해당 사용자가 본인이 맞는지 확인. Who are you?
- 인가 (Authorization): 인증된 사용자가 요청한 자원에 접근 가능한지 확인. What are you allowed to do?

- Spring Security는 기본적으로 인증 절차를 거친 후에 인가 절차를 진행하게 되며, 인가 과정에서 해당 리소스에 대한 접근 권한이 있는지 확인
- Spring Security에서는 이러한 인증과 인가를 위해 Principal을 아이디로, Credential을 비밀번호로 사용하는 Credential 기반의 인증 방식을 사용
    - Principal(접근 주체): 보호받는 Resource에 접근하는 대상
    - Credential(비밀번호): Resource에 접근하는 대상의 비밀번호
    

## 인증(Authentication)
### AuthenticationManager
- 인증에 대한 주요 전략 인터페이스로 authenticate() 메소드 단 한개만 가지고 있음
```
public interface AuthenticationManager {

  Authentication authenticate(Authentication authentication)
    throws AuthenticationException;

}
```

#### authenticate()
1) 유효한 principal이면 Authentication을 리턴
2) 유효하지 않으면 AuthenticationException을 throw
    - AuthenticationException은 런타임 예외
    - 이는 대개 어플리케이션의 스타일과 목적에 따라 일반적인 방법으로 핸들링될 수도 있고, 직접 예외를 catch해서 핸들링할 수 있음
3) 결정하지 못하면 null을 리턴

### ProviderManager
- AuthenticationManager의 가장 일반적인 구현체
- AuthenticationProvider는 호출자자 해당 인증유형을 지원하는지 확인할 수 있는 메소드를 제공
- 일련의 AuthenticationProverder 인스턴스들에게 위임
```
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);

}
```
- Class<?> 는 실제로는 Class<? extends Authentication>
- authenticate() 메소드에 전달될 어떤 것을 지원하는지 여부를 물음
- ProviderManager는 일련의 AuthenticationProvider들에게 위힘함으로써 동일 어플리케이션 내에서 여러 다른 인증 메커니즘을 제공
- ProviderManager가 특정 인증 유형을 인식하지 못한다면 그 인증은 그냥 스킵됨

<img src="https://t1.daumcdn.net/cfile/tistory/2206624D5935160D2C">
<img src="https://gregor77.github.io/images/spring-security/03/AuthenticationManager.png">


## Customizing Authentication Managers
- 스프링 시큐리티는 일반적인 인증관리 특징들을 빠르게 설정할 수 있는 헬퍼들을 제공
- 가장 많이 사용하는 헬퍼는 AuthenticationManagerBuilder 이고, 메모리, JDBC, LDAP 사용자 정보를 셋팅하고 커스텀 UserDetailService를 추가할 수 있음
```
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

   ... // web stuff here

  @Autowired
  public initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```
```
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

  @Autowired
  DataSource dataSource;

   ... // web stuff here

  @Override
  public configure(AuthenticationManagerBuilder builder) {
    builder.jdbcAuthentication().dataSource(dataSource).withUser("dave")
      .password("secret").roles("USER");
  }

}
```
- 스프링부트는 별도의 AuthenticationManager를 선언하지 않으면 기본 전역 AuthentiionManager(with just one user)를 제공
- 커스텀 global AuthenticationManager가 필요하지 않다면 이 디폴트만으로 충분
- 필요하다면 지역적으로 보호가 필요한 자원에 대해서만 AuthenticationManager를 추가할 수 있음

## Authentication or AccessControl
- 인증이 되었다면 다음 인가를 처리할 수 있고, 이것의 핵심은 AccessDecisionManager
- 프레임워크에서 제공되는 3가지 구현체가 있고. 이 3가지는 AccessDecisionVoter 체인에 위임
- ProviderManager가 AuthenticationProviders 들에게 위임하는 것과 유사

<img src="https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FYMRYo%2FbtqNaE4hAfi%2F6FcJ85nXrRyDHxOneoDUK1%2Fimg.png">
- FilterSecurityInterceptor(권한을 처리하는 필터)가 AccessDecisionManager 에게 decide(authentication(인증정보), object(요청정보), configAttributes(권한정보)) 3가지 정보를 AccessDecisionManager 에게도 전달
- 이후에 AccessDecisionManager는 자신이 가지고 있는 Voter클래스 에게도 3가지 정보를 전달해서 권한심사 처리를 위임
- Voter들은 3가지의 정보를 기반으로 허용 또는 거부를 판단
- 최종적으로 승인되면 FilterSecurityInterceptor필터에게 전달
- 거부가 되면  AccessDeniedException가 호출되어서 Exception TranslationFilter로 전달하게 되고 예외처리

### AccessDecisionManager
- Access Control or Authorization 결정을 내리는 인터페이스
- 인증이 완료된 사용자가 리소스에 접근하려고 할때 해당 요청을 허용할것인지 판단하는 인터페이스
- 인터페이스 구현체는 3가지를 기본으로 제공
- AccessDecisionManager는 decide 1개와 supports 메서드 2개를 제공
- AccessDecisionManager 를 구현하는데 필수적으로 구현해야하는 메서드는 decide() 메서드
- decide()는 Authentication, ConfigAttribute 등을 전달받으며, 전달받은 parameter에 대한 access 제어를 결졍하는 역할. 이때 만약 거부된다면 AccessDeniedException을 던짐
- supports()는 각각 전달받은 parameter를 처리 가능한지 판단. 전달받고 있는 ConfigAttribute는 권한을 나타내는 문자열을 말함(ex. ROLE_ADMIN)
- Voter라는 개념을 가지고 있음
- 투표를 기반으로 하는 AccessDecisionManager는 AccessDecisionVoter를 투표에 이용
```
// AffirmativeBased 클래스에서 decide 메서드

@Override
@SuppressWarnings({ "rawtypes", "unchecked" })
public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
		throws AccessDeniedException {
	int deny = 0;
	for (AccessDecisionVoter voter : getDecisionVoters()) {
		int result = voter.vote(authentication, object, configAttributes);
		switch (result) {
		case AccessDecisionVoter.ACCESS_GRANTED:
			return;
		case AccessDecisionVoter.ACCESS_DENIED:
			deny++;
			break;
		default:
			break;
		}
	}
	if (deny > 0) {
		throw new AccessDeniedException(
				this.messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
	}
	// To get this far, every AccessDecisionVoter abstained
	checkAllowIfAllAbstainDecisions();
}
```

### AccessDecisionVoter
- Spring Security는 투표를 기반으로 request에 대한 access에 대한 승인 여부를 결정
- Voter는 의사결정을 내리는데 사용 하며 여러개의 Voter를 가질 수 있음
- 스프링 시큐리티가 기본적으로 3개의 인터페이스 구현체를 제공
	- AffirmativeBased: 여러 Voter중에 하나라도 허용되면 허용 (기본 전략)
	- ConsensusBased: 다수결
	- UnanimousBased: 만장일치
- 모든 Voter가 허용하지 않는다면 예외를 발생
- vote()는 소스에 정의된 1, 0, -1 값을 반환하며, 각각은 승인, 거부, 보류를 나타냄
- 승인 결정에 대한 의견이 없는 경우 ACCESS_ABSTAIN, 승인인 경우는 ACCESS_DENIED를 거부일 경우는 ACCESS_GRANTED를 반환

```
public interface AccessDecisionVoter<S> {

   int ACCESS_GRANTED = 1;
   int ACCESS_ABSTAIN = 0;
   int ACCESS_DENIED = -1;

   int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
   boolean supports(ConfigAttribute attribute);
   boolean supports(Class<?> clazz);
}
```
- AccessDecisionVoter를 보면 AccessDecisionManager가 권한심사를 맡기는데 vote라는 메소드에도 authentication(인증정보), object(요청정보), configAttributes(권한정보) 들을 파라미터로 받고 있음
- 실질적인 권함심사를 한다고 보면됨

## Web Security
