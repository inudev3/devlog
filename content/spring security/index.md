---
emoji: 🧢
title: spring security
date: '2022-12-30 03:00:00'
author: inu
tags: spring security 
categories: spring
---
출처: https://www.marcobehler.com/guides/spring-security
아래 글은 위 원문 포스팅의 번역글입니다.
원문 포스팅에는 이해를 돕는 보다 자세한 이미지

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

   엔티티나 데이터베이스 테이블과의 매핑 측면에서 어느 정도 쓸만한 기본 UserDetails 구현체이다. 대안으로는 역으로 엔티티가 UserDetails 인터페이스를 구현하도록 설계하는 방법이 있다.

그럼 이제 전체적인 UserDetailsService의 워크 플로우를 살펴보자.

1. 필터에서 사용자명/비밀번호 정보를 HTTP 기본 Auth 헤더로부터 추출한다. 이 동작은 자동으로 일어난다.
2. 앞서 구현한 *MyDatabaseUserDetailsService* 를 호출해 데이터베이스에서 유저를 조회한 뒤 이를 UserDetails 객체로 감싸서 반환한다. UserDetails 객체에는 해싱된 비밀번호가 들어있다.
3. HTTP 기본 Auth 헤더로부터 추출한 비밀번호를 자동으로 해싱하여, 이를 UserDetails 객체로부터 전달받은 해싱된 비밀번호와 비교한다. 둘이 일치하면, 유저는 인증을 통과한다.

그런데 스프링 시큐리티가 3번 과정에서 어떻게 자동으로 비밀번호를 해싱해주는 걸까? 바로 `PasswordEncoder` 빈을 사용한다.

스프링 시큐리티가 자동으로 패스워드 해싱 알고리즘을 등록해주진 않기 때문에 직접 빈을등록해 명시해줘야 한다.

모든 비밀번호에 동일한 해싱 알고리즘을 적용하고 싶다면 `SecurityConfig` 설정 클래스에 등록해준다. 예제는 BCrypt 알고리즘을 사용했다.

```java
@Bean
public BCryptPasswordEncoder bCryptPasswordEncoder() {
    return new BCryptPasswordEncoder();
}
```

하지만 만약, 여러 해싱 알고리즘 정책을 어플리케이션에서 사용한다면 다음과 같이 *delegating encoder*를 등록해줘야 한다.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

*delegating encoder* 는 다음과 같이 동작한다. UserDetail에 들어있는 해싱된 패스워드(바로 데이터베이스 테이블에 들어있는 패스워드) 는 접두어로 해싱 방법을 가리키는 단어를 갖는다. 예를 들어 다음과 같다.

스프링 시큐리티는 이 패스워드를 다음과 같이 처리한다.

1. 패스워드를 읽고 해싱 알고리즘을 가리키는 접두어를 제거한다. (`{bcrypt}` 나 `{sha256}` 등)
2. 접두어의 값에 따라 알맞는 PasswordEncoder를 사용한다.
3. 필터를 거쳐 추출된 비밀번호를 선택된 PasswordEncoder로 해싱한다. 해싱된 비밀번호를 저장된 비밀번호와 비교한다.

즉 UserDetailsService가 저장된 비밀번호를 갖고 오면, PasswordEncoder가 접두어로부터 해싱 알고리즘을 읽어 자동으로 필터로 들어온 입력 비밀번호를 해싱하고 비교한다.

요약하면, 스프링 시큐리티를 사용하며 고객의 비밀번호를 직접 접근할 수 있다면 아래의 두가지가 필요하다.

1. UserDetailsService를 등록. 커스텀 구현체를 사용해도 되고, 스프링 디폴트 구현을 사용해도 됨.
2. PasswordEncoder를 등록

**2:Authentication Provider:유저의 해싱된 비밀번호에 접근할 수 없는 경우**

인증 관리 서비스로서 Atlassian Crowd를 사용 중이라고 가정해보자.  이제 모든 인증정보는 Atlassian Crowd에 저장되고 데이터베이스 테이블에는 아무것도 없다.

