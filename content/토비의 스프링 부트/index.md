---
emoji: 
title: inflearn 토비의 스프링 부트
date: '2023-02-04 03:00:00'
author: inu
tags: spring springboot
categories: spring
---

# 스프링 부트

스프링 부트란 스프링 기반 어플리케이션을 만들 수 있도록 도와주는 도구(기술)이다.

특징은 스프링 기반 `독립실행형` standalone 어플리케이션을 만들어주는 도구라는 점이다.

기존에 스프링으로 개발을 시작하는 것은 고민할 지점이 너무 많아 시작을 빠르게 하기가 어려웠다. 고민해야할 선택지가 많았다는 것이다.

스프링부트는 이 선택지를 줄여주는 것이 특징이다.

이러한 빠른 설정을 도와주는 도구들은 초기 어플리케이션이나 MVP 출시에는 적합하지만, 본격적인 엔터프라이즈 어플리케이션으로 확장하는 데에 있어서는 한계에 부딪힌다.

스프링 부트는 `강한 주장` 을 가지고, 즉시 적용 가능한 기술 조합을 제공하면서 빠르게 어플리케이션을 시작할 수 있게 해줌과 동시에 원한다면 언제나 이를 변경해서 사용할 수 있게 해준다.

스프링 부트의 시작점은 스프링이 컨테이너레스 `containerless` 아키텍쳐를 지원해달라는 어떤 개발자의 요청으로부터 출발했다.

### Containerless Architecture

`Containerless Architecture` 란 무엇일까? 이는 `Serverless` 와 유사하다. 서버의 설치와 관리에 신경쓰지 않고 운영하는 것을 말한다.

어떠한 기능을 수행하는 Web Component를 개발했다고 해보자. 웹 컴포넌트는 혼자서는 기능하지 못한다. 기능하기 위해서는 이를 사용하는 Web Client가 필요하다. 웹 컴포넌트로 개발하는 기능은 주로 동적인 컨텐츠를 표시하기 위한 경우가 많다. 정적인 경우 컴포넌트가 아니라 정적 컨텐트를 그저 서버에 올리면 그만이다. 웹 컴포넌트는 동적인 컨텐츠를 만들어서 이를 다시 웹 클라이언트한테 응답으로 보내므로, 요청과 응답의 쌍 Pair로 동작하게 되어있다.

이 때 웹 컴포넌트는 혼자서 실행될 수는 없고 컨테이너 안에 있어야 한다. `web container` 란 웹 컴포넌트를 관리하는 (컴포넌트를 메모리에 올리고, 컴포넌트의 진입점이 될 인스턴스를 생성하고, 서비스가 될 동안 메모리에서 동작하도록 해주는 일종의 라이프사이클 관리) 역할을 한다. 또한 웹 컴포넌트는 하나 뿐만 아니라 많은 종류의 서비스를 각각 담당하는 여러 개가 존재할 수 있다. 또한, 클라이언트로 들어온 요청을 어떤 컴포넌트가 담당할 지를 결정해서 컴포넌트에 연결해줘야 한다. 이런 작업을 라우팅 또는 맵핑이라고 한다.

자바에서 이런 웹 컴포넌트를 부르는 호칭이 바로 `서블릿` servlet 이다. 그리고 가장 유명한 컨테이너에는 톰캣이 있다.

스프링 또한 컨테이너이다. 그러나 서블릿 컨테이너와는 다르다. 스프링 컨테이너는 서블릿 컨테이너 이후에 존재하며, 서블릿이 아닌 `빈` 을 담고 있는 빈 컨테이너이다. 서블렛 컨테이너를 통해서 들어온 웹 요청을 스프링 컨테이너가 전달받아 빈이 여러 요청을 처리하게끔 하여 어플리케이션을 동작하게끔 한 뒤 다시 응답을 서블릿 컨테이너 한테 넘겨준다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/970af7a8-369c-4c1d-9b9b-8d130ffd3e56/Untitled.png)

