---
layout: post
title:  "Hibernate Event Listener를 이용한 항목 입력 및 로깅"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot spring-data-jpa datasource jpa hibernate
---

DB 등록/수정 시 반복적으로 발생하는 로깅이나 Audit 항목의 입력을 `Hibernate Event Listener`를 이용하여 해결가능하다.
하지만 아래와 같은 제약 사항이 존재하므로 참고한다.
>In general, the lifecycle method of a portable application should not invoke EntityManager or Query operations, access other entity instances, or modify relationships within the same persistence context. A lifecycle callback method may modify the non-relationship state of the entity on which it is invoked.

`Hibernate Event Listener`가 지원 하는 `Callback Annotation`은 7종으로 자세한 내용과 사용방법은 `Hibernate`[Reference Doc](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#events-jpa-callbacks)에 기술되어 있다.

추가적으로 `JPA`의 `UPDATE` 경우 `Dirty Check`를 통해 변경된 항목이 있을 경우에만 `UPDATE`를 수행하므로 변경 사항이 없는 경우 <code class="inline">@PreUpdate</code>, <code class="inline">@PostUpdate</code>가 수행 되지 않는다.

또한 <code class="inline">@PreUpdate</code>, <code class="inline">@PostUpdate</code>는 `Flush` 시점 이전, 이후에 수행되므로 <code class="inline">@Transactional</code>과 로직의 구현에 따라 `save` 메소드 호출 이후에도 `Flush`가 발생 하지 않는 경우가 존재하여 항목들의 값이 입력되기 이전의 객체일 수 있다.

--------------------------------

우선 첫 번째 방법으로 `JPA Entity` 내부에 `JPA Event Listener Annotation`를 정의하여 구현하는 방법이다.
{% highlight java %}
@Setter
@Getter
@MappedSuperclass
public class Audit {
    @Id
    @Column(name = "ID")
    private String id;

    @Column(name = "INSERT_TIMESTAMP")
    private Timestamp insertTimestamp;
    @Column(name = "INSERT_HTTP_METHOD")
    private String insertHTTPMethod;
    @Column(name = "INSERT_URL")
    private String insertUrl;
    @Column(name = "INSERT_METHOD")
    private String insertMethod;

    @Column(name = "UPDATE_TIMESTAMP")
    private Timestamp updateTimestamp;
    @Column(name = "UPDATE_HTTP_METHOD")
    private String updateHTTPMethod;
    @Column(name = "UPDATE_URL")
    private String updateUrl;
    @Column(name = "UPDATE_METHOD")
    private String updateMethod;

    @PrePersist
    public void onPersist() {
        HttpServletRequest req = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        HandlerMethod handlerMethod = (HandlerMethod) req.getAttribute(HandlerMapping.BEST_MATCHING_HANDLER_ATTRIBUTE);
        this.id = UUID.randomUUID().toString();
        this.insertTimestamp = new Timestamp(System.currentTimeMillis());
        this.insertHTTPMethod = req.getMethod();
        this.insertUrl = (String) req.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        this.insertMethod = new TargetLengthBasedClassNameAbbreviator(10).abbreviate(handlerMethod.toString());
    }

    @PreUpdate
    public void onUpdate() {
        HttpServletRequest req = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        HandlerMethod handlerMethod = (HandlerMethod) req.getAttribute(HandlerMapping.BEST_MATCHING_HANDLER_ATTRIBUTE);
        this.updateTimestamp = new Timestamp(System.currentTimeMillis());
        this.updateHTTPMethod = req.getMethod();
        this.updateUrl = (String) req.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        this.updateMethod = new TargetLengthBasedClassNameAbbreviator(10).abbreviate(handlerMethod.toString());
    }
}
{% endhighlight %}

{% highlight java%}
@Table(name = "USER")
@Entity
@Setter
@Getter
@NoArgsConstructor
public class User extends Audit {
    @Column(name = "USER_ID")
    private String userId;

    @Column(name = "NAME")
    private String name;
}
{% endhighlight %}

- INSERT 확인 결과
<p align="center" width="100%">
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/1.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/1.PNG">
</a>
</p>
- UPDATE 확인 결과
<p align="center" width="100%">
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/2.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/2.PNG">
</a>
</p>

-------------------------

두 번째 방법은 `JPA Entity Class`에 <code class="inline">@EntityListeners</code>에 `JPA Event Listener Annotation`를 정의하여 구현한 `Class`들을 나열하는 방법이다.

{% highlight java %}
@Setter
@Getter
@NoArgsConstructor
@MappedSuperclass
public class Audit {
    @Id
    @Column(name = "ID")
    private String id;

    @Column(name = "INSERT_TIMESTAMP")
    private Timestamp insertTimestamp;
    @Column(name = "INSERT_HTTP_METHOD")
    private String insertHTTPMethod;
    @Column(name = "INSERT_URL")
    private String insertUrl;
    @Column(name = "INSERT_METHOD")
    private String insertMethod;

    @Column(name = "UPDATE_TIMESTAMP")
    private Timestamp updateTimestamp;
    @Column(name = "UPDATE_HTTP_METHOD")
    private String updateHTTPMethod;
    @Column(name = "UPDATE_URL")
    private String updateUrl;
    @Column(name = "UPDATE_METHOD")
    private String updateMethod;
}
{% endhighlight %}

{% highlight java %}
@Table(name = "USER")
@Entity
@Setter
@Getter
@NoArgsConstructor
@EntityListeners({PrePersistEventListener.class, PreUpdateEventListener.class})
public class User extends Audit {
    @Column(name = "USER_ID")
    private String userId;

    @Column(name = "NAME")
    private String name;
}
{% endhighlight %}

{% highlight java %}
public class PrePersistEventListener {
    @PrePersist
    public void onPersist(Audit audit) {
        HttpServletRequest req = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        HandlerMethod handlerMethod = (HandlerMethod) req.getAttribute(HandlerMapping.BEST_MATCHING_HANDLER_ATTRIBUTE);
        audit.setId(UUID.randomUUID().toString());
        audit.setInsertTimestamp(new Timestamp(System.currentTimeMillis()));
        audit.setInsertHTTPMethod(req.getMethod());
        audit.setInsertUrl((String) req.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE));
        audit.setInsertMethod(new TargetLengthBasedClassNameAbbreviator(10).abbreviate(handlerMethod.toString()));
    }
}
{% endhighlight %}

{% highlight java %}
public class PreUpdateEventListener {
    @PreUpdate
    public void onPersist(Audit audit) {
        HttpServletRequest req = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        HandlerMethod handlerMethod = (HandlerMethod) req.getAttribute(HandlerMapping.BEST_MATCHING_HANDLER_ATTRIBUTE);
        audit.setUpdateTimestamp(new Timestamp(System.currentTimeMillis()));
        audit.setUpdateHTTPMethod(req.getMethod());
        audit.setUpdateUrl((String) req.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE));
        audit.setUpdateMethod(new TargetLengthBasedClassNameAbbreviator(10).abbreviate(handlerMethod.toString()));
    }
}
{% endhighlight %}
- INSERT 확인 결과
<p align="center" width="100%">
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/3.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/3.PNG">
</a>
</p>
- UPDATE 확인 결과
<p align="center" width="100%">
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/4.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/4/4.PNG">
</a>
</p>
- Event Listener [소스](https://github.com/Jr-Developers/eventlistener)