이는 아래 2가지를 의미한다.

1. 어플리케이션에 더 이상 유저 정보가 없으며, 단순히 Crowd 서비스에게 비밀번호를 달라고 요청하는 방식은 작동하지 않는다.
2. 하지만 대신에 유저정보를 갖고 로그인을 요청할 REST API를 갖는다.(예를 들어 Crowd의 `/rest/usermanagement/1/authentication` REST 엔드포인트에 POST 요청을 할 수 있다)

이 경우, 이제 더 이상 UserDetailsService는 필요 없으며, 대신 `AuthenticationProvider` 인터페이스를 구현하여 빈으로 등록해야 한다.

AuthenticationProvider는 하나의 메서드로 구성되며, 단순하게 구현해보면 아래와 같다.

```java
public class AtlassianCrowdAuthenticationProvider implements AuthenticationProvider {

    Authentication authenticate(Authentication authentication)  // (1)
            throws AuthenticationException {
        String username = authentication.getPrincipal().toString(); // (1)
        String password = authentication.getCredentials().toString(); // (1)

        User user = callAtlassianCrowdRestService(username, password); // (2)
        if (user == null) {                                     // (3)
            throw new AuthenticationException("could not login");
        }
        return new UserNamePasswordAuthenticationToken(user.getUsername(), user.getPassword(), user.getAuthorities()); // (4)
    }
  // other method ignored
}
```

1. 유저명 만을 인자로 받았던 `UserDetailsService` 의 *loadByUsername* 메서드와 비교해 보면, 이 메서드는 (일반적으로는 유저명과 비밀번호로 구성된) 전체 인증 정보에 접근할 수 있는 것을 볼 수 있다.
2. 유저를 인증하기 위해 REST API를 호출하거나 그 외 원하는 뭐든지 할 수 있다.
3. 인증이 실패하면 예외를 던진다.
4. 인증이 성공하면, 이름도 긴 `UsernamePasswordAuthenticationToken` 을 반환해야 한다. 이 객체는 *Authentication* 인터페이스의 구현체로, boolen *authenticated* 필드가 true 인 상태여야 한다는 조건을 수반한다.

*AuthenticationProvider*의 워크플로우를 요약하자면 다음과 같다.

1. 자동으로 http 기본 Auth 헤더에서 유저 인증정보를 추출한다.
2. `AuthenticationProvider` 를 호출해서 대신 인증을 수행하도록 REST 호출 등을 수행한다.

UserDetailsService의 경우와 달리, 비밀번호 해싱 등이 없이 서드 파티 서비스에 인증정보 확인을 위임한다.

Spring Security를 사용하면서 유저 비밀번호에 대해 접근할 수 없는 경우, `AuthenticationProvider` 를 구현하여 등록하라.

### Spring Secuirty와 권한(Authority)

지금까지는 인증에 대해서만 다뤄보았다. 이제부터는, 권한과 역할을 Spring Security가 어떻게 다루는지 알아보자.

### 권한(Authorization)

권한이란 뭘까? 이커머스 쇼핑몰 사이트가 있다고 상상해보자. 이 사이트는 다음의 부분으로 이루어져 있을 것이다. 각 부분들 옆 괄호에는 url을 예시로 적었다.

