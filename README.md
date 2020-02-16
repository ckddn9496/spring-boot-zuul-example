## Spring boot - Routing and Filtering 예제
Spring 가이드 Routing and Filtering을 공부하며 만든 프로젝트입니다.

### Routing and Filtering
이번 예제에서는 MSA환경에서 넷플릭스 줄(Netflix Zuul)라이브러리를 사용하여 routing과 request filtering을 적용해본다.
Netflix Zuul을 이용하여 쉽게 Spring환경에서 Reverse Proxy Application을 제작하고 사용할 수 있다.

### Reverse Proxy
클라이언트를 대신해서 한 대 이상의 서버로부터 자원을 추출하는 프록시 서버이다. 
클라이언트로의 요청을 웹서버로 전달되는 도중의 처리에 끼어들어서 다양한 전후처리를 시행할 수가 있다.

### Netflix Zuul
넷플릭스에서 제공하는 Zuul은 API Gateway이자 Reverse Proxy이다.
개발자들은 Zuul을 이용하여 실제 요청에 대하여 로드 밸런싱, 인증, 모니터링 등을 비즈니스 로직과 분리할 수 있으며,
모든 서비스에 공통으로 들어갈 기능에 대해 중복적으로 개발하야만 하는 문제를 해결할 수 있다.

- 참고자료
    * [예제 프로젝트](https://spring.io/guides/gs/routing-and-filtering/)
    * [Zuul 깃허브](https://github.com/Netflix/zuul)
    * [배민 API Gateway](https://woowabros.github.io/r&d/2017/06/13/apigateway.html)
    * [참조 블로그](https://jsonobject.tistory.com/464)
***

### <h1>프로젝트 구조
프록시 서버를 만들고 동작시기키 위해 최소 2개의 서버가 필요하다.
실제 서비스가 구현될 서버 `book`과 프록시역할을 할 `gateway`를 만든다.

### Book Application
RestController를 만들어 `/available`과 `/checked-out`로 접근하는 request에 대해 문자열을 리턴해주도록 제작하였다. 
```java
@RestController
@SpringBootApplication
public class RoutingAndFilteringBookApplication {

  @RequestMapping(value = "/available")
  public String available() {
    return "Spring in Action";
  }

  @RequestMapping(value = "/checked-out")
  public String checkedOut() {
    return "Spring Boot in Action";
  }

  public static void main(String[] args) {
    SpringApplication.run(RoutingAndFilteringBookApplication.class, args);
  }
}
```

두개의 서버를 구동해주어야 하므로 포트번호도 `8090`으로 지정해주었다.
```properties
# resourecs/application.properties
spring.application.name=book
server.port=8090
```

***

### Gateway Application
`Book` 애플리케이션에 대해 리버스 프록시의 역할을 할 것이다. Zuul을 사용하기 위해 설정으로 `@EnableZuulProxy`어노테이션을 작성하고 적용할 필터를 빈으로 등록한다.

```java
@EnableZuulProxy
@SpringBootApplication
public class RoutingAndFilteringGatewayApplication {

  public static void main(String[] args) {
    SpringApplication.run(RoutingAndFilteringGatewayApplication.class, args);
  }

  @Bean
  public SimpleFilter simpleFilter() {
    return new SimpleFilter();
  }

}
```

Gateway application으로 접근하는 request를 처리하기 위해 Zuul에게 어느 서비스로 프록시할지를 알려주어야 한다.
properties에 `zuul.routes.{name}.url={url}`형식으로 라우팅할 곳의 이름과 url을 지정하면
이후 `/name` 으로 들어오는 request에 대해 `url`로 프록시를 적용한다.
```properties
# resourecs/application.properties
zuul.routes.books.url=http://localhost:8090
ribbon.eureka.enabled=false
server.port=8080
```
위의 설정은 Zuul이 `/books` request에 대해 `http://localhost:8090`으로 프록시 하도록 한다.
`ribbon.eureka.enabled=false`는 Zuul이 default으로 Client-side 로드밸런싱을위해 Eureka를 사용하는데, 별도의 설정과 구축이 필요하므로 false로 지정한다. 

이제 필터를 적용해보자. ZuulFilter를 상속하여 쉽게 필터를 만들 수 있다.
```java
public class SimpleFilter extends ZuulFilter {

  private static Logger log = LoggerFactory.getLogger(SimpleFilter.class);

  @Override
  public String filterType() {
    return "pre";
  }

  @Override
  public int filterOrder() {
    return 1;
  }

  @Override
  public boolean shouldFilter() {
    return true;
  }

  @Override
  public Object run() {
    RequestContext ctx = RequestContext.getCurrentContext();
    HttpServletRequest request = ctx.getRequest();

    log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));

    return null;
  }

}
```

filterType으로는 `pre`, `route`, `post`, `error`가 있으며 이번 예제에서는 `pre`를 사용하였다.
각 필터에 대한 설명은 아래와 같다.

* `pre`: 라우팅 전
* `route`': 라우팅 중
* `post`: 라우팅 후
* `error`: request처리 중 에러가 발생 했을 때

filterOrder는 같은 필터들에 대해 우선순의를 설정한다.

shouldFilter는 해당 필터의 적용 여부를 설정하도록한다.

run은 해당 필터가 수행할 기능을 기술하도록 한다.

Zuul Filter는 리퀘스트와 상태 정보를 저장하여 `RequestContext`에 담고있으며 이것을 `HttpServletRequest`로 받아 `run`에서 사용할 수 있다.

***

### Test

두 프로그램을 실행하자.
`book`애플리케이션에 등록한 `localhost:8090/available`과 `localhost:8090/checked-out`으로 접속하면
 문자열이 잘 리턴되는 것을 확인할 수 있다.

Zuul Filter를 이용한 리버스 프록시가 잘 등록되었는지 확인해 보자. properties에서 `books`라는 이름으로 프록시를 등록하였고,
`localhost:8080/books/available`로 접근하면 프록시가 작동한다.
run에서 작성한 코드가 실행 되며 해당 request에 대해 메서드와 url를 콘솔에서 확인 할 수 있다.

```
2020-02-16 14:32:14.341  INFO 24560 --- [nio-8080-exec-1] c.e.r.filters.pre.SimpleFilter           : GET request to http://localhost:8080/books/available
```

화면에는 `localhost:8090/available`에서 보여진 결과와 같은 결과가 나오는 것을 확인 할 수 있다. 하지만 브라우저의 url은 `localhost:8080/books/available`로 보여진다.

### 정리
클라이언트 요청은 많은 트래픽과 다양한 형태(예상하지 못한 형태)의 요청으로 경고없이 운영에 이슈를 발생시킨다.
zuul은 이런한 문제를 신속하고, 동적으로 해결하기 위해서 다양한 형태의 Filter를 실행한다.
Filter에 기능을 정의하고, 이슈사항에 발생시 적절한 filter을 추가함으로써 이슈사항을 간단하게 대비할 수 있다.
중복의 제거로 인한 관리 안정성을 확보시켜주며, 서로를 직접 노출 없이 격리함으로서 보안 안정성을 높일 수 있다는 점이 있다.


![image](https://woowabros.github.io/img/2017-06-06/Request-Lifecycle.png)