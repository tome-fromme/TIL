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
- AuthenticationManager: 인증에 대한 주요 전략 인터페이스로 authenticate() 메소드 단 한개만 가지고 있음
```
    public interface AuthenticationManager {

      Authentication authenticate(Authentication authentication)
        throws AuthenticationException;

    }
```

    - authenticate()
        1) 유효한 principal이면 Authentication을 리턴
        2) 유효하지 않으면 AuthenticationException을 throw
            - AuthenticationException은 런타임 예외
            - 이는 대개 어플리케이션의 스타일과 목적에 따라 일반적인 방법으로 핸들링될 수도 있고, 직접 예외를 catch해서 핸들링할 수 있음
        3) 결정하지 못하면 null을 리턴