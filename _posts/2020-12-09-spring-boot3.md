---
layout: post
title:  "Spring Data JPA Multi DataSource"
date:   2020-12-08T20:00:00+09:00
author: 주니어 개발자
categories: spring-boot
tags: spring-boot spring-data-jpa datasource jpa 
---

DB 서버를 두개를 사용해야 하거나 두 개의 스키마를 사용해야 하는 경우 두 개의 `DataSource`를 구성해야 한다.

두 개의 `DataSource` 설정을 위해 `Spring Boot` Reference Doc[#Configure Two DataSources](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-two-datasources)에서 제시 하는 설정을 참고 하였으며 이에 따른 두개의 `EntityManager`설정은 `Spring Boot` Reference Doc#[Use Two EntityManagers](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-use-two-entity-managers)과 `Spring Data JPA` ReferenceDoc#[Annotation-based Configuration](https://docs.spring.io/spring-data/jpa/docs/2.4.2/reference/html/#jpa.java-config)를 참고하였다.

정상적으로 Multi Source 가 동작하는지 확인하기 위해서 두 개의 `Spring Boot`서버에서 두 개의 `H2`을 기동시키며 자신의 서버 `H2` 와 나머지 서버의 `H2` 서버에 접속해 볼 것이다.

첫 번째 서버는 <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 라고 지징하겠으며 `MEMBER`라는 Table 하나를 갖는다. 해당 서버에서는 자신의 `H2` DataSource 만 이용하며 `Rest API`를 통해 `CLUB`의 정보를 조회 할 것이다.
두 번째 서버는 <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버 라고 지칭하겠으며 `CLUB`이라는 Table 하나를 갖는다. 해당 서버에서는 두 개의 `H2` DataSource 를 이용하여 데이터를 조회 할 것 이다.<br>

<code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버에는 <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버에서 <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버의 `H2`에 접속하기 위해 이전의 [포스팅](https://jr-developers.github.io/spring-boot/2020/12/08/spring-boot2.html)한 `H2` TCP 포트 개방 설정을 하였다.
- <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 - `Main`
{% highlight java %}
@SpringBootApplication
@RestController
public class MemberApplication {

    @Autowired
    private MemberRepository memberRepository;

    public static void main(String[] args) {
        SpringApplication.run(MemberApplication.class, args);
    }

    @GetMapping("/member/{memberName}")
    public Member retrieveByMemberName(@PathVariable("memberName") String memberName) {
        return memberRepository.findById(memberName).orElseGet(() -> null);
    }

    @GetMapping("/club/{clubName}")
    public Object retrieveByClubName(@PathVariable("clubName") String clubName) {
        return new RestTemplate().getForObject("http://localhost:8081/club/" + clubName, Object.class);
    }
}
{% endhighlight %}
<code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 자신의 `MEMBER` 테이블 조회와 `RestTemplate`를 이용한 <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버의 `CLUB` 테이블 조회로 구성하였다.

-------------------------------------------------

<code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버의 `application.yml`에는 두 개의 `DataSource` 설정을 위한 자신의 `H2` 서버와 <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버의 `H2` 서버의 정보를 포함하였으며 `Hikari initialization-fail-timeout` 설정을 통해 <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버가 먼저 기동될 경우 <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버의 기동을 잠시 기다리도록 설정 하였다.<br>
_추가적으로 Hikari Connection Pool 설정은 [HikariCP Dead lock에서 벗어나기 (이론편)](https://woowabros.github.io/experience/2020/02/06/hikaricp-avoid-dead-lock.html)과 [HikariCP Dead lock에서 벗어나기 (실전편)](https://woowabros.github.io/experience/2020/02/06/hikaricp-avoid-dead-lock-2.html)에 자세한 실전 경험과 내용이 정리되어 있다._

- <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 - `application.yml`
{% highlight yml %}
...
spring:
  datasource1:
    url: jdbc:h2:mem:clubdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
    platform: club # 초기 CLUB 테이블 데이터 입력 - /resources/data-club.sql
    hikari:
      maximum-pool-size: 10
      minimum-idle: 3

  datasource2:
    url: jdbc:h2:tcp://localhost:9092/mem:memberdb # Member서버에서 설정한 TCP 포트로 접속한다. 
    username: sa
    password:
    driver-class-name: org.h2.Driver
    hikari:
      initialization-fail-timeout: 1000000 # Member 서버가 기동시간을 기다리기 위해 설정
      maximum-pool-size: 20
      minimum-idle: 6
{% endhighlight %}

-------------------------------------------------

두 개의 `DataSource`를 구성하기 여러 방법이 있으며 나는 `Spring Boot` Reference Doc에서 제시한 방법을 이용하였다. 해당 방법은 `Spring` 내부에서 기본적으로 사용하는 `DataSource` 설정 방법인것으로 확인하였다.

- `Spring` 기본 DataSource 설정 - [DataSourceConfiguration.java](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration.java)
{% highlight java %}
 abstract class DataSourceConfiguration {
    ...
    protected static <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
        return (T) properties.initializeDataSourceBuilder().type(type).build();
    }
    
    ...
    
	/**
	 * Hikari DataSource configuration.
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
 }
{% endhighlight %}

- <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버 - clubdb `DataSource` 설정
{% highlight java %}
@Configuration
@EnableJpaRepositories(
        basePackages = "io.github.jr.developers.club.jpa.club", // 해당 설정이 적용될 JPA Package
        entityManagerFactoryRef = "clubEntityManagerFactory",
        transactionManagerRef = "clubTransactionManager"
)
@EnableTransactionManagement
public class ClubDataSourceConfig {
    @Bean
    @Primary // 메인으로 사용될 DataSource
    @ConfigurationProperties("spring.datasource1")
    public DataSourceProperties clubDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary // 메인으로 사용될 DataSource
    @ConfigurationProperties("spring.datasource1.hikari")
    public HikariDataSource clubDataSource() {
        return clubDataSourceProperties().initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }


    @Bean
    @Primary // 메인으로 사용될 DataSource
    public LocalContainerEntityManagerFactoryBean clubEntityManagerFactory() {

        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        vendorAdapter.setGenerateDdl(true);
        vendorAdapter.setShowSql(true);
//        Properties props = new Properties();  // Properties에 Hibernate Config 설정 추가
//        props.setProperty("hibernate.format_sql", String.valueOf(true));
//        props.setProperty("hibernate.hbm2ddl.auto", "create");
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setJpaVendorAdapter(vendorAdapter);
        factory.setPackagesToScan("io.github.jr.developers.club.jpa.club");
        factory.setDataSource(clubDataSource());
//        factory.setJpaProperties(props);
        return factory;
    }

    @Bean
    @Primary // 메인으로 사용될 DataSource
    public PlatformTransactionManager clubTransactionManager(@Qualifier("clubEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        JpaTransactionManager txManager = new JpaTransactionManager();
        txManager.setEntityManagerFactory(entityManagerFactory);
        return txManager;
    }
}
{% endhighlight %}

{% highlight java %}
@Configuration
@EnableJpaRepositories(
        basePackages = "io.github.jr.developers.club.jpa.member",
        entityManagerFactoryRef = "memberEntityManagerFactory",
        transactionManagerRef = "memberTransactionManager"
)
@EnableTransactionManagement
public class MemberDataSourceConfig {
    @Bean
    @ConfigurationProperties("spring.datasource2")
    public DataSourceProperties memberDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @ConfigurationProperties("spring.datasource2.hikari")
    public HikariDataSource memberDataSource() {
        return memberDataSourceProperties().initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }


    @Bean
    public LocalContainerEntityManagerFactoryBean memberEntityManagerFactory() {

        HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        vendorAdapter.setGenerateDdl(true);

        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setJpaVendorAdapter(vendorAdapter);
        factory.setPackagesToScan("io.github.jr.developers.club.jpa.member");
        factory.setDataSource(memberDataSource());
        factory.setPersistenceUnitName("memberEntityManager");  // EntityManager 직접 사용시 가져오기 위한 Name 설정
        return factory;
    }

    @Bean
    public PlatformTransactionManager memberTransactionManager(@Qualifier("memberEntityManagerFactory") EntityManagerFactory entityManagerFactory) {

        JpaTransactionManager txManager = new JpaTransactionManager();
        txManager.setEntityManagerFactory(entityManagerFactory);
        return txManager;
    }
}
{% endhighlight %}
<code class="inline">@EnableJpaRepositories</code>를 통해 해당 `DataSource`에서 사용될 `JPA Repostiroy`의 Package의 경로와 `EntityManagerFactory`, `TransactionManager`를 등록하며 `EntityManagerFactory` 생성시 `Hibernate` 설정을 추가 할 수 있으며 기본으로 사용될 `DataSource`가 아닌 경우 `PersistenceUnitName`를 설정하여 `EntityManager`를 가져 올 수 있다.

-------------------------------------------------

- <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 - 동작 결과
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/3/1.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/3/1.PNG">
</a>

- <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버 - 동작 결과
{% highlight java %}
@SpringBootApplication
@RestController
public class ClubApplication {

    @Autowired
    private ClubRepository clubRepository;

    @Autowired
    private MemberRepository memberRepository;

    @PersistenceContext
    private EntityManager clubEntityManager;

    @PersistenceContext(unitName = "memberEntityManager")
    private EntityManager memberEntityManager;


    public static void main(String[] args) {
        SpringApplication.run(ClubApplication.class, args);
    }

    @GetMapping("/spring-data-jpa/club")
    public List<Club> retrieveClubs() {
        return clubRepository.findAll();
    }

    @GetMapping("/spring-data-jpa/member")
    public List<Member> retrieveMembers() {
        return memberRepository.findAll();
    }

    @GetMapping("/hibernate/club/{clubName}")
    public Club retrieveByClubName2(@PathVariable("clubName") String clubName) {
        return clubEntityManager
                .createQuery("select c from Club c where c.clubName = :clubName", Club.class)
                .setParameter("clubName", clubName)
                .getSingleResult();
    }

    @GetMapping("/hibernate/member/{memberName}")
    public Member retrieveByMemberName2(@PathVariable("memberName") String memberName) {
        return memberEntityManager
                .createQuery("select m from Member m where m.memberName = :memberName", Member.class)
                .setParameter("memberName", memberName)
                .getSingleResult();
    }
}
{% endhighlight %}
<p align="center" width="100%">
<a href="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/3/2.PNG" data-lightbox="1" data-title="1">
  <img src="https://raw.githubusercontent.com/Jr-Developers/Jr-Developers.github.io/master/assets/post/spring-boot/3/2.PNG">
</a>
</p>

- <code class="inline text">Member</code>{:style="color: blue; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/datasource-member)
- <code class="inline text">Club</code>{:style="color: red; font-weight: bold;"}서버 [소스](https://github.com/Jr-Developers/datasource-club)