그러면 스프링 컨테이너가 그냥 서블릿 컨테이너의 역할도 하면 안되는건가? 자바의 표준 웹기술을 사용하기 위해서는 서블릿 컨테이너가 존재해야 한다. 웹 요청을 받고 다시 응답을 보내주는 동작을 무언가 하나라도 하려면 서블릿 컨테이너는 필수이다. 그러나 개발자는 스프링의 빈 컨테이너로써의 역할에만 집중하여 어플리케이션을 개발하고 싶지, 서블릿 컨테이너를 배포하고, 설정하는 일은 별로 신경쓰고 싶지 않다. 서블릿 컨테이너를 띄우는 일은 여간 복잡한 일이 아니기 때문이다. `web.xml` , `war` , 배포 등등.. 스프링을 시작할 때마다 작성해야 하지만, 실제로는 거의 대부분 복사해서 붙여넣는 부분들이다. 개발할 때는 신경쓰지 않기 때문이다.

서블릿 컨테이너를 준비하기 위해 해야하는 일은 다음과 같다.

- 프로젝트 디렉토리 구조
- web.xml 설정
- 독립적인 서버 프로그램인 서블릿 컨테이너(톰캣 등)을 실행하고 WAR 서블릿 어플리케이션을 배포
- 배포하는 방식도 다양하고 충돌/에러 등도 다양
- port, 클래스로더, 로깅 등등

더 큰 문제는 서블릿 컨테이너는 표준 스펙일 뿐이며, 이를 구현한 다양한 구현체 중 하나를 선택하고 그에 맞는 설정방법/배포 방식등을 적용해야 한다는 점이다. (전부 다르다)

학습곡선도 높고, 활용하기도 쉽지 않다(프로젝트 초반에만 사용한다)

그래서 컨테이너레스 아키텍쳐란 Servlet Container가 없는 어플리케이션이 아니라, Servlet Container를 설치/관리하는 데에 개발자의 수고가 들지 않는 방식을 말한다.

스프링 부트는 서블릿 컨테이너에 필요한 설정을 전부 제공한다.

거기에 더해, 서블릿 컨테이너를 실행하는 과정또한 생략한다. 따로 서블릿 컨테이너를 실행하는게 아니라 스프링 어플리케이션을 실행하면 서블릿 컨테이너와 스프링 컨테이너가 한번에 같이 실행되는 방식이다. 이를 독립실행형 `standalone` 어플리케이션이라고 한다.

스프링 부트는 Opinionated 되어 있다고 한다.

이것의 의마하는 바는 개발자는 고객 가치를 창출하기 위한 소프트웨어 도메인에만 집중하고, 프로젝트를 설정하는 데에 드는 고민은 하지 말라는 얘기다. 즉 `내가 정해줄 테니 너는 개발만 해` 라는 스프링 부트의 철학을 자기주장이 강하다는 뜻에서 `Opinionated` 라고 한다.

스프링 프레임워크는 20년의 역사를 가졌고, 그 역사를 관통하는 프레임워크의 철학들은 아래와 같다.

1. 극단적인 유연함을 추구한다.
2. 다양한 관점을 수용
3. Not Opinionated
4. 수많은 선택지를 다 포용

이는 장점이지만 새로운 프로젝트를 시작할 때 개발자들에게 던져지는 선택지들이 꽤 긴 시간의 고민을 필요로 한다는 단점이 있다.

스프링부트는 수십년 간 쌓여온 스프링의 베스트 프랙티스를 방법을 선택하고 그로부터 출발하도록 `제안`한다.

스프링부트는 어플리케이션이 사용할 기술 , 라이브러리나 버전 등의 선택지를 제안한다. 또한 다양한 구현체 중에서도 검증된 선택지를 결정하여 `표준 구성` 으로 제안한다.

스프링 코어 프로젝트만 보더라도, 코어 프레임워크의 각 모듈이 사용하는 기술과 , 의존하는 라이브러리와 버전 등에 대한 선택지는 수없이 많다. 스프링 부트는 이를 전부 표준 구성으로 제안한다. 또한 각 기술을 스프링에 빈으로 적용하는 DI 구성이나 디폴트 설정값도 어느정도 표준 구성에 포함한다. 물론 비어있는 부분도 있다.

그런데 이런 자동 구성 기술의 단점이라고 할 수 있는, 커스터마이징 가능 여부는 어떨까? 스프링 부트는 자기주장이 강하지만 내장된 디폴트 구성을 커스터마이징 할 수 있는 굉장히 유연한 방식을 제공한다. 또한, 스프링 부트가 스프링을 사용하는 방식을 잘 이해하고 있다면, 언제라도 스프링 부트의 표준 구성을 전부 제공하고 원하는 방식으로 재구성할 수 있다. 또는, 스프링 부트가 제안하는 표준 구성처럼 내가 선택한 기술과 구성을 제공하는 모듈을 만들 수도 있다.

