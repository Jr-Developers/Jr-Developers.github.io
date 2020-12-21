# '무엇' 때문에 스프링 부트를 편리하게 사용할 수 있을까?

`Spring Boot` 프로젝트는 생성 부터 설정, 실행 까지 편리하게 프로젝트를 구성 할 수 있습니다. [`Spring Initializr`](https://start.spring.io/)를 이용하여 프로젝트의 `Build Tool`, `Dependencies`, `언어 및 버전`등을 선택하면 프로젝트가 생성되며 이렇게 생성된 프로젝트는 별도의 `WAS` 설치 없이 바로 내부 `WAS`를 통해 바로 실행 및 빌드가 가능합니다. 또한 `XML`를 통한 설정이 아닌 `application.yml`를 통해 기본적인 설정이 가능하며 추가적인 설정은 `Class`를 통해 설정 가능합니다.

그렇다면 __'무엇'__ 때문에 `Spring Boot`를 우리가 편하게 사용 할 수 있는지 [`Spring Boot`](https://spring.io/projects/spring-boot) 문서를 통해 `Spring Boot`가 이야기 하는 `Spring Boot`의 특징을 살펴보겠습니다.  
>Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".
>We take an opinionated view of the Spring platform and third-party libraries so you can get started with minimum fuss. Most Spring Boot applications need minimal Spring configuration.
>> #### Features
>> - Create stand-alone Spring applications
>> - Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
>> - Provide opinionated 'starter' dependencies to simplify your build configuration
>> - Automatically configure Spring and 3rd party libraries whenever possible
>> - Provide production-ready features such as metrics, health checks, and externalized configuration
>> - Absolutely no code generation and no requirement for XML configuration

위의 문서 내용은 제가 `Spring Boot`를 사용하며 느꼈던 `Spring Boot`의 편의성과 많은 부분 일치하였습니다. 그럼 이제 어떻게 `Spring Boot`는 이러한 편의성을 제공해 주는지 살펴보겠습니다.

--------------------------

1. Provide opinionated 'starter' dependencies to simplify your build configuration

   `Spring`에 속한 프로젝트는 위의 사진과 같이 많은 프로젝트가 존재합니다. 모든 `Spring Porject`는 각각의 버전이 따로 존재하며 프로젝트 내부에서 쓰이는 `Spring`외의 라이브러리들의 버전도 존재합니다. 이러한 많은 `Spring` 프로젝트들을 사용하는 개발자가 의존성을 체크하면서 프로젝트를 구성하게 된다면 많은 시간이 소요될 것입니다.<br>
   `Spring Boot`는 프로젝트 구성을 위해 `stater`를 제공합니다. `spring-boot-starter-data-jpa`의 일부분을 통해 알아보겠습니다.
   - [spring-boot-starter-data-jpa/pom.xml](https://repo1.maven.org/maven2/org/springframework/boot/spring-boot-starter-data-jpa/2.4.1/spring-boot-starter-data-jpa-2.4.1.pom)
   ```xml
   ...
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-jpa</artifactId>
     <version>2.4.1</version>
   ...
     <dependencies>
       <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-aop</artifactId>
         <version>2.4.1</version>
         <scope>compile</scope>
       </dependency>
       <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-jdbc</artifactId>
         <version>2.4.1</version>
         <scope>compile</scope>
       </dependency>
       ...
       <dependency>
         <groupId>org.hibernate</groupId>
         <artifactId>hibernate-core</artifactId>
         <version>5.4.25.Final</version>
         <scope>compile</scope>
         <exclusions>
           <exclusion>
             <artifactId>jaxb-api</artifactId>
             <groupId>javax.xml.bind</groupId>
           </exclusion>
       ...
       <dependency>
         <groupId>org.springframework.data</groupId>
         <artifactId>spring-data-jpa</artifactId>
         <version>2.4.2</version>
         <scope>compile</scope>
         <exclusions>
           <exclusion>
             <artifactId>aspectjrt</artifactId>
             <groupId>org.aspectj</groupId>
           </exclusion>
         </exclusions>
       </dependency>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-aspects</artifactId>
         <version>5.3.2</version>
         <scope>compile</scope>
       </dependency>
     </dependencies>
   ...
   ```
   위의 내용과 같이 `spring-boot-starter-data-jpa`에는 `spring-data-jpa`를 사용하기 위한 모든 의존성이 설정되어 있으므로 현재 프로젝트의 `pom.xml`에는 `spring-boot-starter-data-jpa`의 `dependency`만 추가하면 `spring-data-jpa`가 사용 가능하게 됩니다.<br>
   그렇다면 `spring-boot-starter-data-jpa`의 버전과 빌드를 위한 `plugin`설정을 관리하는 `spring-boot-starter-parent`, `spring-boot-dependencies`를 살펴보겠습니다. 우선 생성한 프로젝트의 `pom.xml`에 `parent`로 설정되어있는 `spring-boot-starter-parent`를 확인해 보겠습니다.
   - [spring-boot-starter-parent/pom.xml](https://repo1.maven.org/maven2/org/springframework/boot/spring-boot-starter-parent/2.4.1/spring-boot-starter-parent-2.4.1.pom)
   ```xml
   ...
     <build>
       <resources>
         <resource>
           <directory>${basedir}/src/main/resources</directory>
           <filtering>true</filtering>
           <includes>
             <include>**/application*.yml</include>
             <include>**/application*.yaml</include>
             <include>**/application*.properties</include>
           </includes>
         </resource>
         <resource>
           <directory>${basedir}/src/main/resources</directory>
           <excludes>
             <exclude>**/application*.yml</exclude>
             <exclude>**/application*.yaml</exclude>
             <exclude>**/application*.properties</exclude>
           </excludes>
         </resource>
       </resources>
       <pluginManagement>
         <plugins>
           <plugin>
             <groupId>org.jetbrains.kotlin</groupId>
             <artifactId>kotlin-maven-plugin</artifactId>
             <version>${kotlin.version}</version>
             <configuration>
               <jvmTarget>${java.version}</jvmTarget>
               <javaParameters>true</javaParameters>
             </configuration>
             <executions>
               <execution>
                 <id>compile</id>
                 <phase>compile</phase>
                 <goals>
                   <goal>compile</goal>
                 </goals>
               </execution>
               <execution>
                 <id>test-compile</id>
                 <phase>test-compile</phase>
                 <goals>
                   <goal>test-compile</goal>
                 </goals>
               </execution>
             </executions>
           </plugin>
           <plugin>
             <groupId>org.apache.maven.plugins</groupId>
             <artifactId>maven-compiler-plugin</artifactId>
             <configuration>
               <parameters>true</parameters>
             </configuration>
           </plugin>          
           ...
         </plugins>
       </pluginManagement>
   ```
   위의 내용과 같이 `spring-boot-starter-parent`에는 해당 프로젝트의 `Build Tool`에 따른 빌드 `plugin`들이 설정되어있습니다. 다음은 `spring-boot-starter-parent`이 `parent`로 설정되어있는 `spring-boot-dependencies`를 살펴보겠습니다.
   - [spring-boot-dependencies/pom.xml](https://repo1.maven.org/maven2/org/springframework/boot/spring-boot-dependencies/2.4.1/spring-boot-dependencies-2.4.1.pom)
   ```xml
   ...
     <properties>
       <activemq.version>5.16.0</activemq.version>
       <antlr2.version>2.7.7</antlr2.version>
       <appengine-sdk.version>1.9.83</appengine-sdk.version>
       <artemis.version>2.15.0</artemis.version>
       <aspectj.version>1.9.6</aspectj.version>
       <assertj.version>3.18.1</assertj.version>
       <atomikos.version>4.0.6</atomikos.version>
       ...
     </properties>
     <dependencyManagement>
       <dependencies>
         <dependency>
           <groupId>org.apache.activemq</groupId>
           <artifactId>activemq-amqp</artifactId>
           <version>${activemq.version}</version>
         </dependency>
         <dependency>
           <groupId>org.apache.activemq</groupId>
           <artifactId>activemq-blueprint</artifactId>
           <version>${activemq.version}</version>
         </dependency>
       ...
     </dependencies>
     <build>
       <pluginManagement>
         <plugins>
           <plugin>
             <groupId>org.codehaus.mojo</groupId>
             <artifactId>build-helper-maven-plugin</artifactId>
             <version>${build-helper-maven-plugin.version}</version>
           </plugin>
           <plugin>
             <groupId>org.flywaydb</groupId>
             <artifactId>flyway-maven-plugin</artifactId>
             <version>${flyway.version}</version>
           </plugin>
         ...
         </plugins>
       </pluginManagement>
     </build>
   ...
   ```
   위의 내용과 같이 해당 `Spring Boot` 버전에서 사용되는 `stater` 및 라이브러리 버전과 빌드 `plugin`들의 버전이 정의되어 있어 프로젝트 `pom.xml`에 버전을 명시하지 않았다면 `spring-boot-dependencies`에 정의된 버전을 사용하게 됩니다.<br>
   이과 같이 `Spring Boot`를 사용하는 개발자는 `stater`, `spring-boot-starter-parent`, `spring-boot-dependencies`들로 인해 `Spring` 프로젝트들의 의존성을 신경쓰지 않고 바로 개발을 진행 할 수 있는 것입니다.
   
   
   
