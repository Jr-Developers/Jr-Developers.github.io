---
layout: post
title:  "여러 Spring Boot 서버 H2 접속 설정"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot spring h2
---

여러 `Spring Boot` 서버에서 `H2` 서버에 접속해야 할 경우 다음과 같이 설정한다.<br>
우선 하나의 서버에서 다음과 같은 <code class="inline">@Configuration</code> 설정을 추가하며 해당 서버를 <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버라고 지칭하겠다.
{% highlight java %}
@Configuration
public class H2Config {
    @Bean(initMethod = "start", destroyMethod = "stop")
    public Server inMemoryH2DatabaseServer() throws SQLException {
        return Server.createTcpServer(
                "-tcp", "-tcpAllowOthers", "-tcpPort", "9091");
    }
}
{% endhighlight %}

<code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버의 `H2`에 다른 `Spring Boot` 서버도 접근 가능하다록 TCP 포트를 개방하는 설정이다.<br>
다음은 <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버 `application.yml` 설정으로 초기 테이블 생성과 데이터 생성을 위한 설정을 하였다.
- application.yml - <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버 
{% highlight yml %}
...
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa: #초기 데이터 입력을 위한 설정 - Hello.java, resources/data.sql
    generate-ddl: true
    hibernate:
      ddl-auto: create 
{% endhighlight %}
<code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버 프로젝트의 자세한 소스는 [여기](https://github.com/Jr-Developers/h2server1)를 확인하며
<code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버가 아닌 다른 접근 하는 서버를 <code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버라고 지칭하겠다.<br>
<code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버에서는 단순히 <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버의 `H2` 서버의 데이터를 조회 할 것이다.
다음은 <code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버의 `application.yml`이다.
- application.yml - <code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버 
{% highlight yml %}
...
spring:
  datasource: # A서버에서 기동 및 포트를 개방한 H2
    url: jdbc:h2:tcp://localhost:9091/mem:testdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa: # A 서버에서 테이블 생성 및 데이터를 입력 
    generate-ddl: true
    hibernate:
      ddl-auto: none
{% endhighlight %}

<code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버 프로젝트의 자세한 소스는 [여기](https://github.com/Jr-Developers/h2server2)를 확인하며
<code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버에서 기동 중이고 포트를 개방한 `H2` 로 접속한다.

다음은 두 서버의 `Spring Boot` 메인으로 테스트를 위한 간단한 조회 결과를 응답한다.
- SpringBootMain
{% highlight java %}
@SpringBootApplication
@RestController
public class H2server1Application {

    @Autowired
    HelloRepository helloRepository;

    public static void main(String[] args) {
        SpringApplication.run(H2server1Application.class, args);
    }

    @GetMapping
    public List<Hello> test(){
        return helloRepository.findAll();
    }
}
{% endhighlight %}
- 확인 결과
<p align="center" width="100%">
    <a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/2/1.PNG" data-lightbox="1" data-title="1">
      <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/2/1.PNG">
    </a>
</p>
<code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버에서 `H2`가 기동되므로 <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버 기동 이후 <code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버를 기동하여 두 서버 모두 같은 조회 결과를 응답하는 것을 확인하였다.

- <code class="inline text">A</code>{:style="color: red; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/h2server1)
- <code class="inline text">B</code>{:style="color: blue; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/h2server2)