- 쇼핑몰 (*`www.youramazinshop.com)`*
- 쇼핑몰 고객센터 상담원이 로그인하여 문의한 고객이 구입한 상품이나, 배송 중인 상품을 조회할 수 있는 페이지 (*`www.youramazinshop.com/callcenter`)*
- 관리자가 고객센터 상담원을 관리하거나 다른 기술적인 부분들을 관리할 수 있는, 분리된 페이지.(*`www.youramazinshop.com/admin)`*

이제 단지 유저를 인증하는 것만으로 충분치 않기 때문에 다음 사항들을 고려해야 한다.

- 고객은 당연히 콜센터나 관리자 페이지에 접근할 수 없어야 한다. 오직 쇼핑몰만 사용할 수 있다.
- 콜센터 상담원은 관리자 페이지를 이용할 수 없다.
- 반면 관리자는 쇼핑몰이나 콜센터 페이지를 이용할 수 있다.

요약하자면 각기 다른 유저들에 대해, 각각의 권한이나 역할에 따라 다른 접근 권한을 부여하고 싶다.

### 권한과 역할

간단하다.

- 권한은 단지 문자열이다. ADMIN, ROLE_ADMIN,12345, 뭐든지 다 될 수 있다.
- 역할은 접두어로 *ROLE*_ 이 붙는 권한이다. 따라서 *ADMIN*이라는 `ROLE` 은 *ROLE_ADMIN* 이라는 권한 `AUTHORITY` 와 동일하다.

권한과 역할의 구분은 순수하게 개념적인 부분이며, 많은 사람들이 이 부분에서 혼란스러워하지만 명쾌한 답은 없다. 데이터베이스 세계에서 두 개념은 내부적으로 완전하게 동일하게 동작한다.

### GrantedAuthorities와 SimpleGrantedAuthorities

Spring Security는 물론 문자열만 가지고 권한을 확인하지는 않는다. 권한을 나타내는 자바 클래스들이 제공되며, SimpleGrantedAuthorities가 대표적이다.

```java
public final class SimpleGrantedAuthority implements GrantedAuthority {

	private final String role;

  @Override
	public String getAuthority() {
		return role;
	}
}
```

권한에 관련된 다른 객체들을 필드로 가지는 클래스들이 더 있지만, 여기서는 다루지 않는다.

그렇다면 인증 과정에서 권한을 어디에 저장하고 어떻게 가져올까?

앞서 인증과정의 2가지 시나리오를 다뤄보았다. 해당시나리오별로 권한 체크 과정을 파악해보자.

**1. UserDetailsService**

첫번째는 어플리케이션에서 유저정보를 직접 저장하는 경우인 UserDetailsService가 되겠다. 직접 유저정보를 저장하기 위해서 데이터베이스에 `Users` 라는 테이블이 있다고 가정하자.

이 테이블에 단순하게 `Authorities` 라는 칼럼하나를 추가하도록 스키마를 변경할 수 있다. 이 예제에서는 이 칼럼을 단순한 varchar 칼럼으로 취급할 것이지만, 실제로는 쉼표로 구분된 다수의 문자열이 올 수 있다. 혹은, 아예 `Authorities`를 테이블로 분리할 수 있지만, 이 예제에서는 하지 않는다.

당신은 권한 *문자열* 을 데이터베이스에 저장하게 될 것이다. 가끔 이 문자열들이 *ROLE_*  접두어로 시작하는 경우가 있는데, 스프링에서는 이러한 *권한* 들이 곧 *역할* 이기도 하다는 점을 알면 된다.

이제 남은 건 UserDetailsService가 `authorities` 칼럼을 조회하여 반환할 UserDetails 객체에 넣어주는 일 뿐이다.

```java
public class MyDatabaseUserDetailsService implements UserDetailsService {

  UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
     User user = userDao.findByUsername(username);
     List<SimpleGrantedAuthority> grantedAuthorities = user.getAuthorities().map(authority -> new SimpleGrantedAuthority(authority)).collect(Collectors.toList()); // (1)
     return new org.springframework.security.core.userdetails.User(user.getUsername(), user.getPassword(), grantedAuthorities); // (2)
  }

}
```

1. 단순하게 데이터베이스에서 authorities 칼럼을 조회하여 *SImpleGrantedAuthority* 객체로 매핑한다. 복수의 문자열을 담고 있다면 컬렉션이 될 것이다.
2. 스프링 시큐리티가 디폴트로 제공하는 UserDetails 객체로 이를 감싸 반환한다. 물론 직접 구현체를 만들어도 된다.

**2. AuthenticationProvider**

써드파티 인증 앱을 사용하는 경우라면, 우성 해당 앱이 권한에 대해서 어떤 개념을 적용시키는지를 알아야 한다. `Atlassian Crowd` 의 경우, `role` 의 개념을 차용하는 대신 `group` 은 사용이 중단되었다.

따라서 이러한 앱의 스펙에 맞게 AuthenticationProvider의 구현을 변경해줘야 한다.

```java
public class AtlassianCrowdAuthenticationProvider implements AuthenticationProvider {

    Authentication authenticate(Authentication authentication)
            throws AuthenticationException {
        String username = authentication.getPrincipal().toString();
        String password = authentication.getCredentials().toString();

        atlassian.crowd.User user = callAtlassianCrowdRestService(username, password); // (1)
        if (user == null) {
            throw new AuthenticationException("could not login");
        }
        return new UserNamePasswordAuthenticationToken(user.getUsername(), user.getPassword(), mapToAuthorities(user.getGroups())); // (2)
    }
	    // other method ignored
}
```

1. 예제는 실제 `Atlassian Crowd` 를 사용하는 코드는 아니고, `Atlassian Crowd` 스펙에 맞게 작성해본 슈도 코드이다. REST 서비스를 통해 인증한 뒤 JSON 결과값을 받아 이를 atlassian.crowd.User 객체로 변환한다.
2. 유저는 어떤 그룹 또는 여러 그룹 소속일 수도 있고, 그룹은 단지 문자열일 뿐이다. 이를 스프링에서 디폴트로 제공하는 `SimpleGrantedAuthority` 구현체에 매핑하면 된다.

### ****WebSecurityConfigurerAdapter 와 권한(Authorities)****

지금까지는 권한을 저장하고 조회하는 기능에 집중했다.  그렇다면 제각기 다른 권한을 필요로하는 여러 URL에 대한 접근을 어떻게 설정할 수 있을까? ****WebSecurityConfigurerAdapter****의 설정 메서드 빈에서 DSL을 통해 가능하다.

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
        http
          .authorizeRequests()
            .antMatchers("/admin").hasAuthority("ROLE_ADMIN") // (1)
            .antMatchers("/callcenter").hasAnyAuthority("ROLE_ADMIN", "ROLE_CALLCENTER") // (2)
            .anyRequest().authenticated() // (3)
            .and()
         .formLogin()
           .and()
         .httpBasic();
	}
}
```

1. `/admin` 주소에 접근하기 위해서는 인증과 더불어 `ADMIN` 역할이 필요하다
2. *`/callcenter` 주소에* 접근하기 위해서는 인증과 더불어 `ADMIN`   또는 `CALLCENTER`역할이 필요하다
3. 그 이외에 모든 요청에 대해서는 특정한 역할이 필요없지만 기본적으로 인증되어야 한다.

위 코드는 아래 코드와 완전히 동일하다.

```java
 http
    .authorizeRequests()
      .antMatchers("/admin").hasRole("ADMIN") //(1)
			.antMatchers("/callcenter").hasAnyRole("ADMIN","CALLCENTER") //(2)
```

1. hasAuthority를 호출하는 대신에 hasRole을 호출하면 스프링 시큐리티는 인증된 유저의 정보를 조회하여 `ROLE_ADMIN` 이라는 권한을 찾는다.
2. hasAnyAuthority를 호출하는 대신에 hasAnyRole 스프링 시큐리티는 인증된 유저의 정보를 조회하여 `ROLE_ADMIN` 또는 `ROLE_CALLCENTER` 이라는 권한 중 하나를 찾는다.

### hasAccess와 스프링 표현식

마지막으로 권한을 설정하는 가장 강력한 방법으로 access가 있다. 이 메서드는 어떤 스프링 표현식이라도 사용하게 해준다.

```java
http
    .authorizeRequests()
      .antMatchers("/admin").access("hasRole('admin') and hasIpAddress('192.168.1.0/24') and @myCustomBean.checkAccess(authentication,request)")
```

1. 유저가 `admin` 역할이 있는지, 특정 ip주소를 사용했는지 확인하고 커스텀한 빈의 메서드를 적용시키고 있다.



```toc
```