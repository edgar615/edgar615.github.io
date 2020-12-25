---
layout: post
title: Spring Boot - 日志
date: 2019-03-22
categories:
    - Spring
comments: true
permalink: spring-logging.html
---

# 1. Spring Logging使用

Spring Boot默认的日志组件为Logback。Spring boot使用Commons Logging来作为internal  logging，但是对底层的log实现是开放的。默认的配置有Java Util Logging, Log4J2, and  Logback。这3种log方式在配置前，默认使用console  output，也可以向文件logging。默认情况下，如果使用starters，将使用logback来logging。

默认情况西，spring boot不会将log message输出到文件，需要指定logging.file或logging.path  property来实现。默认情况下，文件大于10MB就会生成新的log file，只记录ERROR, WARN,  INFO。log文件大小可以使用logging.file.max-size指定，历史文件数使用logging.file.max-history限定。

- **依赖**

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

- **配置参数**

```
logging:
    file:   # 日志文件,绝对路径或相对路径
    path:   # 保存日志文件目录路径
    config: # 日志配置文件,Spring Boot默认使用classpath路径下的日志配置文件,如:logback.xml
    level:  # 日志级别
        org.springframework.web: DEBUG # 配置spring web日志级别
```

Spring Boot中的logging.path和logging.file这2个属性，只需要配置其中之一即可，如果同时配置，则使用logging.file属性。

当配置了loggin.path属性时，将在该路径下生成spring.log文件，即：此时使用默认的日志文件名spring.log

当配置了loggin.file属性时，将在指定路径下生成指定名称的日志文件。默认为项目相对路径，可以为logging.file指定绝对路径。

- **extension**

Spring boot为logback提供了很多extension来帮助advanced configuration。可以在logback-spring.xml文件中使用这些extensions。

注意：由于logback.xml加载太早，不能使用extensions。只能使用logback-spring.xml或定义logging.config property。

<springProperty> tag将Spring environment  property在logback中使用。可用于在logback  configuration中使用application.properties中的值。该tag类似于logback中的<property>,但是能设置更多信息。可以使用source来指定environment中的来源，scope指定存放property的地方，defaultValue来指定默认值，以防environment中没设置。

```
<springProperty scope="context" name="fluentHost" source="myapp.fluentd.host"
defaultValue="localhost"/>
<appender name="FLUENT" class="ch.qos.logback.more.appenders.DataFluentAppender">
<remoteHost>${fluentHost}</remoteHost>
...
</appender>
```

- 默认配置

```
${LOG_FILE}: Whether logging.file was set in Boot’s external configuration.
${LOG_PATH}: Whether logging.path (representing a directory for log files to live in) was set in Boot’s external configuration.
${LOG_EXCEPTION_CONVERSION_WORD}: Whether logging.exception-conversion-word was set in Boot’s external configuration.-->

<!--Configure Logback for File-only Output-->
<include resource="org/springframework/boot/logging/logback/defaults.xml" />
<property name="LOG_FILE" value="${LOG_FILE}"/>
<include resource="org/springframework/boot/logging/logback/file-appender.xml" />
```

# 2. 动态改变日志级别

大部分项目可能都将日志级别设置为info级别，当然也可能有一些追求性能或者说包含很多敏感信息的项目直接将级别设置为warn或者error；这时候如果项目中出现一些未知异常，需要用到很详细的日志信息，此时如果项目中没有动态改变日志级别的机制，排查问题将很棘手。

我们常用的一些日志系统包括：`Log4j2`、`Logback`、`Java Util Logging`；我们想动态改变日志的级别，前提是这些日志系统都支持我们直接设置日志等级，当然这些系统提供了很简单的接口；

- Log4j2

```
LoggerContext loggerContext = (LoggerContext) LogManager.getContext(false);
LoggerConfig loggerConfig = loggerContext.getConfiguration().getLoggers().get("root");
loggerConfig.setLevel(level);
```

- Logback

```
LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
Logger logger = loggerContext.getLogger("root");
((ch.qos.logback.classic.Logger) logger).setLevel(level);
```

- Java Util Logging

```
Logger logger = Logger.getLogger("root");
logger.setLevel(level);
```

除了上面直接设置日志级别的方式，也有可以动态加载配置文件的方式，同样也可以在配置文件中动态改变日志级别，以logback为例：

```
LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
File externalConfigFile = new File("logback.xml");
JoranConfigurator configurator = new JoranConfigurator();
configurator.setContext(lc);
lc.reset();            
configurator.doConfigure(externalConfigFileLocation);
```

知道了日志系统都是如何去设置日志级别的，动态改变日志级别就很简单了，我们可以通过rest接口、配置中心、消息通知等方式实现修改日志级别

```
  @GetMapping(value = "logLevel/{logLevel}")
  public String changeLogLevel(@PathVariable("logLevel") String logLevel) {
    try {
      LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
      Logger logger = loggerContext.getLogger("root");
      ((ch.qos.logback.classic.Logger) logger).setLevel(Level.valueOf(logLevel));
    } catch (Exception e) {
      logger.error("changeLogLevel error", e);
      return "fail";
    }
    return "success";
  }
```

# 3. Spring Actuator logger

`spring-boot-starter-actuator`组件已经提供了动态修改日志级别的功能

添加配置

```
management:
    endpoints:
        web:
            exposure:
                include: loggers
```

启动服务可以通过`/actuator/loggers`查看当前项目每个包的日志级别：

```
{
    "levels": [
        "OFF", 
        "ERROR", 
        "WARN", 
        "INFO", 
        "DEBUG", 
        "TRACE"
    ], 
    "loggers": {
        "ROOT": {
            "configuredLevel": "INFO", 
            "effectiveLevel": "INFO"
        }, 
        "com": {
            "configuredLevel": null, 
            "effectiveLevel": "INFO"
        }, 
        "com.github": {
            "configuredLevel": null, 
            "effectiveLevel": "INFO"
        }, 
        "com.github.edgar615": {
            "configuredLevel": null, 
            "effectiveLevel": "INFO"
        },
}
```

可以通过发送Post请求到`/actuator/loggers/[包路径]`修改日志级别

```
$ curl -s -X POST -d '{"configuredLevel" : "WARN"}' -H 'content-type:application/json' http://localhost:9000/actuator/loggers/com.github.edgar615
```

# 4. 参考资料

https://mp.weixin.qq.com/s/9eY2R7vXXE0KJBb_jJAJZA