기술의 이해와 기술의 활용은 별개이다. 그러나 기술의 이해가 활용에 도움을 주기도 하고, 코드에 응용할 수도 있다.

스프링 부트를 예로 든다면, 부트가 설정한 디폴트 설정과 구성을 그대로 수용하고, 코드에만 집중해서 빠르게 개발을 시작할 수도 있다. 외부 설정 파일(application.yml) 을 통해 설정을 변경할 수도 있다. 그러나 이 방법은 어느 단계에서는 한계에 봉착하게 될 수도 있다.

스프링 부트를 이해함으로써 우리는

- 스프링 부트가 스프링의 기술을 어떻게 활용하는지 학습하고 응용할 수 있다.
- 스프링 부트가 선택한 기술 / 표준 구성이나 디폴트 설정을 확인할 수 있다.
- 필요할 때 부트의 설정을 수정하거나 확장할 수 있다.
- 나만의 스프링 부트 모듈을 만들어 활용할 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ab00bdf7-247f-4e95-a181-56dd3feaa8bf/Untitled.png)

자바 서블릿은 표준 컴포넌트이다. 서블릿은 컨테이너에 여러 개가 들어갈 수 있으며, 컨테이너는 요청을 어떤 서블릿에 전달할지를 결정하는데 이를 `Mapping`이라고 한다. 서블릿은 웹 응답을 만들기 위해 필요한 작업을 수행하고 응답을 돌려주면서 작업을 종료한다.

웹요청과 웹응답

웹 요청과 웹 응답은 다음처럼 이루어져 있다.

http는 웹 요청과 웹 응답의 표준 프로토콜이다.

Request

- Request Line: Method, Path(hostname과 port를 제외한 경로), HTTP Version
- Headers - 요청을 처리하는 방식, 응답 생성 방식등을 지정
- Message Body - Header에 설정된Content-Type에 맞는 메세지 본문

Response

- Status Line: HTTP Version, Status Code(상태 코드, 클라이언트가 응답의 상태를 파악할 수 있다), Status Text(응답 코드를 설명하는 텍스트)
- Headers - 메세지 바디의 형식(Content-Type:필수), 그 외에 서버가 클라이언트에 보내고 싶은 정보들
- Message Body

직접 컨테이너를 구성해서 서블릿을 등록하고 맵핑해보자.

```kotlin
class HellobootApplication
fun main(args: Array<String>) {
    val serverFactory = TomcatServletWebServerFactory()
    serverFactory.getWebServer( ServletContextInitializer{ servletContext:ServletContext? ->
        servletContext?.addServlet("hello", object :HttpServlet(){
            override fun service(req: HttpServletRequest?, resp: HttpServletResponse?) {
                super.service(req, resp)
            }
        })?.addMapping("/hello")
    }).start()
}
```

spring boot가 어플리케이션을 실행해주는 @SpringBootApplication 어노테이션을 제거한 뒤, 서블릿서버 팩토리로 톰캣 구현체를 생성했다.

서블릿 컨텍스트, 그러니까 컨테이너를 만들어준 뒤 addServlet으로 컨테이너에 서블릿을 등록하고, 등록한 서블릿에 매핑을 추가한다.

그렇다면 매핑마다 별도로 서블릿을 만들어서 요청 응답 코드를 전부다 만들면 개발은 끝난다고 생각할 수도 있다.하지만 서블릿에서 요청에 응답하는 코드는 매우 중복되어 있다. 다른 매핑에 다른 서블릿을 모두 만들면 중복된 코드를 제거할 수 없기 때문에 서블릿 앞단에서 공통된 응답처리를 하는 컨트롤러라는 오브젝트를 분리해낸다.

컨트롤러가 담당하는 이러한 공통된 처리에는 보안, 인증, 다국어 처리 등이 있다. 컨트롤러도 또한 서블릿이기 때문에 다음과 같은 형태를 띈다.

