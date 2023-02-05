# Spring Boot 中文参考指南

> Spring Boot 版本 2.7.8
>
> 原文：https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/

# 1. 文档预览

# 2. 开始

本节对Spring Boot进行介绍 以及如何安装，我们将引导您构建第一个Spring Boot 应用，同时讨论一些核心准则。

## 2.1 Spring Boot 介绍

Spring Boot 帮助您创建可以独立运行的，生产级的Spring 应用程序。

您创建的Spring Boot 应用程序，可以通过`java -jar` 或者 传统的war包方式启动，另外还提供了一个运行`spring scripts`的命令行工具。

## 2.2 系统要求

Spring Boot 2.7.8 需要Java8 ，兼容Java19，Spring 版本5.3.25或更高。

| 构建工具 | 版本                  |
| :------- | :-------------------- |
| Maven    | 3.5+                  |
| Gradle   | 6.8.x, 6.9.x, and 7.x |

### 2.2.1 Servlet 容器

Spring Boot 支持如下嵌入式servlet容器：

| 名称         | Servlet 版本 |
| :----------- | :----------- |
| Tomcat 9.0   | 4.0          |
| Jetty 9.4    | 3.1          |
| Jetty 10.0   | 4.0          |
| Undertow 2.0 | 4.0          |

您也可以将Spring Boot部署到任何兼容servlet 3.1+的容器中。

## 2.3 Spring Boot 安装

安装之前使用`java -version`检查 Java 版本，Spring Boot 2.7.8 需要Java8 或更高的版本。

### 2.3.1 面向Java开发人员

#### Maven 安装

Spring Boot 依赖项使用`org.springframework.boot` `groupId`。通常，您的Maven POM文件继承自`spring-boot-starter-parent` ,并声明一个或多个“Starters”的依赖关系。

