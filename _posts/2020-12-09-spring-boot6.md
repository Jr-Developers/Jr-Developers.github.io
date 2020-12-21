---
layout: post
title:  "Multi DataSource Transaction Manager 설정"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot datasource multi-datasource transaction
---

`Multi DataSource`를 구성하여 테스트 하던 중 하나의 `Transaction`에서 `Multi DataSource`의 `Transaction`이 하나로 묶여 동작하지 않는 것을 확인하였다.<br>
`DataSource`는 각각의 `PlatformTransactionManger`(JAP:`JpaTransactionManager`, MyBatis:`DataSourceTransactionManager`)에서 `Transaction`을 관리하며 <code class="inline">@Transactional</code>에 별도의 `TransacationManager`를 지정하지 않을 경우 <code class="inline">@Primary</code>로 지정한 `TrasactionManager`가 지정된다.<br>
이에 따라 추가적인 설정 없이 `Multi DataSource`을 이용하여 저장, 수정시 `Exception` 발생시 하나의 `Transaction`으로 `Rollback`이 동작하지 않는 경우가 발생 하였다.

--------------------------

- application.yml
{% highlight yml %}
spring:
  datasource1:
    url: jdbc:h2:mem:clubdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
    hikari:
      minimum-idle: 1
      maximum-pool-size: 10
  datasource2:
    url: jdbc:h2:mem:userdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
    hikari:
      minimum-idle: 1
      maximum-pool-size: 10
...
{% endhighlight %}

