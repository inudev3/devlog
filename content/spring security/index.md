---
emoji: 🧢
title: spring security
date: '2022-12-30 03:00:00'
author: 줌코딩
tags: spring security 
categories: 블로그
---
출처: https://www.marcobehler.com/guides/spring-security
아래 글은 위 포스팅의 버녕

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


### Spring Security 설정

Spring Security를 설정하기 위해서는 다음 2가지를 충족하는 클래스가 있어야 한다.

1. `@EnableWebSecurity` 어노테이션이 붙어있고,
2. 설정 DSL 메소드를 을 제공하는 `WebSecurityConfigurerAdapter` 클래스를 상속받아야 한다. 이 DSL 메소드로 어플리케이션의 어떤 URI를 허용할지/ 보호할지 유무와 어떤 보호수단을 적용하거나 적용하지 말지를 설정한다.

설정 클래스 예시는 다음과 같다.

```java
@Configuration
@EnableWebSecurity // (1)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter { // (1)

  @Override
  protected void configure(HttpSecurity http) throws Exception {  // (2)
      http
        .authorizeRequests()
          .antMatchers("/", "/home").permitAll() // (3)
          .anyRequest().authenticated() // (4)
          .and()
       .formLogin() // (5)
         .loginPage("/login") // (5)
         .permitAll()
         .and()
      .logout() // (6)
        .permitAll()
        .and()
      .httpBasic(); // (7)
  }
}
```

1. 설정 빈에 사용하는 스프링 @Configuration 어노테이션과 @EnableWebSecurity 어노테이션을 사용하고, WebSecurityConfigurerAdapter를 상속한다.
2. adapter 클래스의 configure(HttpSecurity) 메서드를 상속하여 필터체인을 설정하는 DSL을 사용할 수 있다.
3. 이 예시에서,  *`/`* 나 *`/home`*  경로로 가는 모든 요청은 허용되므로 유저는 인증을 거칠 필요가 없다. 예시에서 처럼 `antMatcher` 를 사용하면 (*, \*\*, ?) 와 같은 와일드카드 문자를 사용할 수 이
4. 그 외의 모든 요청은 우선 인증을 필요로 한다.
5. 예시에서는 는 스프링의 디폴트 로그인 페이지가 아닌 커스텀한 `/login` 를 갖는 로그인 페이지를 통해 `formLogin` 을 할 수 있게 허용하고 있다. permitAll로 로그인 전이더라도 `/login` 페이지에 항상 접근할 수 있게끔 한다. 로그인 되지 않은 상태에서 `/login` 페이지에 접근할 수 없다면 곤란한 상황이 발생할 것이다.
6. 로그아웃 페이지도 동일하다.
7. 거기에 더해 httpBasic 메서드를 통해 인증에 필요한 기본 HTTP Auth 헤더를 전달한다.

익숙해 지는데 시간이 걸리겠지만,  `configure` 메서드에서 다음 사항을 명시해야 한다는 점을 기억하자.

What is important for now, is that *THIS* *`configure`* method is where you specify:

1. 어떤 URL에 보호를 적용할지 (`authenticated()`) 그리고 어떤 경로를 허용할지 (`permitAll()`).
2. 어떤 인증 수단을 사용할지(`formLogin()`, `httpBasic()`) 그리고 그 수단을 어떻게 적용할지
3. 한마디로 말해 어플리케이션의 전체 보안 설정

참고로, 디폴트 `configure` 메서드의 모습은 다음과 같다.

```java
public abstract class WebSecurityConfigurerAdapter implements
		WebSecurityConfigurer<WebSecurity> {

    protected void configure(HttpSecurity http) throws Exception {
            http
                .authorizeRequests()
                    .anyRequest().authenticated()  // (1)
                    .and()
                .formLogin().and()   // (2)
                .httpBasic();  // (3)
        }
}
```

1. 어떤 경로를 접근하던지 상관없이 인증을 필요로 한다.
2. 스프링 디폴트 formLogin()이 설정되어 있다.
3. httpBasic 인증도 설정되어 있다.(http auth 헤더)



이 디폴트 구현이 스프링 시큐리티를 적용하면 바로 로그인 페이지가 나오는 이유이다.

그럼 이제 `BasicAuthFilter` 에 대해 알아

BasicAuthFilter는 유저명과 비밀번호를 추출한다고 했는데, 이 인증정보들을 어디서 확인하는 걸까?

