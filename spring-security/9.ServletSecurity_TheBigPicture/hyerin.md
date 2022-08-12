# 9. Servlet Security: The Big Picture
## 9.1 A Review of Filters
- 스프링 시큐리티는 서블릿 filter 기반으로 기반으로 서블릿 지원
<img src="https://godekdls.github.io/images/springsecurity/filterchain.png">
- 클라이언트는 애플리케이션에 요청을 보내고 컨테이너는 요청 URI의 경로를 기반으로 HttpServletRequest를 처리해야 하는 필터와 서블릿을 포함하는 FilterChain을 생성
- Spring MVC 애플리케이션에서 Servlet은 DispatcherServlet의 인스턴스
- 최대 하나의 서블릿이 단일 HttpServletRequest 및 HttpServletResponse를 처리할 수 있음
- 두 개 이상의 필터를 사용하여 다음을 수행할 수 있음
  - 다운스트림 필터 또는 서블릿이 호출되지 않도록 함. 이 경우 필터는 일반적으로 HttpServletResponse를 작성
  - 다운스트림 필터 및 서블릿에서 사용하는 HttpServletRequest 또는 HttpServletResponse 수정
````
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    // 나머지 응용 프로그램보다 먼저 수행
    chain.doFilter(request, response); // 나머지 응용 프로그램 호출
    // do something after the rest of the application
}
````
- 필터는 다운스트림 필터와 서블릿에만 영향을 미치기 때문에 각 필터가 호출되는 순서는 매우 중요

## 9.2 DelegatingFilterProxy

- Spring은 Servlet 컨테이너의 라이프사이클과 Spring의 ApplicationContext 사이의 연결을 허용하는 DelegatingFilterProxy라는 필터 구현체 제공
- Servlet 컨테이너는 자체 표준을 사용하여 필터를 등록할 수 있지만 Spring에서 정의한 Bean을 인식하지 못함
- DelegatingFilterProxy는 표준 서블릿 컨테이너 메커니즘을 통해 등록할 수 있지만 모든 작업을 Filter를 구현하는 Spring Bean에 위임
<img src="https://godekdls.github.io/images/springsecurity/delegatingfilterproxy.png">
- DelegatingFilterProxy는 ApplicationContext에서 Bean Filter0을 찾은 다음 Bean Filter0을 호출
````
// DelegatingFilterProxy 슈도코드
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    // Spring Bean으로 등록된 Filter를 Lazily get
    // DelegatingFilterProxy의 예에서 delegate는 Bean Filter0의 인스턴스
    Filter delegate = getFilterBean(someBeanName);
    // Spring Bean에 작업 위임
    delegate.doFilter(request, response);
}
````
- DelegatingFilterProxy의 또 다른 이점은 필터 빈 인스턴스를 찾는 것을 지연시킬 수 있다는 것
- 이는 컨테이너가 시작되기 전에 필터 인스턴스를 등록해야 하기 때문에 중요
- 그러나 Spring은 일반적으로 ContextLoaderListener를 사용하여 Filter 인스턴스를 등록해야 할 때까지 수행되지 않는 Spring Bean을 로드

## 9.3 FilterChainProxy
- Spring Security의 Servlet 지원은 FilterChainProxy에 포함되어 있음
- FilterChainProxy는 SecurityFilterChain을 통해 여러 Filter 인스턴스에 위임을 허용하는 Spring Security에서 제공하는 특별한 Filter
- FilterChainProxy는 Bean이므로 일반적으로 DelegatingFilterProxy에 래핑됨
<img src="https://docs.spring.io/spring-security/site/docs/5.3.2.RELEASE/reference/html5/images/servlet/architecture/filterchainproxy.png">

## 9.4 SecurityFilterChain
## 9.5 Security Filters
## 9.6 Handling Security Exceptions