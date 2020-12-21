---
layout: post
title:  "Spring Initializr를 사용한 Spring Boot 프로젝트 생성"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot spring spring-initializr
---

`Spring`에서는 `Spring Boot` 프로젝트의 생성을 위해 `Dependency`, <span class="tooltip" id="build">Build Tool</span>, <span class="tooltip" id="lang">언어</span>등을 쉽게 선택하여 프로젝트를 생성할 수 있는 [사이트](https://start.spring.io/)를 제공한다.
<p align="center" width="100%">
    <a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/1.PNG" data-lightbox="1" data-title="1">
      <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/1.PNG">
    </a>
</p>
또한 `Intellij`에서도 `File > New > Project...`에서도 `Spring Initializr`를 통해 생성이 가능하다.
<p align="center" width="100%">
    <a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/2.PNG" data-lightbox="1" data-title="1">
      <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/2.PNG">
    </a>
</p>
<p align="center" width="100%">
    <a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/3.PNG" data-lightbox="1" data-title="1">
      <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/3.PNG">
    </a>
</p>

`Spring Initializr`를 통해 생성한 프로젝트의 간단한 테스트 결과 정상적으로 `Spring Boot` 프로젝트가 동작하는 것을 확인하였다.
- HelloworldApplication.java
{% highlight java %}
@SpringBootApplication
@RestController
public class HelloworldApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloworldApplication.class, args);
    }

    @GetMapping
    public String test(){
        return "Hello World!!";
    }
}
{% endhighlight %}

- 실행 결과
<p align="center" width="100%">
    <a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/4.PNG" data-lightbox="1" data-title="1">
      <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/1/4.PNG">
    </a>
</p>

<script>
window.tooltips = window.tooltips || [];
window.tooltips.push(['#build', { content: "Maven, Gradle" }]);
window.tooltips.push(['#lang', { content: "Java, Kotlin, Groovy" }]);
</script>