- ClubDataSourceConfig.java - 첫 번째 DataSource
{% highlight java %}
@Configuration
@EnableJpaRepositories(
        basePackages = {"io.github.jr.developers.txmanager.club"},
        entityManagerFactoryRef = "clubEntityManagerFactory",
        transactionManagerRef = "clubTransactionManager"
)
    public class ClubDataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource1")
    public DataSourceProperties clubDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource1.hikari")
    public HikariDataSource clubHikariDataSource() {
        return clubDataSourceProperties().initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean clubEntityManagerFactory() {
        HibernateJpaVendorAdapter jpaVendorAdapter = new HibernateJpaVendorAdapter();
        jpaVendorAdapter.setGenerateDdl(true);
        LocalContainerEntityManagerFactoryBean factoryBean = new LocalContainerEntityManagerFactoryBean();
        factoryBean.setJpaVendorAdapter(jpaVendorAdapter);
        factoryBean.setDataSource(clubHikariDataSource());
        factoryBean.setPackagesToScan("io.github.jr.developers.txmanager.club");
        factoryBean.setPersistenceUnitName("clubEntityManager");
        return factoryBean;
    }

    @Bean
    @Primary
    public PlatformTransactionManager clubTransactionManager(@Qualifier("clubEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
{% endhighlight %}

- UserDataSourceConfig.java - 두 번째 DataSource
{% highlight java %}
@Configuration
@EnableJpaRepositories(
        basePackages = {"io.github.jr.developers.txmanager.user"},
        entityManagerFactoryRef = "userEntityManagerFactory",
        transactionManagerRef = "userTransactionManager"
)
public class UserDataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource2")
    public DataSourceProperties userDataSourceProperties(){
        return new DataSourceProperties();
    }

    @Bean
    @ConfigurationProperties("spring.datasource2.hikari")
    public HikariDataSource userHikariDataSource(){
        return userDataSourceProperties().initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean userEntityManagerFactory(){
        HibernateJpaVendorAdapter vendorAdapter =  new HibernateJpaVendorAdapter();
        vendorAdapter.setGenerateDdl(true);

        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setJpaVendorAdapter(vendorAdapter);
        factory.setDataSource(userHikariDataSource());
        factory.setPackagesToScan("io.github.jr.developers.txmanager.user");
        factory.setPersistenceUnitName("userEntityManager");
        return factory;
    }

    @Bean
    public PlatformTransactionManager userTransactionManager(@Qualifier("userEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}

{% endhighlight %}

--------------------

- 테스트 로직 
{% highlight java %}
    ...
    @GetMapping("/commit")
    @Transactional
    public void commit(){
        Club club = new Club();
        club.setId("Commit");
        User user = new User();
        user.setId("Commit");

        clubService.create(club);
        Thread.sleep(3000);
        userService.create(user);
        Thread.sleep(3000);
    }

    @GetMapping("/rollback")
    @Transactional
    public void rollback(){
        Club club = new Club();
        club.setId("Rollback");
        User user = new User();
        user.setId("Rollback");

        clubService.create(club);
        Thread.sleep(3000);
        userService.create(user);
        Thread.sleep(3000);
        throw new RuntimeException();
    }
    ...
{% endhighlight %}

---------------------------

먼저 추가적이 설정 없이 `commit` 메소드를 실행시 `Club`과 `User` 모두 정상적으로 저장된것을 확인하였다.
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/1.JPG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/1.JPG">
</a>
하지만 로그를 확인해보면 `insert`를 위한 `select`의 실행 순서와 `insert` 실행 순서가 다른것이 확인된다.<br>
우선 `Exception` 발생시를 먼저 확인해 보겠다.
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/2.JPG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/2.JPG">
</a>
`USER` 테이블에는 정상적으로 `Rollback`이 수행되었지만 `USER` 테이블에는 `Commit`이 수행되었다.

테스트를 위한 `commit` 메소드의 로그를 보면 `User`의 `insert` 실행은 59초이며 `Club`의 `insert` 실행은 3초 이후에 이루어 졌다.<br>
 이는 `Club`의 `Transaction`의 시작은 <code class="inline">@Transactional</code>이 선언된 <code class="inline">public void commit()</code>메소드의 시작이고 끝은 <code class="inline">public void commit()</code>메소드의 종료 이다. 이에 따라 `Club`의 `insert`는 메소드 종료 시점에 발생한것이다.<br>
 `User`의 <code class="inline">userService.create(user)</code>도 해당 `Transaction`의 참여했다면 <code class="inline">public void commit()</code>메소드 종료 시점에 `insert`가 발생했겠지만 그렇지 못하고 새로운 `Transacation`을 발생시켜 <code class="inline">userService.create(user)</code> 메소드 실행 직후 `insert`가 발생하였다.<br>
 이에 따라 `Exception`이 발생하였지만 `User`의 <code class="inline">userService.create(user)</code>은 다른 `Transacation`이기 때문에 이미 `Commit`이 수행 된이후 `RuntimeException`의 발생으로 `Club`의 `Transaction`의 `Rollback`이 수행된 것이다.
 
 -----------------------------------

이를 위한 해결 방법은 [`JTA`](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-jta)와 [`ChainedTransactionManager`](https://github.com/spring-projects/spring-data-commons/blob/master/src/main/java/org/springframework/data/transaction/ChainedTransactionManager.java)가 존재한다. `JTA`는 별도로 공부를 해야할 부분이 존재하여 `ChainedTransactionManager`를 이용하여 `Transcation`을 하나로 묶어보겠다.<br>
하지만 `ChainedTransactionManager`은 Java의 비지니스 로직에서 발생하는 `Exception`의 경우 `Rollback`이 가능하지만 비지니스 로직 이후 `Commit`이 발생하여 `DB`에서 `Exception`이 발생하는 경우에는 `Rollback`이 불가능하다. 이에 따라 `ChainedTransactionManager` 생성시 `DB`에서 `Exception`이 발생할 가능성이 높은 `TransactionManager`를 생성자의 마지막에 설정해야한다.
>  The configured instances will start transactions in the order given and commit/rollback in <em>reverse</em> order, which means the  PlatformTransactionManager most likely to break the transaction should be the <em>last</em> in the list configured. A PlatformTransactionManager throwing an exception during commit will automatically cause the remaining transaction managers to roll back instead of committing.

- ChainedTransactionManagerConfig
{% highlight java %}
@Configuration
public class ChainedTransactionManagerConfig {

    @Bean
    @Primary // 기존에 정의한 @Primary는 해당 PlatformTransactionManager로 변경
    public PlatformTransactionManager chainedTransactionManager(
            @Qualifier("clubTransactionManager") PlatformTransactionManager clubTransactionManager,
            @Qualifier("userTransactionManager") PlatformTransactionManager userTransactionManager) {
        return new ChainedTransactionManager(clubTransactionManager, userTransactionManager);
    }
}
{% endhighlight %}

- ChainedTransactionManager 적용 - Commit
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/3.JPG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/3.JPG">
</a>
정상적으로 <code class="inline">public void commit()</code> 메소드 종료이후 `Club`과 `User`의 `insert`가 실행되는것으로 확인되었다.

- ChainedTransactionManager 적용 - Rollbacks
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/4.JPG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/6/4.JPG">
</a>
정상적으로 <code class="inline">public void rollback()</code> 메소드 안에서 `Exception` 발생시 `insert`가 실행되지 않는것으로 확인되었다.

- txmanager [소스](https://github.com/Jr-Developers/txmanager)