이를 알기 위해서는 Spring Security 인증이 어떻게 이루어지는지를 알아야 한다.

Spring Security에서 인증(Authentication)에는 3가지 시나리오가 있다.

1. 데이터베이스 테이블 등에 저장된 인증정보(유저명, 비밀번호)가 있어서 유저의 해싱된 비밀번호에 접근할 수 있는 경우. 가장 일반적인 경우이다.
2. 유저의 해싱된 비밀번호의 접근할 수 없는 경우로, 인증정보가 써드 파티 인증 관리 서비스 제품 등의 다른 곳에 저장된 경우이다. 보다 드문 경우라고 할 수 있다. `Atlassia crowd`등이 이러한 써드 파티 제품으로 인증을 위한 REST 서비스를 제공한다.
3. OAuth2나 OpenId 등과 JWT를 조합하여 사용하는 경우. 이 경우는 OAuth2를 다루는 별도의 장에서 알아본다.

시나리오에 따라 각기 다른 스프링 빈을 등록해야 스프링 시큐리티가 정상적으로 작동하며, 그렇지 않을 경우 예외가 발생하기 때문에 주의해야 한다.(`PassWordEncoder` 빈을 등록하지 않았을 때 NPE가 발생하는 등)

우선 1번과 2번 시나리오에 따른 빈을 알아보자.

**1:UserDetailService: 유저의 해싱된 비밀번호에 접근할 수 있는 경우**

데이터 베이스에 유저 정보를 저장하는 테이블이 있고, 거기에 유저명과 해싱된 비밀번호 칼럼이 있다고 가정하자.

이 경우  인증이 동작하기 위해서 스프링 시큐리티는 2개의 빈을 필요로 한다.

1. UserDetailsService 빈.
2. PasswordEncoder 빈.

UserDetailService 빈은 다음과 같이 간단하게 정의할 수 있다.

```java
@Bean
public UserDetailsService userDetailsService() {
    return new MyDatabaseUserDetailsService(); // (1)
}
```

1.  MyDatabaseUserDetailsService는 UserDetailsService의 구현체이며, UserDetailsService는 하나의 메서드 만을 가지는 매우 간단한 인터페이스이다. 이 메서드는 UserDetails 객체를 반환한다.

```java
public class MyDatabaseUserDetailsService implements UserDetailsService {

	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException { // (1)
         // 1. 유저 테이블에서 유저명으로 유저정보를 조회한다. 만약 조회에 실패하면 UsernameNotFoundException 예외를 발생시킨다.
         // 2. 유저정보를 UserDetails 객체로 감싸 반환한다.
        return someUserDetails;
    }
}

public interface UserDetails extends Serializable { // (2)

    String getUsername();

    String getPassword();

    // <3> 이 이상의 메서드들이 가능함
    // isAccountNonExpired,isAccountNonLocked,
    // isCredentialsNonExpired,isEnabled
}
```

1. UserDetailService는 유저명으로 UserDetails를 불러온다.  패스워드를 받는게 아니라 유저명을 단 한개뿐인 인자로 받는다는 점에 주의하자.
2. UserDetails 인터페이스는 해싱된 패스워드를 반환하는 메서드와 유저명을 반환하는 메서드를 갖느다.
3. UserDetails는 더 많은 메서드들을 가질 수 있다. 계정이 차단되었는지 유무, 인증정보가 유효한지, 혹은 유저의 권한이 무엇인지 등등. 여기선 다루지 않는다.

그러므로 이 2가지 인터페이스들을 직접 구현하던지, 아니면 스프링 시큐리티가 기본으로 제공하는 구현을 사용하면 된다.

스프링 시큐리티의 디폴트 구현체 외에도, 몇가지 구현체들을 추가적으로 제공한다.

1. **JdbcUserDetailsManager**

   JDBC 기반의 UserDetailService 구현체로 유저정보 테이블이나 칼럼 구조에 맞게 설정할 수 있다.

2.  **InMemoryUserDetailsManager**

    유저 정보를 메모리 상에서 관리하는 구현체로 테스팅에 용이하다.

3. **org.springframework.security.core.userdetail.User**

   which is a sensible, default UserDetails implementation that you could use. That would mean potentially mapping/copying between your entities/database tables and this user class. Alternatively, you could simply make your entities implement the UserDetails interface.