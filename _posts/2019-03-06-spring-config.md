---
layout: post
title: Spring Boot - 配置体系
date: 2019-03-06
categories:
    - Spring
comments: true
permalink: spring-config.html
---
在 Spring Boot 中，其核心设计理念是对配置信息的管理采用约定优于配置。在这一理念下，则意味着开发人员所需要设置的配置信息数量比使用传统 Spring 框架时还大大减少。

# 1. 配置文件与 Profile

Profile 本质上代表一种用于组织配置信息的维度，在不同场景下可以代表不同的含义。例如，如果 Profile 代表的是一种状态，我们可以使用 open、halfopen、close 等值来分别代表全开、半开和关闭等。再比如系统需要设置一系列的模板，每个模板中保存着一系列配置项，那么也可以针对这些模板分别创建 Profile。这里的状态或模版的定义完全由开发人员自主设计，我们可以根据需要自定义各种 Profile，这就是 Profile 的基本含义。

另一方面，为了达到集中化管理的目的，Spring Boot 对配置文件的命名也做了一定的约定，分别使用 **label** 和 **profile** 概念来指定配置信息的版本以及运行环境，其中 label 表示配置版本控制信息，而 profile 则用来指定该配置文件所对应的环境。在 Spring Boot 中，配置文件同时支持 .properties 和 .yml 两种文件格式，结合 label 和 profile 概念，如下所示的配置文件命名都是常见和合法的：

```
/{application}.yml
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

显然，类似这样的数据源通常会根据环境的不同而有很多套配置。假设我们存在如下所示的配置文件集合，注意这里有一个全局的 application.yml 配置文件以及多个局部的 Profile 配置文件。

```
application.yml
application-uat.yml
application-test.yml
application-prod.yml
```

在 Spring Boot 中，我们可以在主 application.properties 中使用如下的配置方式来激活当前所使用的 Profile：

```
spring.profiles.active = test
```

上述配置项意味着系统当前会读取 application-test.yml 配置文件中的配置内容。同样，如果使用 .yml 文件，则可以使用如下所示的配置方法：

```
spring:
  profiles:
    active: test
```

事实上，我们也可以同时激活几个 Profile，这完全取决于你对系统配置的需求和维度。

```
spring.profiles.active: prod, myprofile1, myprofile2
```

当然，如果你想把所有的 Profile 配置信息只保存在一个文件中而不是分散在多个配置文件中， Spring Boot 也是支持的，需要做的事情只是对这些信息按 Profile 进行组织、分段，如下所示：

```
spring: 
	profiles: test
	#test 环境相关配置信息
spring: 
	profiles: prod
	#prod 环境相关配置信息
```

推荐按多个配置文件的组织方法管理各个 Profile 配置信息，这样不容易混淆和出错。

最后，如果我们不希望在全局配置文件中静态指定所需要激活的 Profile，而是想把这个过程延迟到运行这个服务时，那么可以直接在 java –jar 命令中添加`--spring.profiles.active`参数，如下所示

```
java –jar customerservice-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod
```

# 2. 代码控制与Profile

在 Spring Boot 中，Profile 这一概念的应用场景还包括动态控制代码执行流程。为此，我们需要使用 `@Profile `注解，先来看一个简单的示例。

```
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        //创建 dev 环境下的 DataSource 
    }

    @Bean()
    @Profile("prod")
    public DataSource prodDataSource(){
        //创建 prod 环境下的 DataSource 
    }

}
```

可以看到，我们构建了一个 `DataSourceConfig` 配置类来专门管理各个环境所需的 DataSource。注意到这里使用 `@Profile` 注解来指定具体所需要执行的 DataSource 创建代码，通过这种方式，可以达到与使用配置文件相同的效果。

更进一步，能够在代码中控制 JavaBean  的创建过程为我们根据各种条件动态执行代码流程提供了更大的可能性。例如，在日常开发过程中，一个常见的需求是根据不同的运行环境初始化数据，常见的做法是独立执行一段代码或脚本。基于 `@Profile` 注解，我们就可以将这一过程包含在代码中并做到自动化，如下所示：

```
@Profile("dev")
@Configuration
public class DevDataInitConfig {

@Bean
  public CommandLineRunner dataInit() { 
    return new CommandLineRunner() {

      @Override
      public void run(String... args) throws Exception {
        //执行 Dev 环境的数据初始化
    };  

}
```

`@Profile` 注解的应用范围很广，我们可以将它添加到包含 `@Configuration` 和 `@Component`   注解的类及其方法，也就是说可以延伸到继承了 `@Component` 注解的 `@Service`、`@Controller`、`@Repository`  等各种注解中。

# 3. 配置文件的加载顺序

我们可以把配置文件保存在多个路径，而这些路径在加载配置文件时具有一定的顺序。Spring Boot 在启动时会扫描以下位置的 application.properties 或者 application.yml 文件作为全局配置文件：

```
# 优先级由高到低
–file:./config/
–file:./
–classpath:/config/
–classpath:/
```

Spring Boot 会全部扫描这四个位置，扫描规则是高优先级配置内容会覆盖低优先级配置内容。而如果高优先级的配置文件中存在与低优先级配置文件不冲突的属性，则会形成一种互补配置，也就是说会整合所有不冲突的属性。

我们也可以通过 `spring.config.location `指定多个配置文件路径

```
--spring.config.location=file:///D:/application.properties, classpath:/config/
```