```kotlin
fun main(args: Array<String>) {
    val serverFactory = TomcatServletWebServerFactory()
    serverFactory.getWebServer( ServletContextInitializer{ servletContext:ServletContext? ->
        servletContext?.addServlet("frontController", object :HttpServlet(){
            override fun service(req: HttpServletRequest?, resp: HttpServletResponse?) {
                //상태코드, 헤더(컨텐트 타입 필수), 메세지 바디
                req?.also {
                    when(it.requestURI){
                        "/hello"->{
                            if(it.method==HttpMethod.GET.name()) {
                                val name = req?.getParameter("name")
                                resp?.also { resp ->
                                    resp.status = HttpStatus.OK.value()
                                    resp.addHeader(HttpHeaders.CONTENT_TYPE, MediaType.TEXT_PLAIN_VALUE)
                                    resp.writer.println("Hello $name")
                                }
                            }
                        }
                        "/user" ->resp?.status=HttpStatus.NOT_FOUND.value()
                        else->resp?.status=HttpStatus.NOT_FOUND.value()
                    }
                }

            }
        })?.addMapping("/*")
    }).start()
}
```

즉 모든 요청에 대해 매핑되고 (”/*”), 요청으로 부터 다른 서블릿을 매핑하는 역할을 해야 한다. 매핑이 모두 끝났다면 프론트 컨트롤러는 나머지 웹 애플리케이션 로직에 대해서는 다른 오브젝트에 위임을 하는 것이 적절한 방식이다.

매핑은 요청 경로와 메서드에 따라 적절한 컨트롤러를 호출해서 결과를 리턴하게 하는 것이다.

바인딩이란  웹요청/웹 응답 등 웹 기술과 관련된 코드를 배제한 평범한 자바타입으로 컨트롤러에게 웹 요청을 변환하여 전달하는 것이다. 요청 파라미터는 `String` 타입으로 넘겨주는 식이다. 웹 MVC는 복잡한 바인딩에 대한 기술을 다룬다.

여기까지 만든 것은 `standalone` 서블릿 어플리케이션이다. 즉, 컨테이너를 설치하거나 배포하는 과정없이 어플리케이션 코드로 실행되는 서블릿을 만들어보았다. 그렇다면 서블릿 컨테이너와 분리된 애플리케이션, 즉 스프링  `standalone` 애플리케이션을 만들어보자.

스프링 컨테이너는 서블릿을 직접 만들고 등록하는 서블릿 컨테이너와는 다르다. 스프링 컨테이너의 컴포넌트는 POJO(상속받지 않은 평범한 자바 오브젝트)이며, 이러한 컴포넌트에 대한 구성정보를 담고 있는 메타데이터 Configuration Metadat를 조합하여 사용가능한 시스템을 만들어낸다.

200 OK는 에러가 없는 경우 응답의 디폴트 상태코드가 된다.(따로 지정하지 않아도 된다.)

스프링 컨테이너의 표준 인터페이스는 ApplicationContext이다. 서블릿 컨테이너가 서블릿을 직접 생성해서 넣어주었다면, 스프링 컨테이너는 컴포넌트, 즉 Bean Object가 될 클래스 정보를 구성 정보 Configuration Metadata로써 등록하는 방법이 일반적이다.

구성정보를 등록했다면, 아무때나 스프링 빈을 사용하기 원할 때 ApplicationContext로 부터  빈의 이름, 또는 클래스 타입 등의 구성정보를 인자로 전달하여 빈을 가져와서 사용할 수 있다.이렇게 스프링 컨테이너는 POJO 오브젝트를 등록하고 관리해준다.

정확히 말하면, 빈의 구성정보를 전달해두면, 컨테이너가 초기화 될 때 해당 빈 오브젝트를 컨테이너가 생성해둔다.(여기서 싱글톤 개념이 적용 된다.) getBean은 이미 생성된 인스턴스를 전달받는 것이다. 스프링 컨테이너를 싱글톤 레지스트리라고 한다. 이는 싱글톤 디자인 패턴을 사용하지 않아도 초기화시 인스턴스를 한번만 생성해두고 이를 재사용하는 특성을 말하는 용어이다.

### DI

어떤 클래스가 의존하는 구현체를 바꿔야 할 때 우리는 어떤 방법을 사용해야 할까?

클래스는 인터페이스에만 의존하게 하고 인터페이스를 구현한 구현체들을 만드는 것이 좋은 방법이다. 인터페이스에 의존하면 구현체가 아무리 바뀌어도 코드를 변경할 필요가 없기 때문이다. 그러나, 이 방법을 사용하더라도 런타임에 어떤 구현 클래스를 사용할 것인지 결정이 되어야 한다.

따라서 어떻게든 소스코드와 소스코드가 실제로 런타임에 사용할 구현체와의 연결고리를 만들어줘야 하며 이를 `DI` 라고 부른다. `DI` 에는 제 3의 존재가 필요하며 이를 어셈블러라고 부른다. 어셈블러는 런타임에 필요한 구현체를 만들어서 주입을 해준다. `코드에서 직접 인스턴스를 생성하여 주입하지 않는다.`  스프링 컨테이너는 어셈블러이다.

스프링 컨테이너는 런타임에 싱글톤 오브젝트를 만들고, 해당 오브젝트가 의존하는 클래스를 주입해주고, 해당 오브젝트에 의존하는 다른 빈 오브젝트에 주입해준다.

주입을 해준다는 건, 여러 방법이 있지만, 구현체의 참조를 해당 구현체에 의존하는 오브젝트의 생성자 파라미터로 넘겨준다던가, 혹은 Setter의 파라미터로 넘겨주는 등등의 작업을 말한다.

그런데 `어떤 소스코드에 주입을 해줘야 한다는 사실`을 스프링 컨테이너가 어떻게 아는걸까? 초창기에 스프링 컨테이너는 xml을 이용해서 빈 구성정보를 등록하고 소스코드의 생성자에 어떤 빈을 주입할지를 전부 기술했다.모든 걸 명시적으로 정의해야했던 셈이다. 요즘에는 많은 부분을 생략하고 몇가지 룰을 사용해서 주입을 자동화한다. 소스코드의 생성자 파라미터에  인터페이스 타입이 있는 것을 확인하면 등록정보를 전부 검색하여 구현체를 탐색하고 자동으로 주입해준다.

그렇다면 다른 빈에게 의존받는 빈 오브젝트를 먼저 등록해줘야 하는 순서의 문제는 없는걸까? 어떤 빈이 다른 빈에 생성자 파라미터에서 의존하고 있는데 탐색결과 해당 빈이 아직 등록되지 않았다면 주입이 안되지 않을까라는 궁금점이 생긴다. 이 부분 또한 스프링 컨테이너가 의존성의 순서에 맞게 알아서 빈을 주입해준다. 사용자는 순서에 관계없이 등록만 하면 스프링 컨테이너가 의존성에 맞게 빈을 순서대로 주입해준다.

컨테이너리스 지향이라는 목표를 살펴보면, 현재 직접 컨테이너를 생성하고 서블릿을 등록하고 있는 상태이므로 갈길이 멀다. 가장 시급한 건,  서블릿 코드 안에서 웹 요청으로부터 스프링 컨테이너에서 어떤 빈 컴포넌트를 호출할지를 결정하는 `맵핑` 이 이루어지고 있고, 웹 요청에서 비즈니스 로직에 해당하는 쿼리 파라미터 등의 `바인딩` 작업 또한 하드코딩되어 있다. 컨테이너리스를 위해서는 서블릿으로부터 `맵핑` 과 `바인딩` 을 제거해야 한다.

이를 위해서는 스프링 컨테이너에 맵핑과 바인딩을 위임하는 `DispatcherServlet` 을 사용한다. DispatcherServlet은 서블릿 구현체로  `WebApplicationContext` 인터페이스 타입을 생성자 파라미터로 받고, 바로 이 스프링 컨테이너에게 맵핑과 바인딩을 위임한다. `Dispatch` 란 위임이란 뜻이다.

코드가 다음과 같이 줄어든다

```kotlin
fun main(args: Array<String>) {
    val serverFactory = TomcatServletWebServerFactory()
    serverFactory.getWebServer( ServletContextInitializer{ servletContext:ServletContext? ->
        val applicationContext= GenericWebApplicationContext().also{

            it.registerBean{SimpleHelloService()}
            it.registerBean<HelloController>()
            it.refresh()
        }
        servletContext?.addServlet("dispatcherServlet", DispatcherServlet(applicationContext))?.addMapping("/*")
    }).start()
}
```

이 코드를 통해 알 수 있는 것은 DispatcherServlet은 맵핑과 바인딩을 시도할 때 스프링 컨테이너에서 필요한 빈 오브젝트를 불러와서 작업을 위임한다는 것이다. 그러나 이 애플리케이션을 실행한 뒤 기존의 경로인 /hello?name=이름 으로 요청을 보내면`404 not found`가 응답으로 돌아온다.

와일드카드로 서블릿을 매핑해줬으니 요청이 전달됐을텐데 왜일까? 당연하게도, 해당 경로와 어떤 컨트롤러를 매핑할지 어떤 정보도 제공하지 않았기 때문이다. 즉 스프링 컨테이너의 컨트롤러 빈과 요청 경로와의 매핑이 존재해야 한다.

이러한 매핑은 과거 스프링에서는 xml로 수행했다. 그 이후에도 여러 관례나 방법들이 있었지만 가장 각광받고 많이 쓰이는 방법은 컨트롤러 코드에 매핑정보를 코드로 작성하는 방식이다.

디스패처 서블렛은 어플리케이션 컨텍스트에 요청된 경로를 전달한다. 어플리케이션 컨텍스트는 빈을 검색하여, `GetMapping` 이나 `RequestMapping` 등의 매핑 정보 어노테이션을 가진 클래스를 찾는다. 그 뒤 매핑정보를 추출하여 매핑 테이블을 만든다.

그러나 메소드 레벨의 매핑 어노테이션으로는 빈을 찾아낼 수 없다. 빈 탐색의 단위는 클래스이기 때문에, 매핑 어노테이션 또한 클래스 레벨에 존재해야 한다.

```kotlin
@RequestMapping("/hello")
class HelloController(val helloService: HelloService) {
    @GetMapping
    fun hello( @RequestParam("name") name:String?)=helloService.sayHello(requireNotNull(name))
}
```

그러나 이 컨트롤러로도 /hello?name=이름 경로의 GET 요청은 404 NOT FOUND 응답을 돌려준다. 왜일까? 디스패처 서블렛은  디폴트로 컨트롤러에서 String 타입을 반환할 때 이를 해당하는 뷰 템플릿으로 응답하기 때문이다. 우리가 원하는건 뷰 템플릿이 아니라, String 문자열이 응답의 Message Body에 들어가길 원한다. 이를 위해서는 @ResponseBody 어노테이션을 추가해야 한다.

그러나 이 코드도 부트 3.0과 스프링 6.0에서는 변경되었다. 컨트롤러가 서블릿의 매핑테이블에 등록되기 위해서는 @Controller 어노테이션이 있어야 한다.

현재 코드는 스프링 컨테이너와 서블릿 컨테이너의 생성 및 서블릿, 빈 등록이 분리되어 있다. 이를 스프링 컨테이너가 초기화 되면서 한번에 일어나도록 수정할 수 있다.

스프링 컨테이너의 인스턴스인 `ApplicationContext` 에 빈 오브젝트의 의존성에 대한 정보를 어떻게 제공해줄까? 최근에 각광받는 방법은 팩토리 메서드를 (어노테이션)구성 정보로 등록해서 리턴하는 인스턴스를 빈으로 등록하는 방법이다. 빈 오브젝트를 만드는 방법을 알려주는 셈이다. 이방법이 각광받는 이유는 인스턴스를 생성/초기화하는 복잡한 방법을 자바코드로 간결하게 기술할 수 있기 때문이다.

어노테이션 구성정보를 사용하기 위해서는 컨테이너 구현체를 기존의 `GenericWebApplicationContext` 대신에 `AnnotationConfigWebApplicationContext`

로 변경하고, 빈을 등록하는 대신 빈이 존재하는 @Configuration 어노테이션이 있는 클래스를 등록한다.

```kotlin
@Configuration
class TobyspringApplication{
    @Bean
    fun helloController(helloService: HelloService) = HelloController(helloService)
    @Bean
    fun helloService():HelloService = HelloServiceImpl()
}

fun main(args: Array<String>) {

    object: AnnotationConfigWebApplicationContext(){
        override fun setClassLoader(classLoader: ClassLoader) {
            this.classLoader = classLoader
        }

        override fun onRefresh() {
            super.onRefresh()
            TomcatServletWebServerFactory().getWebServer(ServletContextInitializer { servletContext ->
                servletContext.addServlet("dispatcherServlet", DispatcherServlet(this)).addMapping("/*")
            }).start()
        }
    }.also { it.register(TobyspringApplication::class.java) }
}
```

@Configuration 어노테이션이 있는 클래스는 빈 팩토리 메서드 이상으로 더 많은 구성정보를 담고 있다.

빈을 등록하기 위해서 팩토리 메서드를 사용할 수도 있지만, 클래스 자체에 어노테이션을 적용할 수도 있다. 이 때 사용되는 어노테이션이 바로 `@Component` 이다. Configuration 클래스가 컴포넌트 어노테이션이 붙은 클래스를 찾아서 등록하지 위해서는 @ComponentScan 어노테이션이 있어야 한다.

@Component는 메타 어노테이션(어노테이션에 적용되는 어노테이션)이기도 하기 때문에 @Component 어노테이션을 가지는 @Controller, @Service 등을 사용하거나 커스텀 어노테이션을 사용해도 컴포넌트 스캔의 대상이 된다.

@Component 메타 어노테이션을 이용해서 레이어 아키텍쳐로 빈 오브젝트를 구분하는 역할을 하기 위해 사용하는 것이 스프링이 제공하는 @Service, @Controller 등의 레이어를 나타내는 어노테이션들이며, 이들을 `스테레오타입 어노테이션` 이라고 부른다.

클래스 레벨의 @Controller는 디스패처 서블릿이 @RequestMapping이 붙어있지 않아도 매핑 정보를 등록할 빈으로 인식한다.

현재는 서블릿 컨테이너와 서블릿의 인스턴스를 직접 생성해주고 있다. 하지만 이 인스턴스 또한 빈 오브젝트로 스프링 IoC 컨테이너가 관리하도록 등록할 수 있다. 기존과 동일하게 빈 팩토리 메서드를 Configuration 어노테이션이 있는 클래스에 등록하고, 어플리케이션 컨텍스트로 부터 가져오면 된다.

```kotlin
@Configuration
@ComponentScan
class TobyspringApplication{
    @Bean
    fun servletWebServerFactory():ServletWebServerFactory=TomcatServletWebServerFactory()
    @Bean
    fun dispatcherServlet():DispatcherServlet=DispatcherServlet()
}

fun main(args: Array<String>) {

    object: AnnotationConfigWebApplicationContext( ){
        @Suppress("INAPPLICABLE_JVM_NAME")
        @JvmName("myClassLoader")
        override fun setClassLoader(classLoader: ClassLoader) {
            this.classLoader=classLoader
        }
        override fun onRefresh() {
            val servletFactory = getBean(ServletWebServerFactory::class.java)
            val dispatcherServlet = getBean(DispatcherServlet::class.java)
            dispatcherServlet.setApplicationContext(this) //필요없음
            super.onRefresh()
            servletFactory.getWebServer(ServletContextInitializer { servletContext ->
                servletContext.addServlet("dispatcherServlet", dispatcherServlet).addMapping("/*")
            }).start()
        
    }.also { it.register(TobyspringApplication::class.java);it.refresh()}
}
```

dispatcherServlet이 사용할 스프링 컨테이너(어플리케이션컨텍스트)를 생성자로 지정해주었던 것을 setter로 지정하는 정도의 변경사항이다. 심지어 별도로 지정하지 않아도 된다.

이는 스프링 컨테이너가 applicationContext를 자동으로 주입해줬기 때문이다. 빈을 자동으로 주입해주는 메커니즘은 빈의 라이프사이클에 메소드에 따라 실행된다. DisPatcherServlet은 ApplicationContextAware라는 인터페이스를 구현하고 있고, 해당 인터페이스는 스프링 IoC 컨테이너가 관리하는 빈 오브젝트를 등록된 빈에 주입해주는 라이프사이클 메서드인 `setApplicationContext` 를 가진다. 이 인터페이스를 구현한 클래스는 스프링 컨테이너에 등록되면 ApplicationContext를 컨테이너가 주입해준다. `ApplicationContext`외에도 여타 다른 Aware 인터페이스들 또한 자동주입을 위한 라이프사이클 메서드들을 가지고 있다.

컨테이너 입장에서 ApplicationContext는 자기 자신이다. 자기 자신을 빈으로 취급하여 관리한다고 볼 수 있다.

Aware 컨텍스는 어노테이션이 등장하기도 전부터  써왔던 매우 오래된 인터페이스로, 최근에는 생성자와 final 멤버(코틀린은 생성자 val 파라미터)로 자동으로 주입받을 수 있다.

```kotlin
class SpringApplication {
    companion object Context : AnnotationConfigWebApplicationContext() {
        @Suppress("INAPPLICABLE_JVM_NAME")
        @JvmName("myClassLoader")
        override fun setClassLoader(classLoader: ClassLoader) {
            this.classLoader = classLoader
        }

        override fun onRefresh() {
            val servletFactory = getBean(ServletWebServerFactory::class.java)
            val dispatcherServlet = getBean(DispatcherServlet::class.java)
            dispatcherServlet.setApplicationContext(this)
            super.onRefresh()
            servletFactory.getWebServer(ServletContextInitializer { servletContext ->
                servletContext.addServlet("dispatcherServlet", dispatcherServlet).addMapping("/*")
            }).start()
        }
        inline fun <reified T:Any>  runApplication(vararg args:String) {
           register(T::class.java)
            refresh() 
        }
    }
}

```

```kotlin
fun main(args: Array<String>) {

    SpringApplication.runApplication<TobyspringApplication>(*args)
}
```

코드를 사용해서 테스트 하는 것의 장점은 뭘까?

api의 기능을 테스트한다고 하면 컨트롤러와 서비스의 코드가 모두 정상적으로 동작해야 테스트가 통과할 것이다. 이는 통합테스트에 가까우며, 보다 간편하게는 코드를 직접적으로 테스트할 수 있다. 즉 서버를 실행하지 않고 평범한 자바 오브젝트 코드를 직접 테스트하는 방법이다. 이를 단위 테스트라고 한다. 물론  데이터베이스 트랜잭션을 검증한다고 할 때는 불가능한 방법이다.

단위 테스트의 장점은 매우 간결하다는 것과 고립된 테스트가 가능하다는 것이다.

그런데, 컨트롤러가 어떤 서비스를 주입받아 메서드에서 사용한다고 해보자. 이 메서드를 테스트하기 위해선 어떻게 해야할까? 컨트롤러에 대한 단위테스트, 고립된 테스트를 하는 것이 우리의 문제지만 서비스에 대한 의존성을 가진다. 의존성을 고립시키기 위해서는 `DI` 가 필요하다.

스프링 컨테이너가 관리하지 않는 테스트 환경해서 DI를 하기 위해서는 적절한 의존성을 직접 만들 수도 있다. 테스트 용으로 매우 심플하게 구현된 의존성을 `stub` 이라고 한다.

컨테이너가 빈 오브젝트가 의존하고 있는 빈을 주입해줄 때, 주입해줄 빈을 탐색한 결과가 한개라면 아무 문제업이 해당 빈을 주입해준다. 단 한개만 있을 때 주입해주는 이런 방식을 `Autowiring` 이라고 한다.  그런데 데코레이터 패턴처럼 동일한 타입을 가지면서 권한이나 책임을 다른 구현체에 위임하는 방식을 사용하는 구현체가 있어서, 빈이 2개 이상 등록되어 있다면 어떨까?

그대로 스프링 컨테이너를 실행하면 에러가 나게 되어 있다. 컨트롤러는 해당 타입에 대해 하나의 빈을 주입받으려고 하는데, 빈이 2개가 있으니 결정하지 못하는 것은 당연하다.

물론 xml로 설정을 명시적으로 작성할 수도 있을 것이다. 혹은 @Component 메타 어노테이션 등록을 하지 않고 빈 팩토리 메서드를 @Configuration 클래스에 등록하여 동일 타입 구현체간의 의존성 설정을 해줄 수도 있다.

또다른 방법은 오토와이어링 될 우선순위를 지정하는 `@Primary` 어노테이션을 사용하는 것이다.

또다른 유사한 구조로는 `Proxy 패턴` 이 있다. 프록시는 실제 오브젝트를 대리해서 네트워크 프록시처럼 부가저인 효과를 주거나, 실제 오브젝트에 대한 lazy loading을 구현하기 위해 사용한다. 혹인 서비스가 다른 api에 의존하여 네트워크 호출이 발생할 때 proxy로 이를 대리하기도 한다.

```toc
```