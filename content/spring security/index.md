---
emoji: 🧢
title: spring security
date: '2022-12-30 03:00:00'
author: 줌코딩
tags: spring security 
categories: 블로그 featured
---

# 스프링 시큐리티

spring security란 무엇인가?

우선 세가지 개념을 확실히 하자

1. 인증 (Authentication)
2. 권한 부여(Authorization)
3. Servlet Filter

인증이란 유저가 접속하는 계정의 주인이 맞는지 `식별` 하는 행위이다.

권한 부여란 식별된 유저가 접속한 어플리케이션에서 어떤 행위를 할수 있는 권한이 있는지를 `확인` 하는 행위이다.

스프링의 `DispatcherServlet` 은 상당히 고전적인 기술로써 http 요청을 `@Controller` 클래스에 리다이렉트한다. 기존의 `DispatcherServlet` 에 적용된 보안은 없다. 또한, 우리는 컨트롤러에서 HTTP auth 헤더에 직접 접근해서 이를 다루고 싶지 않다. 보안이 적용되야하는 최적의 시점은 바로 http요청이 `@Controller` 에 도달하기 전이다.

따라서 우리는 `filter` 를 서블렛의 전방에 배치하여 http 요청이 서블렛에 전달되기 전에 보안을 적용하여 걸러내게 된다.

SecurityFilter가 하는 일을 4가지로 매우 단순화 시켜본 코드는 다음과 같다.

```java
import javax.servlet.*;
import javax.servlet.http.HttpFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class SecurityServletFilter extends HttpFilter {

    @Override
    protected void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {

        UsernamePasswordToken token = extractUsernameAndPasswordFrom(request);  // (1)

        if (notAuthenticated(token)) {  // (2)
            // either no or wrong username/password
            // unfortunately the HTTP status code is called "unauthorized", instead of "unauthenticated"
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // HTTP 401.
            return;
        }

        if (notAuthorized(token, request)) { // (3)
            // you are logged in, but don't have the proper rights
            response.setStatus(HttpServletResponse.SC_FORBIDDEN); // HTTP 403
            return;
        }

        // allow the HttpRequest to go to Spring's DispatcherServlet
        // and @RestControllers/@Controllers.
        chain.doFilter(request, response); // (4)
    }

    private UsernamePasswordToken extractUsernameAndPasswordFrom(HttpServletRequest request) {
        // Either try and read in a Basic Auth HTTP Header, which comes in the form of user:password
        // Or try and find form login request parameters or POST bodies, i.e. "username=me" & "password="myPass"
        return checkVariousLoginOptions(request);
    }

    private boolean notAuthenticated(UsernamePasswordToken token) {
        // compare the token with what you have in your database...or in-memory...or in LDAP...
        return false;
    }

    private boolean notAuthorized(UsernamePasswordToken token, HttpServletRequest request) {
       // check if currently authenticated user has the permission/role to access this request's /URI
       // e.g. /admin needs a ROLE_ADMIN , /callcenter needs ROLE_CALLCENTER, etc.
       return false;
    }
}
```

1. 첫번째로, 필터는 http 요청에서 유저명과 패스워드를 `추출` 한다. 추출하는 방법에는 여러가지가 있을 수 있다. 기본적인 http auth 헤더나, form 항목 또는 쿠키가 해당된다.
2. 두번째는  `인증` 단계로 필터는 `db` 등의 데이터를 조회해서 유저명과 패스워드가 맞는지 확인한다.
3. 두번째 단계가 성공했다면 세번째는  `권한 체크` 단계로 유저가 요청한 `URI` 에 접근할 권한이 있는지를 확인한다.
4. 세번째 단계까지 모두 통과했다면, 필터는 요청이 마저 `DisatcherServlet` 으로 전달되도록 허용한다.

위의 클래스는 필터가 하는 4가지 사항을 매우 추상적인 수준에서 구현한 것이므로, 컴파일이 되긴 하지만 실제로 작동하기 위해서는 어마어마한 코드 라인 수를 가진 하나의 거대한 필터가 탄생할 것이다. 이를 방지하기 위해 우리는 4가지 작업을 담당하는 여러 개의 필터를 만들어 `필터 체인` 을 구성한다.

예를 들어,

<aside>
💡 유저명과 패스워드를 추출하는 필터 - 인증 필터 - 권한 체크 필터 - 서블렛 전달

</aside>

을 담당하는 각각의 필터를 만들어 체인을 구성할 것이다. 체인이 작동하기 위해서는 필터에서 최종적으로 다음의 메서드를 호출해서 다음 체인에 요청을 위임하면 된다.

`chain.doFilter(request,response);`

그럼이제 spring boot 프로젝트에서 `spring security` 를 세팅하고 어플리케이션을 실행해보자. 다음과 같은 로그가 보일것이다.

```java
2020-02-25 10:24:27.875  INFO 11116 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@46320c9a, org.springframework.security.web.context.SecurityContextPersistenceFilter@4d98e41b, org.springframework.security.web.header.HeaderWriterFilter@52bd9a27, org.springframework.security.web.csrf.CsrfFilter@51c65a43, org.springframework.security.web.authentication.logout.LogoutFilter@124d26ba, org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@61e86192, org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@10980560, org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter@32256e68, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@52d0f583, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@5696c927, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@5f025000, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@5e7abaf7, org.springframework.security.web.session.SessionManagementFilter@681c0ae6, org.springframework.security.web.access.ExceptionTranslationFilter@15639d09, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@4f7be6c8]|
```

횡으로 스크롤이 이렇게 긴 로그는 처음 봤다.

리스트업해보면 무려 15개의 필터(!)가 들어있다.

여기서 이 필터들을 전부 다루진 않겠지만, 간략하게 종류들을 소개하면 다음과 같다.

- **BasicAuthenticationFilter**:  기본 HTTP Auth 헤더를 찾은 뒤, 헤더에 들어있는 유저명과 비밀번호로 인증을 시도한다.
- **UsernamePasswordAuthenticationFilter**: 요청 파라미터 또는 POST 요청의 본문에서 유저명과 비밀번호를 탐색한뒤, 찾았다면 이 값들로 인증을 시도한다.
- **DefaultLoginPageGeneratingFilter**: 명시적으로 이 기능을 설정에서 끄지 않았다면 로그인 페이지를 자동으로 설정해준다. spring security를 설정하면 기본으로 로그인 페이지가 보이는 이유가 이 필터 때문이다.
- **DefaultLogoutPageGeneratingFilter**: 명시적으로 기능을 설정에서 끄지 않았다면 로그아웃 페이지를 자동으로 설정해준다.
- **FilterSecurityInterceptor**: 권한체크 과정을 수행한다.