---
layout: post
title:  "Spring Boot Log4j2 설정"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot log4j2 spring-cloud-sleuth
---

`Spring Boot`에서는 기본적으로 [`Logback`](http://logback.qos.ch/)를 사용하기 때문에 `Logback`의 `Dependency`를 제외시키고 `Log4j2`를 추가 해야 한다. 자세한 내용은 설정 내용은 `Sprinb Boot` [Reference Doc](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-configure-log4j-for-logging)를 참고한다.
{% highlight xml %}
...
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
...
{% endhighlight %}

`Log4j2`는 로깅 포멧을 지원하는 파일 형식이 `xml`, `yml`, `json` 등이 있지만 기본적으로 지원하는 파일 형식은 `xml`이며 다른 파일 형식일 경우 추가적인 [`Dependency`](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-configure-log4j-for-logging-yaml-or-json-config)를 추가 해야 한다.

--------------------

`Spring Boot`의 기본 `Lof4j2`의 설정은 [log4j2.xml](https://github.com/spring-projects/spring-boot/blob/v2.4.1/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/log4j2/log4j2.xml)를 참고 하며 자세한 설정은 `Log4j2` [문서](http://logging.apache.org/log4j/2.x/index.html)를 참고한다.<br>
- [Appenders](http://logging.apache.org/log4j/2.x/manual/appenders.html)
    - [Console Appender](http://logging.apache.org/log4j/2.x/manual/appenders.html#ConsoleAppender), [Kafka Appender](http://logging.apache.org/log4j/2.x/manual/appenders.html#KafkaAppender), [RollingFile Appender](http://logging.apache.org/log4j/2.x/manual/appenders.html#RollingFileAppender)
- [Pattern Layout](http://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout)
- [Mixing Synchronous and Asynchronous Loggers](http://logging.apache.org/log4j/2.x/manual/async.html#MixedSync-Async)

------------------------------

다음은 `Log4j2` 테스트를 위한 프로젝트는 <code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버와 <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버를 준비하였다.<br>
두 서버 모든 같은 `log4j2.xml`를 사용하며 <code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버는 <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버를 호출 하고 <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버는 `application.yml`설정을 통해 환경별로 로깅 레벨을 설정하여 내용을 비교할 것이다.<br>
추가적으로 `Spring Cloud Sleuth`를 통해 로깅에 `traceId`와 `spandId`를 출력하여 서버간 호출도 확인해 볼 것이다.<br>
간단한 테스트를 위해 같은 `log4j2.xml`을 사용하였지만 `log4j2.xml` 또한 환경별로 파일을 분리하여 설정 가능하다.

- `log4j2.xml` - 공통
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="INFO">
    <Properties>
        <Property name="layoutPattern">
            {% raw %}%highlight{[%-5p]}{FATAL=bg_red, ERROR=red, INFO=green, DEBUG=cyan} %style{%mdc{traceId}}{Red}|%style{%mdc{spanId}}{blue}|%style{%d{yyyy-MM-dd HH:mm:ss:SSS}}{cyan} [%50c{1.}] %style{[%t]}{yellow} - %m%n {% endraw %}
        </Property>
    </Properties>
    <Appenders>
        <Console name="Console_Appender" target="SYSTEM_OUT">
            <PatternLayout disableAnsi="false" pattern="${layoutPattern}"/>
        </Console>
        <RollingFile name="File_Appender" fileName="logs\app.log" filePattern="logs\%d{yyyy-MM-dd}_%i.log.zip">
            <PatternLayout pattern="${layoutPattern}"/>
            <Policies>
                <SizeBasedTriggeringPolicy size="100MB"/>
                <TimeBasedTriggeringPolicy interval="1"/>
            </Policies>
            <DefaultRolloverStrategy max="10" fileIndex="min"/>
        </RollingFile>
        <Async name="Async_Console" includeLocation="true">
            <AppenderRef ref="Console_Appender"/>
        </Async>
        <Async name="Async_File" includeLocation="true">
            <AppenderRef ref="File_Appender"/>
        </Async>
    </Appenders>
    <Loggers>
        <Root level="info" additivity="false">
            <AppenderRef ref="Async_Console"/>
            <AppenderRef ref="File_Appender"/>
        </Root>
        <Logger name="org.hibernate.SQL" level="debug" additivity="false">
            <AppenderRef ref="Async_Console"/>
        </Logger>
        <Logger name="org.hibernate.type" level="trace" additivity="false">
            <AppenderRef ref="Async_Console"/>
        </Logger>
    </Loggers>
</Configuration>
{% endhighlight %}

- `application.yml` - <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버
{% highlight yml %}
logging:
  config: ${logging.config}
  level:
    root: ${logging.level.root}
    org.hibernate.SQL: ${logging.level.org.hibernate.SQL}
    org.hibernate.type: ${logging.level.org.hibernate.type}
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa:
    generate-ddl: true
    hibernate:
      ddl-auto: create
{% endhighlight %}
- `application-dev.yml` - <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버
{% highlight yml %}
logging:
  config: classpath:log4j2.xml
  level:
    root: info
    org.hibernate.SQL: debug # debug - SQL 로그 출력
    org.hibernate.type: trace #mtrace - binding 변수 출력 
{% endhighlight %}
- `application-prod.yml` - <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버
{% highlight yml %}
logging:
  config: classpath:log4j2.xml
  level:
    root: info
    org.hibernate.SQL: info
    org.hibernate.type: info
{% endhighlight %}

---------------------------

<code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버에서는 <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버를 호출하기 위해 `RestTemplate`를 사용하였으며 `RestTemplate`를 `new`를 통해서 생성시 `Spring Cloud Sleuth`의 `TracingClientHttpRequestInterceptor`가 등록되지 못하는 것으로 [확인](https://docs.spring.io/spring-cloud-sleuth/docs/2.2.5.RELEASE/reference/html/#synchronous-rest-template)된다.
>You have to register RestTemplate as a bean so that the interceptors get injected. If you create a RestTemplate instance with a new keyword, the instrumentation does NOT work.

- TraceRestTemplateBeanPostProcessor
{% highlight java %}
public class TraceRestTemplateBeanPostProcessor implements BeanPostProcessor {
    private final BeanFactory beanFactory;

    public TraceRestTemplateBeanPostProcessor(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof RestTemplate) {
            RestTemplate rt = (RestTemplate)bean;
            (new RestTemplateInterceptorInjector(this.interceptor())).inject(rt);
        }

        return bean;
    }

    private LazyTraceClientHttpRequestInterceptor interceptor() {
        return new LazyTraceClientHttpRequestInterceptor(this.beanFactory);
    }
}
{% endhighlight %}

- LazyTraceClientHttpRequestInterceptor
{% highlight java %}
public class LazyTraceClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
    private final BeanFactory beanFactory;
    private TracingClientHttpRequestInterceptor interceptor;

    public LazyTraceClientHttpRequestInterceptor(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        return this.isContextUnusable() ? execution.execute(request, body) : this.interceptor().intercept(request, body, execution);
    }

    boolean isContextUnusable() {
        return ContextUtil.isContextUnusable(this.beanFactory);
    }

    ClientHttpRequestInterceptor interceptor() {
        if (this.interceptor == null) {
            this.interceptor = (TracingClientHttpRequestInterceptor)this.beanFactory.getBean(TracingClientHttpRequestInterceptor.class);
        }

        return this.interceptor;
    }
}
{% endhighlight %}

- TracingClientHttpRequestInterceptor
{% highlight java %}
public final class TracingClientHttpRequestInterceptor implements ClientHttpRequestInterceptor {
...
    public ClientHttpResponse intercept(HttpRequest req, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        TracingClientHttpRequestInterceptor.HttpRequestWrapper request = new TracingClientHttpRequestInterceptor.HttpRequestWrapper(req);
        Span span = this.handler.handleSend(request);
        if (log.isDebugEnabled()) {
            log.debug("Wrapping an outbound http call with span [" + span + "]");
        }

        ClientHttpResponse response = null;
        Throwable error = null;

        Object var10;
        try {
            Scope ws = this.currentTraceContext.newScope(span.context());
            Throwable var9 = null;

            try {
                response = execution.execute(req, body);
...
{% endhighlight %}

----------------------------
- <code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버 - Main
{% highlight java %}
@SpringBootApplication(scanBasePackages = {"brave", "io.github.jr"})
@RestController
@Log4j2 // private static final Logger log = LogManager.getLogger(StartserverApplication.class); 생성
public class StartserverApplication {
    @Autowired
    RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }


    public static void main(String[] args) {
        SpringApplication.run(StartserverApplication.class, args);
    }

    @GetMapping
    public Object retrieveByClubName() throws JsonProcessingException {
        log.info("[startserver] start");
        Map<String, Object> map = new HashMap<>();
        map.put("userId", "Hello");
        map.put("name", "World");
        Object object = restTemplate.postForObject("http://localhost:8080/user", map, Object.class);
        log.info("[startserver] end : " + new ObjectMapper().writeValueAsString(object));
        return object;
    }
}
{% endhighlight %}

------------------------------------

- <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버 - Main
{% highlight java %}
@SpringBootApplication(scanBasePackages = {"brave","io.github.jr"})
@RestController
@Log4j2 // private static final Logger log = LogManager.getLogger(LoggingApplication.class); 생성
public class LoggingApplication {
    @Autowired
    private UserRepository userRepository;

    public static void main(String[] args) {
        SpringApplication.run(LoggingApplication.class, args);
    }

    @PostMapping("/user")
    public User insert(@RequestBody User user) throws JsonProcessingException {
        log.info("[logging] start : " + new ObjectMapper().writeValueAsString(user));
        User user1 = userRepository.save(user);
        log.info("[logging] end : " + new ObjectMapper().writeValueAsString(user1));
        return user1;
    }
}
{% endhighlight %}

------------------------------------

- <code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버 
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/1.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/1.PNG">
</a>
- <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버 - `spring.profiles.active: dev`
    : Hibernate SQL 및 Binding Parameter 출력
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/2.JPG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/2.JPG">
</a>
- <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버 - `spring.profiles.active: prod`
    : Hibernate SQL 및 Binding Parameter 미출력
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/3.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/5/3.PNG">
</a>

--------------------------------------

- <code class="inline text">startserver</code>{:style="color: blue; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/startserver)
- <code class="inline text">logging</code>{:style="color: red; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/logging)