另外Spring Boot 还提供了可选的Maven 插件来创建可执行的jar,更多信息参考 [Spring Boot Maven 插件文档](https://docs.spring.io/spring-boot/docs/2.7.8/maven-plugin/reference/htmlsingle/#getting-started)。

#### Gradle 安装

同Maven，Spring Boot 也提供了一个 Gradle插件，用于创建可执行的jar，更多信息参考 [Spring Boot Gradle 插件文档](https://docs.spring.io/spring-boot/docs/2.7.8/gradle-plugin/reference/htmlsingle/#getting-started)。

### 2.3.2 安装Spring Boot CLI

Spring Boot CLI是一个命令行工具，可用于快速创建Spring Boot 初始化应用程序，这在没有IDE的情况下非常有用。

#### 手动安装

您可以从如下地址下载Spring CLI发行版本：

- [spring-boot-cli-2.7.8-bin.zip](https://repo.spring.io/release/org/springframework/boot/spring-boot-cli/2.7.8/spring-boot-cli-2.7.8-bin.zip)
- [spring-boot-cli-2.7.8-bin.tar.gz](https://repo.spring.io/release/org/springframework/boot/spring-boot-cli/2.7.8/spring-boot-cli-2.7.8-bin.tar.gz)

另外提供了 [快照列表](https://repo.spring.io/ui/native/snapshot/org/springframework/boot/spring-boot-cli/)

下载后，按照解压缩存档中的[INSTALL.txt](https://raw.githubusercontent.com/spring-projects/spring-boot/v2.7.8/spring-boot-project/spring-boot-cli/src/main/content/INSTALL.txt)说明进行操作。.zip文件的bin/目录中有一个spring脚本（适用于Windows的spring.bat），或者可以使用`jar -jar `运行 jar包。

#### 使用SDKMAN安装

SDKMAN（软件开发工具包管理器）可用于管理各种二进制SDK版本，包括Groovy和Spring Boot CLI。从[sdkman.io](https://sdkman.io/)获取并使用以下命令安装 Spring Boot：

```shell
$ sdk install springboot
$ spring --version
Spring CLI v2.7.8
```

如果您为 CLI 开发功能并希望访问您构建的版本，请使用以下命令：

```shell
$ sdk install springboot dev /path/to/spring-boot/spring-boot-cli/target/spring-boot-cli-2.7.8-bin/spring-2.7.8/
$ sdk default springboot dev
$ spring --version
Spring CLI v2.7.8
```

前面的说明安装了一个`spring`名为instance 的本地`dev`实例。它指向您的目标构建位置，因此每次您重建 Spring Boot 时，`spring`它都是最新的。

您可以通过运行以下命令来查看它：

```shell
$ sdk ls springboot

================================================================================
Available Springboot Versions
================================================================================
> + dev
* 2.7.8

================================================================================
+ - local version
* - installed
> - currently in use
================================================================================
```

#### 使用OSX Homebrew安装

在Mac上可以使用[Homebrew](https://brew.sh/)安装。

```shell
$ brew tap spring-io/tap
$ brew install spring-boot
```

Homebrew 安装 `spring` 到 `/usr/local/bin`.

> 如果找不到这个命令，尝试使用`brew update`更新后重试

#### 使用MacPorts安装

在Mac上使用[MacPorts](https://www.macports.org/) 安装。

```shell
$ sudo port install spring-boot-cli
```

#### 使用 Windows Scoop安装

在Window使用[Scoop](https://scoop.sh/)安装。

```shell
> scoop bucket add extras
> scoop install springboot
```

Scoop 安装 `spring` 到 `~/scoop/apps/springboot/current/bin`

> 如果提示命令不存，请使用`scoop update`更新后再重试

#### Spring CLI 快速启动示例

首先创建一个名为`app.groovy`的文件。

```groovy
@RestController
class ThisWillActuallyRun {

    @RequestMapping("/")
    String home() {
        "Hello World!"
    }

}
```

然后使用如下命令运行：

```shell
$ spring run app.groovy
```

第一次运行需要下载依赖，会比较慢，后面运行会快很多。

最后，使用浏览器打开`localhost:8080`，输出

```
Hello World!
```

## 2.4 开发第一个Spring Boot 应用程序

建议使用[start.spring.io](https://start.spring.io/) 创建Spring Boot 应用程序。

# 3. 升级Spring Boot

## 3.1 从1.x升级

从1.x升级，可以查看[GitHub wiki上的升级指南](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Migration-Guide)

## 3.2 升级到最新的功能版本

Spring Boot提供了一种方法来分析应用程序的环境并在启动时打印诊断信息，还可以在运行时临时迁移属性，要启动该功能，在项目中添加以下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-properties-migrator</artifactId>
    <scope>runtime</scope>
</dependency>
```

完成升级后，删除该依赖。

# 4. Spring Boot开发

## 4.1 构建系统

可以使用Maven、Gradle、Ant 构建系统

### 4.1.1 Starters

所有官方启动器都遵循类似的命名模式：`spring-boot-starter-*`，其中`*`是特定类型的应用程序，如果是自己创建的启动器一般以项目名称开头，如`thirdpartyproject-spring-boot-starter`。

Spring Boot 提供了以下应用启动器`org.springframework.boot`：

| 名称                                          | 描述                                                         |
| :-------------------------------------------- | :----------------------------------------------------------- |
| `spring-boot-starter`                         | Core starter，包括自动配置支持、日志记录和 YAML              |
| `spring-boot-starter-activemq`                | 使用 Apache ActiveMQ 的 JMS 消息传递启动器                   |
| `spring-boot-starter-amqp`                    | 使用 Spring AMQP 和 Rabbit MQ 的启动器                       |
| `spring-boot-starter-aop`                     | 使用 Spring AOP 和 AspectJ 进行面向方面编程的入门            |
| `spring-boot-starter-artemis`                 | 使用 Apache Artemis 的 JMS 消息传递启动器                    |
| `spring-boot-starter-batch`                   | 使用 Spring Batch 的启动器                                   |
| `spring-boot-starter-cache`                   | 使用 Spring Framework 的缓存支持的 Starter                   |
| `spring-boot-starter-data-cassandra`          | 使用 Cassandra 分布式数据库和 Spring Data Cassandra 的 Starter |
| `spring-boot-starter-data-cassandra-reactive` | 使用 Cassandra 分布式数据库和 Spring Data Cassandra Reactive 的 Starter |
| `spring-boot-starter-data-couchbase`          | 使用 Couchbase 面向文档的数据库和 Spring Data Couchbase 的启动器 |
| `spring-boot-starter-data-couchbase-reactive` | 使用 Couchbase 面向文档的数据库和 Spring Data Couchbase Reactive 的 Starter |
| `spring-boot-starter-data-elasticsearch`      | 使用 Elasticsearch 搜索和分析引擎以及 Spring Data Elasticsearch 的 Starter |
| `spring-boot-starter-data-jdbc`               | 使用 Spring Data JDBC 的启动器                               |
| `spring-boot-starter-data-jpa`                | 将 Spring Data JPA 与 Hibernate 一起使用的启动器             |
| `spring-boot-starter-data-ldap`               | 使用 Spring Data LDAP 的启动器                               |
| `spring-boot-starter-data-mongodb`            | 使用 MongoDB 面向文档的数据库和 Spring Data MongoDB 的启动器 |
| `spring-boot-starter-data-mongodb-reactive`   | 使用 MongoDB 文档型数据库和 Spring Data MongoDB Reactive 的 Starter |
| `spring-boot-starter-data-neo4j`              | 使用 Neo4j 图形数据库和 Spring Data Neo4j 的启动器           |
| `spring-boot-starter-data-r2dbc`              | 使用 Spring Data R2DBC 的启动器                              |
| `spring-boot-starter-data-redis`              | 用于将 Redis 键值数据存储与 Spring Data Redis 和 Lettuce 客户端一起使用的 Starter |
| `spring-boot-starter-data-redis-reactive`     | 将 Redis 键值数据存储与 Spring Data Redis 反应式和 Lettuce 客户端一起使用的启动器 |
| `spring-boot-starter-data-rest`               | 使用 Spring Data REST 通过 REST 公开 Spring Data 存储库的 Starter |
| `spring-boot-starter-freemarker`              | 使用 FreeMarker 视图构建 MVC Web 应用程序的启动器            |
| `spring-boot-starter-graphql`                 | 使用 Spring GraphQL 构建 GraphQL 应用程序的 Starter          |
| `spring-boot-starter-groovy-templates`        | 使用 Groovy 模板视图构建 MVC web 应用程序的启动器            |
| `spring-boot-starter-hateoas`                 | 使用 Spring MVC 和 Spring HATEOAS 构建基于超媒体的 RESTful Web 应用程序的启动器 |
| `spring-boot-starter-integration`             | 使用 Spring Integration 的启动器                             |
| `spring-boot-starter-jdbc`                    | 将 JDBC 与 HikariCP 连接池一起使用的启动器                   |
| `spring-boot-starter-jersey`                  | 使用 JAX-RS 和 Jersey 构建 RESTful Web 应用程序的启动器。的替代品[`spring-boot-starter-web`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-web) |
| `spring-boot-starter-jooq`                    | 使用 jOOQ 通过 JDBC 访问 SQL 数据库的启动器。替代[`spring-boot-starter-data-jpa`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-data-jpa)或[`spring-boot-starter-jdbc`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-jdbc) |
| `spring-boot-starter-json`                    | 读写json的starter                                            |
| `spring-boot-starter-jta-atomikos`            | 使用 Atomikos 的 JTA 事务启动器                              |
| `spring-boot-starter-mail`                    | 使用 Java Mail 和 Spring Framework 的电子邮件发送支持的 Starter |
| `spring-boot-starter-mustache`                | 使用 Mustache 视图构建 Web 应用程序的启动器                  |
| `spring-boot-starter-oauth2-client`           | 使用 Spring Security 的 OAuth2/OpenID Connect 客户端功能的 Starter |
| `spring-boot-starter-oauth2-resource-server`  | 使用 Spring Security 的 OAuth2 资源服务器功能的启动器        |
| `spring-boot-starter-quartz`                  | 使用 Quartz 调度器的启动器                                   |
| `spring-boot-starter-rsocket`                 | 用于构建 RSocket 客户端和服务器的启动器                      |
| `spring-boot-starter-security`                | 使用 Spring Security 的启动器                                |
| `spring-boot-starter-test`                    | 用于使用 JUnit Jupiter、Hamcrest 和 Mockito 等库测试 Spring Boot 应用程序的 Starter |
| `spring-boot-starter-thymeleaf`               | 使用 Thymeleaf 视图构建 MVC Web 应用程序的启动器             |
| `spring-boot-starter-validation`              | 将 Java Bean Validation 与 Hibernate Validator 结合使用的 Starter |
| `spring-boot-starter-web`                     | 用于使用 Spring MVC 构建 Web（包括 RESTful）应用程序的 Starter。使用 Tomcat 作为默认的嵌入式容器 |
| `spring-boot-starter-web-services`            | 使用 Spring Web 服务的启动器                                 |
| `spring-boot-starter-webflux`                 | 用于使用 Spring Framework 的 Reactive Web 支持构建 WebFlux 应用程序的 Starter |
| `spring-boot-starter-websocket`               | 使用 Spring Framework 的 MVC WebSocket 支持构建 WebSocket 应用程序的 Starter |

除了应用程序启动器之外，还可以使用以下启动器来添加*[生产就绪](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#actuator)*功能：

| 名称                           | 描述                                                         |
| :----------------------------- | :----------------------------------------------------------- |
| `spring-boot-starter-actuator` | 使用 Spring Boot Actuator 的 Starter，它提供生产就绪功能来帮助您监控和管理您的应用程序 |

最后，Spring Boot 还包括以下启动器:

| 名称                                | 描述                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| `spring-boot-starter-jetty`         | 使用 Jetty 作为嵌入式 servlet 容器的启动器的替代品[`spring-boot-starter-tomcat`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-tomcat) |
| `spring-boot-starter-log4j2`        | 使用 Log4j2 进行日志记录的启动器的替代品[`spring-boot-starter-logging`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-logging) |
| `spring-boot-starter-logging`       | 使用 Logback 进行日志记录的启动器。默认日志记录启动器        |
| `spring-boot-starter-reactor-netty` | 使用 Reactor Netty 作为嵌入式响应式 HTTP 服务器的启动器。    |
| `spring-boot-starter-tomcat`        | 将 Tomcat 用作嵌入式 servlet 容器的启动器。使用的默认 servlet 容器启动器[`spring-boot-starter-web`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-web) |
| `spring-boot-starter-undertow`      | 使用 Undertow 作为嵌入式 servlet 容器的启动器的替代品[`spring-boot-starter-tomcat`](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#spring-boot-starter-tomcat) |

> 其他社区贡献的starter列表，请参阅GitHub 上模块 中的[自述文件。](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-starters/README.adoc)`spring-boot-starters`

## 4.2 构建代码

Spring Boot 没有固定的代码布局，但是有些实践提供参考。

### 4.2.1 "default"包

当一个类不包含`package`时，它被认为在“default package”中。通常不建议使用“default package”，它可能会导致`@ComponentScan`、`@ConfigurationPropertiesScan`、`@EntityScan`或`@SpringBootApplication` 注解出现问题，我们应该遵循推荐的包命名方式，比如`com.example.project`

### 4.2.2 主程序类

通常建议将主程序类放在其他类之上的根包中，`@SpringBootApplication`通常放在主类中，其隐式的定义了基本的包搜索功能，其内部引入了`@EnableAutoConfiguration`和`@ComponentScan`。

下面是一个典型的布局：

```
com
 +- example
     +- myapplication
         +- MyApplication.java
         |
         +- customer
         |   +- Customer.java
         |   +- CustomerController.java
         |   +- CustomerService.java
         |   +- CustomerRepository.java
         |
         +- order
             +- Order.java
             +- OrderController.java
             +- OrderService.java
             +- OrderRepository.java
```

`MyApplication.java` 定义了一个`main`方法以及`@SpringBootApplication`,如下所示：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

## 4.3 配置类

Spring Boot 支持使用Java进行配置，虽然SpringApplication 跟XML可以一起使用，但还是建议`@Configuration`是独立的类。

### 4.3.1 导入其他的配置类

你不需要把所有的`@Configuration` 放到一个类中，该`@Import`注解可用于导入其他的配置类，或者可以使用`@ComponentScan`自动获取所有的Spring 组件，包括`@Configuration`类。

### 4.3.2 导入XML配置

如果你还是要使用XML配置，依然建议使用`@Configuration`类，然后使用`@ImportResource`来加载XML配置。

## 4.4 自动配置

Spring Boot会尝试将starter自动配置到应用程序，比如引入了`HSQLDB`的starter，但是没有手动配置任何数据库连接bean，那么Spring Boot 会自动配置一个内存数据库。

开启自动配置，需要添加`@EnableAutoConfiguration`或者`@SpringBootApplication`。

### 4.4.1 替换自动配置

自动配置是非侵入式的，任何时候都可以使用自定义配置替换自动配置的指定部分，比如，添加了`DataSource` bean，默认的嵌入式数据库就会被替换。

使用`--debug`启动应用程序，可以打印出当前应用了哪些自动配置。

### 4.4.2 禁用指定的自动配置类

如果想要禁用指定的自动配置类，可以使用`@SpringBootApplication`的`exclude`属性，如：

```java
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;

@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApplication {

}
```

如果排除的类不在类路径中，可以使用`excludeName`指定类的完全限定名，另外如果不用`@SpringBootApplication`，`@EnableAutoConfiguration`的`exclude`和`excludeName`也是可用的。

最后也能用`spring.autoconfigure.exclude`的配置来排除自动配置类。

## 4.5 Spring Beans和依赖注入

通常建议使用构造函数注入依赖项，和`@ComponentScan`查找bean。

如果是按照4.2的方式构建的代码，则可以使用`@ComponentScan`不带任何参数或者使用`@SpringBootApplication`其已经包含了`@ComponentScan`注解，这样所有的组件（`@Component`、`@Service`、`@Repository`、`@Controller`和其他）都会自动注册为Spring Beans。

如下示例表示一个`@Service`使用构造函数来注入`RiskAssessor` Bean。

```java
import org.springframework.stereotype.Service;

@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;

    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
    }

    // ...

}
```

如果一个Bean有多个构造函数，需要使用`@Autowired`标记哪个需要Spring 注入：

```java
import java.io.PrintStream;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;

    private final PrintStream out;

    @Autowired
    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
        this.out = System.out;
    }

    public MyAccountService(RiskAssessor riskAssessor, PrintStream out) {
        this.riskAssessor = riskAssessor;
        this.out = out;
    }

    // ...

}
```

> 使用构造函数注入应该使用 `final`标记，表示后面不能再被修改。

## 4.6 使用@SpringBootApplication 注解

使用`@SpringBootApplication`注解可以启用如下三个功能：

- `@EnableAutoConfiguration`: 启用Spring Boot 的自动配置机制

- `@ComponentScan @Component`：在应用程序所在的包上启用扫描
- `@SpringBootConfiguration`: 允许在上下文中注册额外的 beans 或导入额外的配置类。Spring 标准的替代方案`@Configuration`，有助于在集成测试中进行配置检测。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// Same as @SpringBootConfiguration @EnableAutoConfiguration @ComponentScan
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

如果不想使用@SpringBootApplication，也可以单独使用注解，如下示例并未使用@ComponentScan 自动扫描功能，而使用显示导入(`@Import`)：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringBootConfiguration;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.Import;

@SpringBootConfiguration(proxyBeanMethods = false)
@EnableAutoConfiguration
@Import({ SomeConfiguration.class, AnotherConfiguration.class })
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

## 4.7 运行应用程序

### 4.7.1 从IDE运行

### 4.7.2 作为打包应用程序运行

使用`java -jar`运行：

```
$ java -jar target/myapplication-0.0.1-SNAPSHOT.jar
```

也可以附加远程调式器：

```
$ java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n \
       -jar target/myapplication-0.0.1-SNAPSHOT.jar
```

### 4.7.3 使用Maven插件

Spring Boot Maven 插件包含一个`run`命令：

```
$ mvn spring-boot:run
```

另外还可以使用`MAVEN_OPTS` 设置环境变量：

```
$ export MAVEN_OPTS=-Xmx1024m
```

### 4.7.4 使用Gradle插件

Gradle插件包含一个`bootRun`命令：

```
$ gradle bootRun
```

使用`JAVA_OPTS`设置环境变量：

```
$ export JAVA_OPTS=-Xmx1024m
```

### 4.7.5 热插拨

Spring Boot 的热插拨基于JVM，JVM在某种程序上受限于可以替换的字节码，对于完整方案可以使用[JRebel ](https://www.jrebel.com/products/jrebel)。

`spring-boot-devtools`模块还包括对应用程序快速重启的支持，详细信息查看后面的[热插拔“操作方法”](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#howto.hotswapping)。

![](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/CSDN-%E5%85%AC%E4%BC%97%E5%8F%B7-20230125234615204-20230125234702260.png)

## 4.8 开发者工具

Spring Boot 提供`spring-boot-devtools` 模块提供开发时的额外功能，要支持该功能，需要将依赖添加到项目中:

*Maven*

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

*Gradle*

```
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}
```

>  默认情况下，打包的应用程序不包含devtools，如果想要使用某个远程devtool特性，在Maven插件中配置，excludeDevtools为false，Gradle插件中配置task任务以包含`developmentOnly`，如
>
> ```
> tasks.named("bootWar") {
> 	classpath configurations.developmentOnly
> }
> ```



> 当打包生产应用程序时，开发者工具将被自动禁用。如果您的应用程序是从一个特殊的类加载器启动的或者 使用`java -jar`，那么会被认为是一个"生产的应用程序"。可以使用`spring.devtools.restart.enabled`来控制，要开启devtools，使用`-Dspring.devtools.restart.enabled=true`启动，要禁用devtools，排除依赖项或者使用`-Dspring.devtools.restart.enabled=false`启动。

Maven 中使用optional，Gradle 使用  developmentOnly，表示可以防止devtools被传递到项目的其他模块。

### 4.8.1 诊断类加载问题

开发者工具的重启功能是通过使用两个类加载器实现的，对于大不多应用程序效果很好，但是有时候会导致类加载问题，特别是在多模块项目中。

要判断是不是由于这个问题，可以[尝试禁用重启](#禁用重启)，使用`spring.devtools.restart.enabled=false`属性禁用它。

另外可以 [自定义重启类加载器](#自定义重启类加载器)，自定义由哪个类加载加载，详见[4.8.3自动重启](#4.8.3 自动重启)。 

### 4.8.2 属性默认值

Spring Boot 的一些库使用缓存来提高性能，比如，模版引擎会缓存编译后的模版，以此避免重复解析，但这样在开发过程中我们就不能即时看到模版的变更。spring-boot-devtools 默认禁用了缓存。

下表列出了所有应用的属性：

| 名称                                             | 默认值   |
| :----------------------------------------------- | :------- |
| `server.error.include-binding-errors`            | `always` |
| `server.error.include-message`                   | `always` |
| `server.error.include-stacktrace`                | `always` |
| `server.servlet.jsp.init-parameters.development` | `true`   |
| `server.servlet.session.persistent`              | `true`   |
| `spring.freemarker.cache`                        | `false`  |
| `spring.graphql.graphiql.enabled`                | `true`   |
| `spring.groovy.template.cache`                   | `false`  |
| `spring.h2.console.enabled`                      | `true`   |
| `spring.mustache.servlet.cache`                  | `false`  |
| `spring.mvc.log-resolved-exception`              | `true`   |
| `spring.template.provider.cache`                 | `false`  |
| `spring.thymeleaf.cache`                         | `false`  |
| `spring.web.resources.cache.period`              | `0`      |
| `spring.web.resources.chain.cache`               | `false`  |

> 如果不想应用属性默认值，可以在应用程序配置文件中配置`spring.devtools.add-properties=false`



在开发WEB应用的时候，可以开启`DEBUG`日志，这样会显示请求、正在处理的程序，响应结果和其他详细信息，如果希望显示所有的详细信息（比如潜在的敏感信息），可以打开`spring.mvc.log-request-details`或`spring.codec.log-request-details`。



> ⚠️笔者注：
>
> 开启spring.mvc.log-request-details 后的日志![image-20230125202313711](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/image-20230125202313711.png)
>
> 关闭spring.mvc.log-request-details后的日志：
>
> ![image-20230125203304926](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/image-20230125203304926.png)



### 4.8.3 自动重启

只要类路径上的文件发生变更，使用了`spring-boot-devtools`的应用程序就会自动重启，但是某些资源（如静态资源和视图模版）不需要重启应用程序。

> 触发重启的方法：
>
> 由于DevTools 通过监听类路径上的资源来触发重启，所以不管使用哪个IDE都需要重新编译后才能触发重启：
>
> Eclipse 中，保存修改后会更新类文件并触发重启
>
> IDEA中，通过Build 触发或者编辑项目的`Edit Configurations -> On Update action：Update classes and resources`也可以触发重启
>
> 使用构建工具，`mvn compile`或者`gradle build`可以触发重启

>⚠️笔者注：
>
>官方文档提示：使用Maven或者Gradle时，需要将`forking`设置为`enabled`,才能触发重启。
>
>实测，新版本的`spring-boot-maven-plugin`在项目引入`spring-boot-devtools`后会自动开启`fork`，如图：
>
>![image-20230125221020529](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/image-20230125221020529.png)
>
>并且插件的注释也标记为过期，将在3.0.0中彻底删除：
>
>![image-20230125221154769](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/image-20230125221154769.png)

在重启期间 DevTools 依赖应用上下文的 shutdown hook 来关闭，如果设置为`SpringApplication.setRegisterShutdownHook(false)`，就会导致其无法正常工作。

>⚠️笔者注：
>
>在笔者按照这样设置后，发现自动重启并无失效
>
>```java
>public static void main(String[] args) {
>  SpringApplication application = new SpringApplication(SpringBootDemoApplication.class);
>  application.setRegisterShutdownHook(false);
>  application.run(args);
>}
>```

> AspectJ 切面不支持自动重启

重新启动与重新加载

Spring Boot 的重启技术通过使用两个类加载器来工作的，不会更改的类（如：第三方jar的类）被加载到基类加载器中，频繁修改的类被加载到一个重启类加载器中。当应用程序重启时，旧的重启类加载器被丢弃并创建一个新的类加载器，这种方法会被“冷启动”快得多，因为基类加载器已经可用。

如果自动重启还是比较慢的，或者遇到类加载问题，可用尝试使用重新加载技术，如[JRebel](https://jrebel.com/software/jrebel/)，他们通过加载类时重写类来获得更快的速度。

#### 记录条件评估中的变化

默认每次自动重启应用程序的时候，都会显示一份对自动配置的变更报告（比如添加或删除bean或者设置配置属性）

禁用报告设置：

```yml
spring.devtools.restart.log-condition-evaluation-delta=false
```

> ⚠️笔者注：
>
> 开启时候的报告示例：
>
> ![image-20230125233334186](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/image-20230125233334186.png)

#### 排除资源

某些资源在更改时并不会触发自动重启，默认情况下更改` /META-INF/maven`, `/META-INF/resources`, `/resources`, `/static`, `/public`,  `/templates`目录下的资源不会触发重启但是会触发[实时加载](#4.8.4 实时加载)，如果要自定义这些排除项，可以使用`spring.devtools.restart.exclude`属性，比如仅排除`/static`和`/public`目录：

```yml
spring.devtools.restart.exclude=static/**,public/**
```

如果要保留默认的配置，并且添加新的排除项，使用`spring.devtools.restart.additional-exclude`。



#### 监听其他路径文件

如果要监听不在类路径中的文件时，使用`spring.devtools.restart.additional-paths`属性。另外可以配合`spring.devtools.restart.exclude`来设置其他路径下的文件变更是触发重启还是实时加载。

#### 禁用重启

使用`spring.devtools.restart.enabled`禁用重启，如果在`application.properties`配置，重启类加载器还是会初始化，只是不会监听文件的变更，要完全禁用需要设置系统变量`spring.devtools.restart.enabled`为`false`，如下：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        System.setProperty("spring.devtools.restart.enabled", "false");
        SpringApplication.run(MyApplication.class, args);
    }

}
```

#### 使用触发式文件

使用某个指定的文件变更来触发自动重启，使用`spring.devtools.restart.trigger-file`配置指定文件（不包括路径），该文件必须在类路径下。

比如：有这样一个结构的项目：

```
src 
+- main 
   +- resources 
      +- .reloadtrigger
```

那么trigger-file的配置是

```
spring.devtools.restart.trigger-file=.reloadtrigger
```



#### 自定义重启类加载器

默认情况下，IDE中打开的项目都使用重启类加载器，其他.jar文件使用基类加载器。使用`mvn spring-boot:run`或者`gradle bootRun`也是这样。

可以通过`META-INF/spring-devtools.properties`文件来自定义，`spring-devtools.properties`文件包含前缀为`restart.exclude`和`restart.include`的属性，`include`属性被重启类加载器加载，`exclude`属性被基类加载器排除，该属性适用类路径的正则表达式，如：

```
restart.exclude.companycommonlibs=/mycorp-common-[\\w\\d-\\.]+\\.jar
restart.include.projectcommon=/mycorp-myproj-[\\w\\d-\\.]+\\.jar
```

> 键必须是唯一的，只要是`restart.exclude`和`restart.include`开头的属性都会被考虑。
>
> `META-INF/spring-devtools.properties`的内容可以打包中项目中，也可以打包到库中。

#### 已知限制

对于使用标准`ObjectInputStream`反序列化的对象，重新启动功能不起作用。如果您需要反序列化数据，则可能需要将Spring的`ConfigurableObjectInputStream`与`Thread.currentThread().getContextClassLoader()`结合使用。

> ⚠️笔者注：
>
> 这个点我觉得略过即可

### 4.8.4 实时加载

`spring-boot-devtools`包含一个嵌入式的LiveReload服务器，可用于资源变更时实时触发浏览器刷新。LiveReload 浏览器扩展可从[livereload.com](http://livereload.com/extensions/)免费获得 Chrome、Firefox 和 Safari 。如果您不想在应用程序运行时启动 LiveReload 服务器，您可以将该`spring.devtools.livereload.enabled`属性设置为`false`.

> 您一次只能运行一个 LiveReload 服务器。在启动您的应用程序之前，请确保没有其他 LiveReload 服务器正在运行。如果您从 IDE 启动多个应用程序，则只有第一个应用程序支持 LiveReload。

> ⚠️笔者注：
>
> 这个点我觉得略过即可，浏览器手动刷新一下也不费事🤪

### 4.8.5 全局设置

可以通过将以下任何文件添加到`$HOME/.config/spring-boot`目录来配置全局 devtools 设置：

1. `spring-boot-devtools.properties`
2. `spring-boot-devtools.yaml`
3. `spring-boot-devtools.yml`

添加到该文件的任何配置都适用于该机器上的所有Spring Boot 应用程序，例如，要将自动重启配置为使用[触发式文件](#使用触发式文件)，可以这样配置：

```
spring.devtools.restart.trigger-file=.reloadtrigger
```

默认情况下，`$HOME`是用户的主目录。要自定义此位置，请设置`SPRING_DEVTOOLS_HOME`环境变量或`spring.devtools.home`系统属性。

> 如果在`$HOME/.config/spring-boot`中找不到 devtools 配置文件，则会在根`$HOME`目录中搜索是否存在`.spring-boot-devtools.properties`文件。这允许您与不支持该`$HOME/.config/spring-boot`位置的旧版本 Spring Boot 上共享 devtools 全局配置。

> 在`.spring-boot-devtools.properties`中的配置都不会影响其他的应用配置文件（如`application-{profile}`之类的文件），并且不支持spring-boot-devtools-<profile>.properties和spring.config.activate.on-profile 之类的配置。

#### 配置文件监听器

[FileSystemWatcher](https://github.com/spring-projects/spring-boot/tree/v2.7.8/spring-boot-project/spring-boot-devtools/src/main/java/org/springframework/boot/devtools/filewatch/FileSystemWatcher.java)通过一定的时间间隔轮询类文件的变更来工作，然后等待预定义的静默期以确保没有更多变更。如果您发现有时候某些更改并没有及时变化，可以尝试修改`spring.devtools.restart.poll-interval`和`spring.devtools.restart.quiet-period`参数。

```
spring.devtools.restart.poll-interval=2s
spring.devtools.restart.quiet-period=1s
```

受监视的类路径目录现在每 2 秒轮询一次更改，并保持 1 秒的静默期以确保没有其他类更改。

### 4.8.6 远程应用

Spring Boot 支持部分远程功能，但有一定安全风险，只能在受信任的网络或SSL保护下运行，并且不能在生产环境上开启该功能。

启用该功能，确保如下配置：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludeDevtools>false</excludeDevtools>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后使用`spring.devtools.remote.secret`设置一个复杂的密码。

> Spring WebFlux 不支持该功能

#### 运行远程客户端应用程序

远程客户端应用程序旨在从IDE中运行。您需要使用与连接到的远程项目相同的类路径运行`org.springframework.boot.devtools.RemoteSpringApplication`。应用程序的单个必需参数是它连接的远程URL。

例如，如果您使用的是Eclipse或STS，并且已经部署到Cloud Foundry的项目名为`my-app`，则可以执行以下操作：

- 从`Run`菜单中选择`Run Configurations…`。
- 创建一个新的`Java Application`“启动配置”。
- 浏览`my-app`项目。
- 使用`org.springframework.boot.devtools.RemoteSpringApplication`作为主类。
- 将`https://myapp.cfapps.io`添加到`Program arguments`（或任何远程URL）。

正在运行的远程客户端可能类似于以下列表：

```
  .   ____          _                                              __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _          ___               _      \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` |        | _ \___ _ __  ___| |_ ___ \ \ \ \
 \\/  ___)| |_)| | | | | || (_| []::::::[]   / -_) '  \/ _ \  _/ -_) ) ) ) )
  '  |____| .__|_| |_|_| |_\__, |        |_|_\___|_|_|_\___/\__\___|/ / / /
 =========|_|==============|___/===================================/_/_/_/
 :: Spring Boot Remote ::  (v2.7.8)

2023-01-19 14:18:32.205  INFO 16947 --- [           main] o.s.b.devtools.RemoteSpringApplication   : Starting RemoteSpringApplication v2.7.8 using Java 1.8.0_362 on myhost with PID 16947 (/Users/myuser/.m2/repository/org/springframework/boot/spring-boot-devtools/2.7.8/spring-boot-devtools-2.7.8.jar started by myuser in /opt/apps/)
2023-01-19 14:18:32.211  INFO 16947 --- [           main] o.s.b.devtools.RemoteSpringApplication   : No active profile set, falling back to 1 default profile: "default"
2023-01-19 14:18:32.566  INFO 16947 --- [           main] o.s.b.d.a.OptionalLiveReloadServer       : LiveReload server is running on port 35729
2023-01-19 14:18:32.584  INFO 16947 --- [           main] o.s.b.devtools.RemoteSpringApplication   : Started RemoteSpringApplication in 0.804 seconds (JVM running for 1.204)
```

> 因为远程客户端使用与真实应用程序相同的类路径，所以它可以直接读取应用程序属性。这是`spring.devtools.remote.secret`属性的读取方式并传递给服务器进行身份验证。

> 始终建议使用`https://`作为连接协议，以便加密连接并且不会截获密码。
>
> 如果需要使用代理来访问远程应用程序，请配置`spring.devtools.remote.proxy.host`和`spring.devtools.remote.proxy.port`属性。

#### 远程更新

远程客户端以与[本地重新启动](#4.8.3 自动重启)相同的方式监视应用程序类路径以进行更改 。任何更新的资源都会被推送到远程应用程序，并且（*如果需要*）会触发重新启动。如果您迭代使用本地没有的云服务的功能，这将非常有用。通常，远程更新和重新启动比完全重建和部署周期快得多。

> 仅在远程客户端运行时监视文件。如果在启动远程客户端之前更改文件，则不会将其推送到远程服务器

> ⚠️笔者注：
>
> 对于目前的大型微服务集群来说，并不实用，而且操作繁琐，使用这种更新方式部分类还有可能不生效，如果只是在测试环境使用，还不如Jenkins重新打包部署

![](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/CSDN-%E5%85%AC%E4%BC%97%E5%8F%B7.png)

## 4.9 打包应用程序

使用Maven 或者Gradle 打包应用程序，生成jar包文件。



# 5. 核心功能

## 5.1 SpringApplication

`SpringApplication`提供了一个`main()`方法方便引导Spring 应用程序启动，并委托给静态方法`SpringApplication.run`。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

启动后，会看到如下信息:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.8)

2023-01-19 14:18:33.375  INFO 17059 --- [           main] o.s.b.d.f.s.MyApplication                : Starting MyApplication using Java 1.8.0_362 on myhost with PID 17059 (/opt/apps/myapp.jar started by myuser in /opt/apps/)
2023-01-19 14:18:33.379  INFO 17059 --- [           main] o.s.b.d.f.s.MyApplication                : No active profile set, falling back to 1 default profile: "default"
2023-01-19 14:18:34.288  INFO 17059 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2023-01-19 14:18:34.301  INFO 17059 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-01-19 14:18:34.301  INFO 17059 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.71]
2023-01-19 14:18:34.371  INFO 17059 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-01-19 14:18:34.371  INFO 17059 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 943 ms
2023-01-19 14:18:34.754  INFO 17059 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2023-01-19 14:18:34.769  INFO 17059 --- [           main] o.s.b.d.f.s.MyApplication                : Started MyApplication in 1.789 seconds (JVM running for 2.169)
```

默认情况下日志级别是`INFO`,如果需要额外的日志级别设置，查看[5.4.5 日志级别](#5.4.5 日志级别)。通过`spring.main.log-startup-info`设置为`false`,可以关闭应用程序的日志记录。

### 5.1.1 启动失败

如果应用启动失败，能够通过已注册的`FailureAnalyzers`获取错误信息以便修复问题。比如应用程序启动的8080端口被占用。

```
***************************
APPLICATION FAILED TO START
***************************

Description:

Embedded servlet container failed to start. Port 8080 was already in use.

Action:

Identify and stop the process that is listening on port 8080 or configure this application to listen on another port.
```

> Spring Boot 支持自定义`FailureAnalyzer实现`

如果没有故障分析器能够处理异常，您需要给`org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener`[启用`debug`属性](#5.2 外部配置)，或者[开启DEBUG日志](#5.4.5 日志级别)。

使用`java -jar`启动应用程序，使用`debug`开启日志：

```
$ java -jar myproject-0.0.1-SNAPSHOT.jar --debug
```

### 5.1.2 惰性初始化

SpringApplication 允许延迟初始化应用程序，当启用惰性初始化时，bean 在需要时创建，而不是在启动期间创建。

惰性初始化的一个缺点是会延迟发现应用程序的问题，如果配置错误的bean被惰性初始化，则在启动期间不会发生故障，只有在bean 被初始化时才发现问题。另外还要注意确保JVM有足够的内存来容纳所有的bean。因此建议在启用惰性初始化前微调JVM堆大小。

使用`SpringApplicationBuilder`的`lazyInitialization`或者`SpringApplication`的`setLazyInitialization`方法开启惰性初始化，也可以使用`spring.main.lazy-initialization`开启。

```
spring.main.lazy-initialization=true
```

> 指定某些bean延迟初始化，使用`@Lazy(false)`

### 5.1.3 自定义横幅

通过将`banner.txt`添加到类路径中，或者设置`spring.banner.location`为该类文件的位置，来更改应用启动时打印的横幅。

如果文件编码不是UTF-8，可以设置`spring.banner.charset`。

除了使用文本文件外，还可以使用图片，将图片添加到类路径中，或者设置`spring.banner.image.location`，图形将被转换为ASCII格式。

在`banner.txt`文件中，您可以使用`Environment`中可用的任何键和以下占位符。

| 占位符                                                       | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `${application.version}`                                     | 您的应用程序的版本号，如在`MANIFEST.MF`声明的那样。例如，`Implementation-Version: 1.0`打印为`1.0`. |
| `${application.formatted-version}`                           | 您的应用程序的版本号，在`MANIFEST.MF`中声明并格式化显示（用括号括起来并以 为前缀`v`）。例如`(v1.0)`。 |
| `${spring-boot.version}`                                     | 您正在使用的 Spring Boot 版本。例如`2.7.8`。                 |
| `${spring-boot.formatted-version}`                           | 您正在使用的 Spring Boot 版本，经过格式化以供显示（用方括号括起来并以 为前缀`v`）。例如`(v2.7.8)`。 |
| `${Ansi.NAME}`（或`${AnsiColor.NAME}`，，`${AnsiBackground.NAME}`）`${AnsiStyle.NAME}` | `NAME`ANSI 转义代码的名称在哪里。详情请见[`AnsiPropertySource`](https://github.com/spring-projects/spring-boot/tree/v2.7.8/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/ansi/AnsiPropertySource.java)。 |
| `${application.title}`                                       | 您的应用程序的标题，如`MANIFEST.MF`中声明的那样。例如`Implementation-Title: MyApp`打印为`MyApp`. |

> 使用`SpringApplication.setBanner(…)`以编程方式设置横幅，使用`org.springframework.boot.Banner`接口并实现`printBanner()`方法自定义打印横幅。

可以使用`spring.main.banner-mode` 设置是否在`System.out`( `console`)、或者日志文件中打印横幅、或者不打印横幅

> `${application.version}`和`${application.formatted-version}`配置仅仅在使用Spring Boot启动器的时候可用。如果你使用未打包的jar并使用`java -cp <classpath> <mainclass>`启动，则不会生效。
>
> 这就是为什么我们建议您始终使用java org.springframework.boot.loader.JarLauncher启动未打包的jar。这将在构建类路径和启动应用程序之前初始化`application.*banner`变量。

### 5.1.4 自定义SpringApplication

如果默认的SpringApplication 不适合您，您可以自己创建一个实例，并进行自定义。例如，要关闭横幅：

```java
import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setBannerMode(Banner.Mode.OFF);
        application.run(args);
    }

}
```

也可以使用`application.properties`配置`SpringApplication`，详细查看[外部配置](#5.2 外部配置)

### 5.1.5 流式API生成器

如果您需要构建`ApplicationContext`层次结构（具有父子关系的多个上下文）或者更喜欢使用流式API构建器，可以使用`SpringApplicationBuilder`。

`SpringApplicationBuilder`让你将多个方法链式调用，包括`parent`和`child`方法创建层级结构，比如：

```java
new SpringApplicationBuilder()
        .sources(Parent.class)
        .child(Application.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
```

`ApplicationContext`创建层次结构 时有一些限制。例如，Web 组件**必须**包含在子上下文中，并且同样`Environment`用于父上下文和子上下文。有关详细信息，请参阅[`SpringApplicationBuilder`Javadoc](https://docs.spring.io/spring-boot/docs/2.7.8/api/org/springframework/boot/builder/SpringApplicationBuilder.html)。

### 5.1.6 应用可用性

当部署在平台上时，应用程序可以使用[Kubernetes Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)等基础设施向平台提供有关其可用性的信息。

Spring Boot 对常用的“liveness” 和 “readiness”状态提供开箱即用的支持。如果您使用了Spring Boot 的`actuator`那么状态将以监控端点的形式提供。

另外，您还可以通过`ApplicationAvailability`接口将可用性状态注入到自己的bean中。

#### Liveness 状态

应用程序的"Liveness"状态表示其是否能正常工作，或者当前是失败状态，则自行修复。中断的“Liveness”状态意味着应用程序处于无法恢复的状态，那么基础架构应重启应用程序。

> “Liveness”状态不应该基于外部检查，比如健康检查。如果这样，一个失败的外部信息（如数据库、外部缓存等）将导致大规模重启和整个平台的连锁故障。

Spring Boot 应用程序的内部状态主要根据Spring  的 `ApplicationContext`。如果应用程序上下文成功启动，则Spring Boot 会认为应用程序处于有效状态，上下文刷新的话，应用程序被认为处于活跃，更多参考[5.1.7 应用程序事件和监听器](#5.1.7 应用程序事件和监听器)

#### Readiness 状态

“Readiness”状态表示应用程序是否已经准备好处理请求。失败的“Readiness”状态表示现在不应该接收流量。这通常发生在启动期间，同时处理`CommandLineRunner`和`ApplicationRunner`组件，或者在应用程序认为太忙的时候发生。一旦应用程序和使用命令行调用应用程序被调用，就被认为是“Readiness 状态”。

> 预期在启动期间运行的任务应该由组件`CommandLineRunner`和`ApplicationRunner`执行，不是使用 Spring 组件生命周期回调，例如`@PostConstruct`

#### 管理应用程序可用性状态

应用程序可用随时通过注入`ApplicationAvailability` 接口并调用其上的方法来获取其可用性状态。还有的情况是，应用程序希望监听状态更新或者更新应用程序的状态。

例如，我们可以将应用程序的“Readiness”状态导出到一个文件中，以便Kubernetes“exec Probe”可以查看该文件：

```java
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class MyReadinessStateExporter {

    @EventListener
    public void onStateChange(AvailabilityChangeEvent<ReadinessState> event) {
        switch (event.getState()) {
            case ACCEPTING_TRAFFIC:
                // create file /tmp/healthy
                break;
            case REFUSING_TRAFFIC:
                // remove file /tmp/healthy
                break;
        }
    }

}
```

当应用程序中断并且不能恢复的时候，还可以更新这个应用程序的状态。

```java
import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Component
public class MyLocalCacheVerifier {

    private final ApplicationEventPublisher eventPublisher;

    public MyLocalCacheVerifier(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void checkLocalCache() {
        try {
            // ...
        }
        catch (CacheCompletelyBrokenException ex) {
            AvailabilityChangeEvent.publish(this.eventPublisher, ex, LivenessState.BROKEN);
        }
    }

}
```

Spring Boot 提供[Kubernetes HTTP接口](#11.2.9 Kubernetes Probes)，用于Actuator 健康端点的"Liveness" and "Readiness"状态，你能够获取更多指导关于[在Kubernetes上部署应用程序](#12.1.2 Kubernetes)。

### 5.1.7 应用程序事件和监听器

除了Spring Framework事件之外，比如[`ContextRefreshedEvent`](https://docs.spring.io/spring-framework/docs/5.3.25/javadoc-api/org/springframework/context/event/ContextRefreshedEvent.html)，SpringApplication 还会发送一些额外的事件。

> 有些事件实际上是在创建`ApplicationContext`创建之前，因此你不能作为`@Bean`注册监听器。你能够通过`SpringApplication.addListeners(…)`方法或者`SpringApplicationBuilder.listeners(…)`注册。
>
> 如果希望自动注册这些监听器，可以将监听器添加到`META-INF/spring.factories`中，使用`org.springframework.context.ApplicationListener`做为key。

运行应用程序的时候，按以下顺序发送事件：

1. `ApplicationStartingEvent` 在应用程序开始运行时发送（任何处理之前），除了监听器和初始化程序的注册
2. `ApplicationEnvironmentPreparedEvent`发送，当上下文中要使用的已知`Environment`时但在创建上下文之前。
3. `ApplicationContextInitializedEvent`发送，在准备了`ApplicationContext`并且调用了`ApplicationContextInitializers`后，但在加载任何bean之前
4. `ApplicationPreparedEvent`在刷新开始之前，但在加载Bean定义后发送
5. `ApplicationStartedEvent`在刷新上下文之后，但在任何应用程序和命令行程序被调用之前发送
6. `AvailabilityChangeEvent`在表示应用程序状态为`LivenessState.CORRECT`时发送
7. `ApplicationReadyEvent`在任何应用程序和命令行程序被调用之后发送
8. `AvailabilityChangeEvent` 在表示应用程序已经做好接收请求准备时发送，状态为`ReadinessState.ACCEPTING_TRAFFIC`
9. `ApplicationFailedEvent`在启动异常时发送

上面的列表只包括与`SpringApplication`相关的`SpringApplicationEvent`事件。以下事件也在`ApplicationPreparedEvent`之后和`ApplicationStartedEvent`之前发送。

- `WebServerInitializedEvent`在`WebServer`准备好后发送，`ServletWebServerInitializedEvent`和`ReactiveWebServerInitializedEvent`分别是servlet 和 reactive的变体
- `ContextRefreshedEvent`在`ApplicationContext`刷新后发送

> 事件监听器不应该运行冗长的任务，因为他们默认在同一线程中运行

应用程序事件使用Spring Framework的事件发布机制发送。此机制的一部分确保在子上下文中发布给监听器的事件也会在任何祖先上下文中发布给监听器。因此，如果您的应用程序使用`SpringApplication`实例的层次结构，则监听器可能会收到相同类型的应用程序事件的多个实例。

为了允许监听器区分其上下文的事件和后代上下文的事件，它应该请求注入其应用程序上下文，然后将注入的上下文与事件的上下文进行比较。可以通过实现`ApplicationContextAware`或者如果监听器是bean，使用`@Autowired`来注入上下文。

### 5.1.8 Web环境

SpringApplication会试图创建正确类型的ApplicationContext。用于确定WebApplicationType的算法如下：

- 如果存在 Spring MVC，使用`AnnotationConfigServletWebServerApplicationContext`
- 如果 Spring MVC 不存在而 Spring WebFlux 存在，使用`AnnotationConfigReactiveWebServerApplicationContext`
- 否则，使用`AnnotationConfigApplicationContext`

这意味着如果您`WebClient`在同一应用程序中使用 Spring MVC 和 Spring WebFlux ，则默认情况下将使用 Spring MVC。您可以通过调用`setWebApplicationType(WebApplicationType)`来覆盖。

也可以完全控制`ApplicationContext`调用所使用的类型`setApplicationContextClass(…)`。

>  在JUnit测试中使用`SpringApplication`时，通常需要调用`setWebApplicationType(WebApplicationType.NONE)`

### 5.1.9 访问应用程序参数

如果您需要访问传递给`SpringApplication.run(…)`的应用程序参数，则可以注入`org.springframework.boot.ApplicationArguments` bean。`ApplicationArguments`接口提供对原始`String[]`参数以及解析的`option`和`non-option`参数的访问，如以下示例所示：

```java
import java.util.List;

import org.springframework.boot.ApplicationArguments;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        if (debug) {
            System.out.println(files);
        }
        // if run with "--debug logfile.txt" prints ["logfile.txt"]
    }

}
```

>  Spring Boot还注册`CommandLinePropertySource`和Spring `Environment`。这使您还可以使用`@Value`注释注入单个应用程序参数。

### 5.1.10 使用ApplicationRunner 或 CommandLineRunner

如果您需要在启动后运行一些特定的代码`SpringApplication`，您可以实现`ApplicationRunner`或`CommandLineRunner`接口。这两个接口以相同的方式工作，并提供一个`run`方法，该方法在`SpringApplication.run(…)`完成之前被调用。

> 非常适合在应用程序启动后但在接受请求之前运行的任务

这些`CommandLineRunner`接口提供字符串数组用于访问对应用程序参数，而`ApplicationRunner`使用`ApplicationArguments`。以下示例显示了`CommandLineRunner`一个`run`方法：

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        // Do something...
    }

}
```

如果CommandLineRunner或ApplicationRunner bean必须按顺序调用，可以实现`org.springframework.core.Ordered`接口或者使用`org.springframework.core.annotation.Order`注解，

### 5.1.11 应用程序退出

每个都`SpringApplication`向 JVM 注册一个关闭钩子，以确保`ApplicationContext`在退出时正常关闭。可以使用所有标准的 Spring 生命周期回调（例如`DisposableBean`接口或`@PreDestroy`注释）。

此外，如果希望在`SpringApplication.exit()`被调用时返回特定的退出代码，则可以实现该接口`org.springframework.boot.ExitCodeGenerator`，然后可以将此退出代码传递给`System.exit()`其将作为状态码返回，如以下示例所示：

```java
import org.springframework.boot.ExitCodeGenerator;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class MyApplication {

    @Bean
    public ExitCodeGenerator exitCodeGenerator() {
        return () -> 42;
    }

    public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(MyApplication.class, args)));
    }

}
```

此外，`ExitCodeGenerator`接口可以通过异常来实现。遇到此类异常时，Spring Boot 会返回已实现`getExitCode()`方法提供的退出代码。

如果存在多个`ExitCodeGenerator`，则使用生成的第一个非零退出代码。要控制调用生成器的顺序，请另外实现`org.springframework.core.Ordered`接口或使用`org.springfframework.core.annotation.order`注解。

### 5.1.12 管理员功能

可以通过指定`spring.application.admin.enabled`属性为应用程序启用与管理相关的功能。这暴露了[`SpringApplicationAdminMXBean`](https://github.com/spring-projects/spring-boot/tree/v2.7.8/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/admin/SpringApplicationAdminMXBean.java)平台上的`MBeanServer`。您可以使用此功能远程管理您的 Spring Boot 应用程序。此功能也可用于任何服务包装器实现。

> 如果您想知道应用程序在哪个 HTTP 端口上运行，获取`local.server.port`属性的值。

### 5.1.13 应用程序启动跟踪

在应用程序启动期间，`SpringApplication`执行`ApplicationContext`许多与应用程序生命周期、bean 生命周期甚至处理应用程序事件相关的任务。有了[`ApplicationStartup`](https://docs.spring.io/spring-framework/docs/5.3.25/javadoc-api/org/springframework/core/metrics/ApplicationStartup.html)Spring Framework [，您就可以使用`StartupStep`对象](https://docs.spring.io/spring-framework/docs/5.3.25/reference/html/core.html#context-functionality-startup)跟踪应用程序启动顺序。可以收集这些数据用于分析目的，或者只是为了更好地了解应用程序启动过程。

可以使用`setApplicationStartup`设置一个实现`ApplicationStartup`的实例，比如，使用`BufferingApplicationStartup`的示例：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.metrics.buffering.BufferingApplicationStartup;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setApplicationStartup(new BufferingApplicationStartup(2048));
        application.run(args);
    }

}
```

第一个可用的实现`FlightRecorderApplicationStartup`由 Spring Framework 提供。它将特定于 Spring 的启动事件添加到 Java Flight Recorder 会话，旨在分析应用程序并将其 Spring 上下文生命周期与 JVM 事件（例如分配、GC、类加载……）相关联。配置完成后，您可以在启用飞行记录器的情况下运行应用程序来记录数据：

```
$ java -XX:StartFlightRecording:filename=recording.jfr,duration=10s -jar demo.jar
```

Spring Boot 附带`BufferingApplicationStartup`变体；此实现旨在缓冲启动步骤并将它们排入外部指标系统。`BufferingApplicationStartup`应用程序可以在任何组件中请求类型的 bean 。

Spring Boot 还可以配置为公开一个以 JSON 文档形式提供此信息的[`startup`端点。](https://docs.spring.io/spring-boot/docs/2.7.8/actuator-api/htmlsingle/#startup)

![](https://itsaysay-1313174343.cos.ap-shanghai.myqcloud.com/blog/CSDN-%E5%85%AC%E4%BC%97%E5%8F%B7.png)

## 5.2 外部化配置

Spring Boot 允许您外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。您可以使用各种外部配置源，包括 Java 属性文件、YAML 文件、环境变量和命令行参数。

属性值可以通过注解直接注入 bean `@Value`，通过 Spring 的[抽象](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.typesafe-configuration-properties)`Environment`访问，或者通过`@ConfigurationProperties`绑定到对象。

Spring Boot 使用一种非常特殊的`PropertySource`顺序，旨在允许合理地覆盖值。后面的属性源可以覆盖前面定义的值。来源按以下顺序考虑：

1. 默认properties，由`SpringApplication.setDefaultProperties`指定
2. 在`@Configuration`上的[`@PropertySource`](https://docs.spring.io/spring-framework/docs/5.3.25/javadoc-api/org/springframework/context/annotation/PropertySource.html)注解，请注意，在刷新应用程序上下文之前，这些属性源不会添加到`Environment`中。而`logging.*` 和 `spring.main.*` 是在应用程序上下文刷新之前读取。
3. Config 数据，比如`application.properties`
4. `RandomValuePropertySource`中仅为`random.*`的配置
5. 操作系统环境变量
6. Java系统配置，System.getProperties()
7. 来自`java:comp/env`的JNDI属性
8. `ServletContext`初始化参数
9. `ServletConfig`初始化参数
10. 来自`SPRING_APPLICATION_JSON`的属性，嵌入在环境变量(environment variable )或系统属性(system property)中的内联 JSON
11. 命令行参数
12. 在单元测试中的`properties`，在[`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.7.8/api/org/springframework/boot/test/context/SpringBootTest.html) 和[用于测试应用程序特定部分的测试注解](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.testing.spring-boot-applications.autoconfigured-tests)上有效。
13. 单元测试中的[`@TestPropertySource`](https://docs.spring.io/spring-framework/docs/5.3.25/javadoc-api/org/springframework/test/context/TestPropertySource.html) 
14. 在devtools激活下，`$HOME/.config/spring-boot`目录中的[Devtools](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#using.devtools.globalsettings)全局设置配置

Config 数据的加载按照以下顺序：

- 打包在jar包中的[Application配置](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files)，`application.properties` 和 YAML变体
- 打包在jar包中的`application-{profile}.properties`和 YAML 变体
- jar包外的`application.properties`和 YAML 变体
- jar包外的`application-{profile}.properties`和 YAML 变体

> 建议使用一种配置文件格式，如果同时有properties和yaml，properties优先。

假设您开发了一个`@Component`使用`name`属性的应用程序，如以下示例所示：

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

    @Value("${name}")
    private String name;

    // ...

}
```

在您的应用程序类路径中（例如，在您的 jar 中），您可以有一个`application.properties`文件为`name`. 在新环境中运行时，`application.properties`可以在 jar 之外提供一个文件来覆盖`name`. 对于一次性测试，您可以使用特定的命令行开关（例如，`java -jar app.jar --name="Spring"`）启动。

> env和configprops端点可用于确定属性具有特定值的原因。可以使用这两个端点来诊断预期外的属性值。有关详细信息，请参阅"[Production ready features](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#actuator.endpoints)"部分。

### 5.2.1 访问命令行属性

默认情况下，`SpringApplication`将任何命令行选项参数（即以 `--`开头的参数，例如`--server.port=9000`）转换为`property`并将它们添加到 Spring`Environment`中，命令行属性始终优先于基于文件的属性源。

如果您不想将命令行属性添加到 中`Environment`，您可以使用`SpringApplication.setAddCommandLineProperties(false)`禁用它们。

### 5.2.2 JSON 应用程序属性

环境变量和系统属性通常有限制，这意味着某些属性名称不能使用。为了解决这个问题，Spring Boot 允许您将属性块编码为单个 JSON 结构。

当您的应用程序启动时，任何`spring.application.json`或`SPRING_APPLICATION_JSON`属性将被解析并添加到`Environment`.

例如，`SPRING_APPLICATION_JSON`可以在 UN*X shell 的命令行中将属性作为环境变量提供：

```
$ SPRING_APPLICATION_JSON='{"my":{"name":"test"}}' java -jar myapp.jar
```

在前面的示例中，您最终在 Spring 的`Environment`中获取`my.name=test`。

同样的 JSON 也可以作为系统属性提供：

```
$ java -Dspring.application.json='{"my":{"name":"test"}}' -jar myapp.jar
```

或者您可以使用命令行参数提供 JSON：

```
$ java -jar myapp.jar --spring.application.json='{"my":{"name":"test"}}'
```

如果要部署到经典应用程序服务器，您还可以使用名为`java:comp/env/spring.application.json`的JNDI 变量。

> 虽然JSON中的null将添加到结果属性源中，但PropertySourcesPropertyResolver会将null属性视为缺少的值。这意味着JSON不能用null值覆盖低优先级属性源中的属性。

### 5.2.3 外部应用程序属性

当您的应用程序启动时，Spring Boot 将自动从以下位置查找并加载`application.properties`和`application.yaml`

1. 从classpath
   - classpath根目录
   - classpath 的 /config 包
2. 从当前目录
   - 当前目录
   - 当前目录的config/子目录
   - config/的直接子目录

这个列表按顺序排列（较低的项会覆盖较早的项）。加载的文件会做为`PropertySources`添加到Spring `Environment`中。

如果您不想用`application`作为配置文件名，您可以通过指定一个`spring.config.name`环境属性来切换到另一个文件名。例如，要查找`myproject.properties`和`myproject.yaml`文件，您可以按如下方式运行您的应用程序：

```
$ java -jar myproject.jar --spring.config.name=myproject
```

您还可以使用`spring.config.location`环境属性来引用显式位置。此属性接受一个或多个要检查的位置的逗号分隔列表。

以下示例显示如何指定两个不同的文件：

```
$ java -jar myproject.jar --spring.config.location=\
    optional:classpath:/default.properties,\
    optional:classpath:/override.properties
```

> `optional`前缀表示，位置是可选的，允许不存在

>`spring.config.name`, `spring.config.location`, 和`spring.config.additional-location`很早就用于确定必须加载哪些文件。它们必须定义为环境属性（通常是操作系统环境变量、系统属性或命令行参数）。

如果`spring.config.location`包含目录(而不是文件)，应该以`/`结尾。在运行时，它们将在加载之前附加从`spring.config.name`生成的名称

> 目录和文件定位也用于 检查 [profile指定文件](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.profile-specific)。例如，如果`spring.config.location`配置为`classpath:myconfig.properties`，`classpath:myconfig-<profile>.properties`的文件也会被加载

在大多数情况下，`spring.config.location`您添加的每个项目都将引用一个文件或目录。位置按照它们被定义的顺序处理，后面的可以覆盖前面的值。

如果您有一个复杂的位置要设置，并且您使用profile指定的配置文件，那么您可能需要提供进一步的提示，以便Spring Boot知道它们应该如何分组。位置组是所有被认为处于同一级别的位置的集合。例如，您可能希望对所有类路径位置进行分组，然后对所有外部位置进行分组。位置组中的项目用`;`分隔，详细查看[Profile Specific Files](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.profile-specific)

使用`spring.config.location`替换默认的位置配置。例如，如果`spring.config.location`设置为`optional:classpath:/custom-config/,optional:file:./custom-config/`，则完整的位置集是：

1. `optional:classpath:custom-config/`
2. `optional:file:./custom-config/`

如果您更喜欢添加其他位置，而不是替换它们，您可以使用`spring.config.additional-location`. 从其他位置加载的属性可以覆盖默认位置中的属性。

例如，如果`spring.config.additional-location`配置了值`optional:classpath:/custom-config/,optional:file:./custom-config/`，则考虑的完整位置集是：

1. `optional:classpath:/;optional:classpath:/config/`
2. `optional:file:./;optional:file:./config/;optional:file:./config/*/`
3. `optional:classpath:custom-config/`
4. `optional:file:./custom-config/`

此搜索顺序使您可以在一个配置文件中指定默认值，然后在另一个配置文件中有选择地覆盖这些值。您可以在默认位置之一的`application.properties`（或您选择的任何其他基本名称`spring.config.name`）中为您的应用程序提供默认值。然后可以在运行时使用位于自定义位置之一的不同文件覆盖这些默认值。

> 如果您使用环境变量而不是系统属性，大多数操作系统不允许使用句点分隔的键名，但您可以使用下划线代替（例如，`SPRING_CONFIG_NAME`代替`spring.config.name`）。有关详细信息，请参阅[从环境变量绑定](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)。

> 如果您的应用程序在 servlet 容器或应用程序服务器中运行，则可以使用 JNDI 属性（在`java:comp/env`中）或 servlet 上下文初始化参数来代替或同时使用环境变量或系统属性。

#### 可选位置

默认情况下，当指定的配置数据位置不存在时，Spring Boot 将抛出`ConfigDataLocationNotFoundException`，并且应用程序将停止。

如果需要指定一个位置，但不是必须存在，使用`optional:`前缀。可以在`spring.config.location`和`spring.config.additional-location`以及`spring.config.import`中声明。

比如，`spring.config.import`属性，值为`optional:file:./myconfig.properties`，在文件不存在的情况下，应用程序也能够启动。

如果你想要忽略所有的`ConfigDataLocationNotFoundExceptions`异常，并且始终允许应用程序继续启动，可以使用`spring.config.on-not-found`配置。或者通过`SpringApplication.setDefaultProperties(…)`或者使用系统/环境变量设置忽略的值。

#### 通配符位置定位

如果一个配置文件位置路径最后包含`*`，则表示其为通配符位置。这在多个配置文件的情况下，非常有用。

比如，有一些Redis配置和Mysql配置，可以想要把这两个配置文件分开，但又在`application.properties`文件中，这样可能会有两个不同的路径，`/config/redis/application.properties` 和`/config/mysql/application.properties`，通过`config/*/`可以将两个配置文件都进行加载。

默认情况下，Spring Boot在默认搜索位置包含`config/*/`，这意味着将搜索jar之外的/config目录的所有子目录。

您可以将通配符与`spring.config.location`和`spring.config.additional-location`一起使用。

> 通配符位置定位只能包含一个`*`，对于搜索目录必须以`*/`结尾，对于搜索文件，则必须以`*/<filename>`结尾。带有通配符的位置根据文件名的绝对路径按字母顺序排序。

> 通配符位置仅适用于外部目录。不能在`classpath:location`中使用通配符。

#### Profile特定文件

除了`application`属性文件之外，Spring Boot还将尝试使用命名约定`application-{profile}`加载profile特定文件。例如，如果应用程序激活名为`prod`的配置文件并使用YAML文件，那么将同时加载`application.yml`和`application-prod.yml`。

Profile特定文件的属性加载与标准应用程序属性加载的位置相同，profile特定文件总是覆盖非特定的文件（application.yml）。如果指定了多个配置文件，则采用最后获胜策略。例如，如果配置文件 `prod` 、`live` 是由`spring.profiles.active`属性指定的，那么`application-prod.properties`中的值可以被`application-live.properties`中的值覆盖。

> 最后获胜策略适用于[位置组](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.location-groups)级别。`spring.config.location`的`classpath:/cfg/,classpath:/ext/`配置和`classpath:/cfg/;classpath:/ext/`配置的覆盖规则不同。
>
> 例如，继续上面的prod、live示例，我们可能有以下文件：
>
> ```
> /cfg
>   application-live.properties
> /ext
>   application-live.properties
>   application-prod.properties
> ```
>
> `spring.config.location`的值为`classpath:/cfg/,classpath:/ext/`,程序会先处理`/cfg`下的所有文件，再处理`/ext`
>
> 1. `/cfg/application-live.properties`
> 2. `/ext/application-prod.properties`
> 3. `/ext/application-live.properties`
>
> 如果值为`classpath:/cfg/;classpath:/ext/`,程序视为同一级别
>
> 1. `/ext/application-prod.properties`
> 2. `/cfg/application-live.properties`
> 3. `/ext/application-live.properties`

`Environment`有一组默认配置文件（默认情况下为[default]），如果未设置活动配置文件，则使用这些配置文件。换句话说，如果没有显式激活配置文件，那么将考虑`application-default`。

> 配置文件只加载一次。如果您已经[直接导入](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.importing)了特定配置文件的属性文件，则不会再次导入该文件。

#### 导入附加数据

应用程序配置可以使用`spring.config.import` 属性从其他位置导入更多配置数据。

例如，classpath `application.properties`文件中可能包含以下内容：

```
spring.application.name=myapp
spring.config.import=optional:file:./dev.properties
```

这将触发当前目录中dev.properties文件的导入（如果存在这样的文件）。导入的`dev.properties`中的值将优先于触发导入的文件。在上面的示例中，`dev.properties`可以将`spring.application.name`重新定义为不同的值。

无论声明多少次，都只能导入一次。在导入properties/yaml的文件中定义的单个文档顺序是无关紧要的，比如，下面的两个例子产生相同的结果。

```
spring.config.import=my.properties
my.property=value
```

```
my.property=value
spring.config.import=my.properties
```

在上述两个示例中，`my.properties`文件中的值将优先于触发其导入的文件。可以在一个`spring.config.import`下指定多个位置，位置将按照定义的顺序进行处理，以后导入的配置优先。

> 适当时，特定配置文件的变体还会导入，上面的示例将导入`my.properties`以及任何`my-<profile>.properties`变体。

> Spring Boot包括可插拔API，允许支持各种不同的位置地址。默认情况下，您可以导入Java配置、YAML和“[配置树](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.configtree)”。
>
> 第三方jar可以提供对其他技术的支持（不要求文件是本地的）。例如，您可以想象配置数据来自Consul、Apache ZooKeeper或Netflix Archaius等外部存储。
>
> 如果要支持自定义位置，请参阅`org.springframework.boot.context.config`包中的`ConfigDataLocationResolver`和`ConfigDataLoader`类。

#### 导入无扩展名文件

某些云平台无法向卷装载的文件添加文件扩展名。要导入这些无扩展名文件，您需要给Spring Boot一个提示，以便它知道如何加载它们。您可以通过在方括号中放置扩展提示来完成此操作。

例如，假设您有一个`/etc/config/myconfig`文件，希望将其作为yaml导入。您可以使用以下命令从`application.properties`导入它：

```
spring.config.import=file:/etc/config/myconfig[.yaml]
```

#### 使用配置树

在云平台（如Kubernetes）上运行应用程序时，通常需要读取平台提供的配置值。出于这种目的使用环境变量并不罕见，但这可能会有缺点，特别是如果值应该保密的话。

作为环境变量的替代方案，许多云平台现在允许您将配置映射到装载的数据卷中。例如，Kubernetes可以卷装载[ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap)和[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。

可以使用两种常见的卷装载模式：

1. 单个文件包含一组完整的属性（通常写为 YAML）
2. 多个文件被写入目录树，文件名成为“key”，内容成为“value”

对于第一种情况，可以[上述配置](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#features.external-config.files.importing)使用`spring.config.import`导入YAML或Properties文件。

对于第二种情况，您需要使用`configtree:`前缀，以便Spring Boot知道它需要将所有文件公开为Properties。

例如，让我们假设Kubernetes安装了以下卷：

```
etc/
  config/
    myapp/
      username
      password
```

`username` 是一个配置的值，`password`是一个加密字符串

要导入这些配置，你可以将如下内容导入`application.properties`或者`application.yaml`

```
spring.config.import=optional:configtree:/etc/config/
```

然后，您可以用通常的方式从`Environment`中访问或注入`myapp.username`和my`app.password`属性。

> 📌 配置树下的文件夹构成属性名称。在上面的示例中，要以`username`和`password`的形式访问属性，可以将`spring.config.import`设置为`optional:configtree:/etc/config/myapp`。

> 带有点符号的文件名也正确映射。例如，在上面的示例中，`/etc/config`中名为`myapp.username`的文件将在`Environment`中生成`myapp.username`属性。

> 📌 配置树值可以绑定到字符串`String`和`byte[]`类型，具体取决于预期的内容。

如果要从同一父文件夹导入多个配置树，则可以使用通配符快捷方式。任何以`/*/`结尾的`configtree:location`都会将所有直接子级作为配置树导入。	

```
etc/
  config/
    dbconfig/
      db/
        username
        password
    mqconfig/
      mq/
        username
        password
```

您可以使用`configtree:/etc/config/*/`作为导入位置：

```
spring.config.import=optional:configtree:/etc/config/*/
```

如上配置将导入 `db.username`, `db.password`, `mq.username` 和 `mq.password` 属性。

> 🚩使用通配符加载的目录按字母顺序排序。如果您需要不同的排序，则应将每个位置列为单独的导入

配置树也可以用于Docker 保密数据。当Docker群服务被授权访问一个保密数据时，该保密数据被装入容器中。例如，如果名为`db.password`的保密数据安装在位置`/run/secrets/`，则可以使用以下变量`db.passwords`在Spring环境中：

```
spring.config.import=optional:configtree:/run/secrets/
```

#### 属性占位符

`application.properties`和`application.yml`中的值在使用时会通过现有的`Environment`进行过滤，因此您可以引用以前定义的值（例如，从系统属性或环境变量）。标准的`${name}`属性占位符语法可以在值的任何位置使用，属性占位符还可以使用`:`指定默认值，将默认值与属性名称分开，例如`${name:default}`。

以下示例显示了带默认值和不带默认值的占位符的使用：

```
app.name=MyApp
app.description=${app.name} is a Spring Boot application written by ${username:Unknown}
```

> 您应该始终使用占位符中的规范形式（kebab-case仅使用小写字母）引用占位符中的属性名称。这将允许Spring Boot使用与`@ConfigurationProperties`相同的宽松绑定逻辑。
>
> 例如，${demo.item-price}将从`application.properties`文件中获取`demo.iterm-price`和`demo.itemPrice`，并从系统环境中获取`DEMO_ITEMPRICE`。如果改用`${demo.itemPrice}`，则不会考虑`demo.item-price`和`DEMO_ITEMPRICE`。

> 🚩您还可以使用此技术创建现有SpringBoot属性的“短”变体。有关详细信息，请参阅使用[“短”命令行参数](https://docs.spring.io/spring-boot/docs/2.7.8/reference/htmlsingle/#howto.properties-and-configuration.short-command-line-arguments)的方法。

#### 使用多文档文件

Spring Boot允许您将单个物理文件拆分为多个逻辑文档，每个逻辑文档都是独立添加的。文档按照从上到下的顺序进行处理。后续文档可以覆盖早期文档中定义的配置。

对于`application.yml`文件，使用标准的YAML多文档语法。三个连续的连字符表示一个文档的结尾和下一个文档开始。

例如，以下包含两个逻辑文档：

```yaml
spring:
  application:
    name: "MyApp"
---
spring:
  application:
    name: "MyCloudApp"
  config:
    activate:
      on-cloud-platform: "kubernetes"
```

`application.properties`文件使用`#---`或者`!---` 来分割文档

```properties
spring.application.name=MyApp
#---
spring.application.name=MyCloudApp
spring.config.activate.on-cloud-platform=kubernetes
```

> 🚩 配置文件分隔符不能有任何前导空格，必须正好有三个连字符。

> 📌  多文档属性文件通常与激活配置（如`spring.config.activate.on-profile`）结合使用。有关详细信息，[请参阅下一节](#激活属性)。
>
> 无法使用@PropertySource或@TestPropertySource注解加载多文档属性文件。

#### 激活属性

您可能具有仅在特定配置文件处于激活状态时才关联配置。

您可以使用`spring.config.activate.*`有条件地激活配置属性。

下面的激活配置可用：

| Property            | Note                                            |
| :------------------ | :---------------------------------------------- |
| `on-profile`        | 必须匹配才能激活文档的配置文件表达式            |
| `on-cloud-platform` | 要使文档处于活动状态，必须检测到“CloudPlatform” |

例如，下面指定第二个文档仅在Kubernetes上运行时有效，并且仅在“prod”或“staging”配置文件处于激活状态时有效：

```properties
myprop=always-set
#---
spring.config.activate.on-cloud-platform=kubernetes
spring.config.activate.on-profile=prod | staging
myotherprop=sometimes-set
```

### 5.2.4 加密属性

Spring Boot不提供任何加密属性的内置支持，但它提供了修改Spring环境中包含值所需的钩子点。`EnvironmentPostProcessor`允许你在应用程序启动的时候控制`Environment`，详细查看[启动时自定义环境变量](#15. "How-to" 指南)。

如果您需要一种安全的方式来存储凭据和密码，[Spring Cloud Vault](https://cloud.spring.io/spring-cloud-vault/)项目将支持在[HashiCorp Vault](https://www.vaultproject.io/)中存储外部化配置。

### 5.2.5 使用YAML文件

[YAML](https://yaml.org/)是JSON的超集，是指定分层配置数据的便捷格式。只要类路径上有SnakeYAML库，`SpringApplication`类就会自动支持YAML作为properties的替代。

#### YAML 映射到Properties

YAML文档需要从其分层格式转换为可用于Spring `Environment`的平面结构。

例如，如下的YAML文档：

```yaml
environments:
  dev:
    url: "https://dev.example.com"
    name: "Developer Setup"
  prod:
    url: "https://another.example.com"
    name: "My Cool App"
```

为了从`Environment`中访问这些属性，它们将按以下方式展平：

```
environments.dev.url=https://dev.example.com
environments.dev.name=Developer Setup
environments.prod.url=https://another.example.com
environments.prod.name=My Cool App
```

同样，YAML列表也需要扁平化，使用`[index]`做为键，比如下面的YAML。

```yaml
my:
 servers:
 - "dev.example.com"
 - "another.example.com"
```

上述示例转为properties后：

```properties
my.servers[0]=dev.example.com
my.servers[1]=another.example.com
```

使用`[index]`表示的properties能够绑定到Java的`List`或`Set`对象。有关更多详细信息，请参阅下面的“[类型安全配置属性](#5.2.8 类型安全配置属性)”部分。

> 无法使用@PropertySource或@TestPropertySource注解加载YAML文件。因此，如果需要以这种方式加载值，则需要使用properties文件。

#### 直接加载YAML

Spring Framework提供了两个方便类，可用于加载YAML文档。`YamlPropertiesFactoryBean`将YAML作为`Properties`加载，`YamlMapFactoryBean`将YAML作为`Map`加载。

如果要将YAML作为Spring `PropertySource`加载，也可以使用`YamlPropertySourceLoader`类。

### 5.2.6 配置随机值

`RandomValuePropertySource`用于注入随机值（例如，注入加密字符或测试用例）。它可以生成integer、long、uuid或string，如下例所示：

```properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number-less-than-ten=${random.int(10)}
my.number-in-range=${random.int[1024,65536]}
```

`random.int*`语法是`OPEN value (,max) CLOSE`，其中OPEN，CLOSE是任何字符，value、max是整数，如果提供了max，则value是最小值，max是最大值（不包括）。

### 5.2.7 配置系统环境属性

Spring Boot支持为环境属性设置前缀。如果系统环境由具有不同配置要求的多个Spring Boot应用程序共享，这将非常有用。系统环境属性的前缀可以直接在SpringApplication上设置。

例如，如果将前缀设置为`input`，则诸如`remote.timeout`之类的属性也将在系统环境中解析为`input.remote.timeout`。

### 5.2.8 类型安全的配置属性

使用`@Value("${property}")`注入配置属性有时会很麻烦，特别是当您有多个属性或数据本质上是分层的时候。SpringBoot提供了另一种使用properties的方法，该方法允许强类型bean管理和验证应用程序的配置。

> 也可以查看[`@Value` 和 type-safe configuration properties](#@ConfigurationProperties vs. @Value)的不同点

#### JavaBean 属性绑定

可以绑定到一个标准的JavaBean，如下例所示：

```java
import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("my.service")
public class MyProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() {
        return this.enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public InetAddress getRemoteAddress() {
        return this.remoteAddress;
    }

    public void setRemoteAddress(InetAddress remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    public Security getSecurity() {
        return this.security;
    }

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() {
            return this.username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getPassword() {
            return this.password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        public List<String> getRoles() {
            return this.roles;
        }

        public void setRoles(List<String> roles) {
            this.roles = roles;
        }

    }

}
```

前面的POJO定义了以下属性：

- my.service.enabled，默认为false
- my.service.remote-address，能够强制转成`String`
- my.service.security.username
- my.service.security.password
- my.service.security.roles String类型的列表，默认是USER

#### 构造函数绑定

#### 启用@ConfigurationProperties注解类型

#### 使用@ConfigurationProperties 注解类型

#### 三方配置

#### 宽松绑定

#### 合并复杂类型

#### 属性转换

#### @ConfigurationProperties 验证

#### @ConfigurationProperties vs. @Value

## 5.3 配置文件

### 5.3.1 添加活动配置文件

### 5.3.2 配置文件组

### 5.3.3 以编程方式设置配置文件

### 5.3.4 指定配置文件

## 5.4 日志

### 5.4.1 日志格式化

### 5.4.2 Console 输出

#### 彩色输出

### 5.4.3 文件输出

### 5.4.4 文件周期

### 5.4.5 日志级别

### 5.4.6 日志组

### 5.4.7 使用日志 ShutDown 钩子

### 5.4.8 自定义日志配置

### 5.4.9 Logback扩展

#### Profile指定配置

#### 环境属性

## 5.5 国际化

## 5.6  JSON

## 5.7 任务执行和调度

## 5.8 测试

### 5.8.1 测试范围依赖

### 5.8.2 测试Spring 应用程序

### 5.8.3 测试Spring Boot 应用程序

#### 检测Web应用程序类型

#### 检测测试配置

#### 排除测试配置

#### 使用应用程序参数

#### 使用Mock环境测试

#### 使用正在运行的服务器测试

#### 自定义WebTestClient

#### 使用JMX

#### 使用Metrics

#### Mocking and Spying Beans

#### Auto-configured 测试

#### Auto-configured JSON 测试

#### Auto-configured Spring MVC 测试

#### Auto-configured Spring WebFlux 测试

#### Auto-configured Spring GraphQL 测试

#### Auto-configured Data Cassandra 测试

#### Auto-configured Data Couchbase 测试

#### Auto-configured Data Elasticsearch 测试

#### Auto-configured Data JPA 测试

#### Auto-configured JDBC 测试

#### Auto-configured Data JDBC 测试

#### Auto-configured jOOQ 测试

#### Auto-configured Data MongoDB 测试

#### Auto-configured Data Neo4j 测试

#### Auto-configured Data Redis 测试

#### Auto-configured Data LDAP 测试

#### Auto-configured REST Clients

#### Auto-configured Spring REST Docs Tests

##### 使用Mock MVC 测试 Auto-configured Spring REST Docs

##### 使用WebTestClient测试Auto-configured Spring REST Docs

##### 使用REST Assured 测试Auto-configured Spring REST Docs

#### Auto-configured Spring Web Services测试

##### Auto-configured Spring Web Services Client 测试

##### Auto-configured Spring Web Services Server 测试

#### 其他 Auto-configuration and 分片

#### 使用 Configuration 和 分片

#### 使用Spock 测试 Spring Boot 应用程序

### 5.8.4 测试实用程序

#### ConfigDataApplicationContextInitializer

#### TestPropertyValues

#### OutputCapture

#### TestRestTemplate

## 5.9 创建自己的自动化配置

### 5.9.1 了解自动配置的Bean

### 5.9.2 查找候选的自动配置

### 5.9.3 条件注解

#### Class条件

#### Bean 条件

#### Property 条件

#### Resource 条件

#### Web Application 条件

#### SpEL 表达式条件

### 5.9.4 测试自动配置

#### 模拟Web上下文

#### 覆盖类路径

### 5.9.5 创建自己的Starter

#### 命名

#### 配置键

#### 自动配置模块

#### Starter模块

## 5.10 Kotlin Support

略

# 6. Web

# 7. Data

# 8. 消息

# 9. IO

# 10. 容器镜像

# 11. 生产就绪功能

## 11.2 端点

### 11.2.9 Kubernetes Probes

# 12. 部署Spring Boot 应用程序

## 12.1 部署到云

### 12.1.2 Kubernetes

# 13. Spring Boot CLI

# 14. 构建工具插件

# 15. "How-to" 指南

# 附录

