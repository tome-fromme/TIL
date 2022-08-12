# 9. Servlet Security: The Big Picture
## 9.1 A Review of Filters
- 스프링 시큐리티는 서블릿 filter 기반으로 기반으로 서블릿 지원
<img src="https://godekdls.github.io/images/springsecurity/filterchain.png">
- 클라이언트는 어플리케이션으 요청을 전송하고 컨테이너는 서블릿과 여러 필터들로 구성된 Filter chain을 만들어 요청 URI path 기반으로 HttpServletRequest 처리
- 스프링 MVC 에서 서블릿은 DispatcherServlet
- 단일 HttpServletRequest와 HttpServletResponse 처리는 최대 한개의 Servlet이 담당
- 하지만 filter는 여러개 사용 가능
- filter 사용
  - 다운스트림 Servlet과 여러 filter의 실행을 막음. 이경우엔 보통 filter에서 HttpServletResponse를 작성
  - 다운스트림에 있는 Servlet과 여러 filter로 HttpServletRequest나 HttpServletResponse를 수정
````
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    // 나머지 응용 프로그램보다 먼저 수행
    chain.doFilter(request, response); // 나머지 응용 프로그램 호출
    // do something after the rest of the application
}

````
- filter는 다운스트림에 있는 나머지 filter와 servlet에만 영향을 주기 때문에 filter의 실행 순서는 중요

## 9.2 DelegatingFilterProxy
## 9.3 FilterChainProxy
## 9.4 SecurityFilterChain
## 9.5 Security Filters
## 9.6 Handling Security Exceptions