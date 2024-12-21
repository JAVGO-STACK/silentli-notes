# SpringBoot - 日志管理

在现代 Java 应用中，日志记录是至关重要的一环。与此同时，市面上存在多个日志框架（如 Logback、Log4j2、Java Util Logging 等）。为了避免在代码中直接使用某个特定日志实现（例如直接调用 Log4j2 或 Logback 的 API），业界普遍推荐通过 SLF4J（Simple Logging Facade for Java）来统一日志接口，这样开发者的代码只需面向 SLF4J，而底层真正的日志实现（Log4j2、Logback、Log4j等）可以在无须改变应用代码的前提下灵活切换。

> 本文我们将详细阐述在 Spring Boot 3.x 项目中以 SLF4J 为上层接口、Log4j2 为下层实现的最佳实践与配置方案。

## 为什么使用 SLF4J？

* **统一接口**：SLF4J 是一个日志门面，使开发者在应用中只依赖 SLF4J 的接口，而无需绑定到某个具体实现。
* **可插拔性**：无论底层使用 Log4j2、Logback 还是其他框架，都可以透明替换，不需要修改应用代码。
* **行业标准**：SLF4J 作为日志门面早已成为业界标准实践，提高了团队和项目的可维护性。

## 为何选择 Log4j2 作为底层实现？

* **高性能**：Log4j2 提供异步日志等特性，在高并发场景下表现出色。
* **灵活性强**：Log4j2 拥有丰富的配置策略（时间滚动、文件大小滚动、归档、压缩、过滤等），可轻松应对生产环境中的日志管理需求。
* **丰富的扩展**：内置与多种格式（JSON、XML）、Appender（Console、File、Async、Socket）和布局支持，以便日志聚合与分析。

## 基础配置步骤

### 1.准备与移除默认日志框架

Spring Boot 在默认情况下使用 `spring-boot-starter-logging`（Logback）。要换成 Log4j2，需要先将其排除。

在 `pom.xml` 中进行如下调整：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
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
```

以上配置将默认的 Logging Starter 排除，并引入 `spring-boot-starter-log4j2`，从而使用 Log4j2 作为日志实现框架。

### 2.在代码中统一使用 SLF4J 接口

在应用代码中，不直接调用 `Log4j2` 或 `Logback` 的 `Logger`，而是通过 SLF4J 的 `Logger` 和 `LoggerFactory`：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ExampleService {
    private static final Logger logger = LoggerFactory.getLogger(ExampleService.class);

    public void doSomething() {
        logger.info("This is an info message");
        logger.error("This is an error message");
    }
}
```

通过这种方式，代码层面上永远只依赖 SLF4J，底层实现可以无缝变更而无需改动业务代码。

### 3.放置 Log4j2 的配置文件

将 `log4j2-spring.xml` 放置在 `src/main/resources` 下。当 Spring Boot 启动时，会自动加载该配置文件。

> 为什么是 `log4j2-spring.xml` 而不是 `log4j2.xml`？
>
> Spring Boot 建议使用 `-spring` 后缀的文件名（如 `log4j2-spring.xml`），这样 Spring 框架可以更好地参与日志配置的加载过程，尤其在某些高级场景下（如基于 Profile 的配置）。

以下是一个示例的 `log4j2-spring.xml` 配置文件，该配置体现以下特性：

* 统一使用 SLF4J 接口，在底层由 Log4j2 实现日志输出。
* 控制台（Console）输出使用颜色高亮，不仅便于开发阶段快速阅读日志，也能在测试与运维阶段提高可读性（前提是终端支持）。
* 基于时间滚动（RollingFile）策略，将日志按天切分并自动归档清理，确保生产环境下日志不会无限增长。
* 针对不同包、类灵活控制日志级别，满足精细化调优需求。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="60">

    <Properties>
        <!-- 日志文件的基础路径 -->
        <Property name="LOG_PATH">logs/app</Property>
        <!-- 日志输出格式: 包含时间、线程、日志级别、调用位置、消息内容 -->
        <Property name="LOG_PATTERN">%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - [%method:%line] - %msg%n</Property>
    </Properties>

    <Appenders>
        <!-- 控制台输出（带颜色高亮） -->
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%highlight{%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n}{FATAL=red bright,BOLDERROR=red,ERROR=red,BOLDWARN=yellow bold,WARN=yellow,BOLDINFO=green bold,INFO=green,BOLDDEBUG=cyan bold,DEBUG=cyan,BOLDTRACE=black bold,TRACE=black}"/>
        </Console>

        <!-- 基于时间滚动的INFO及以上级别日志文件 -->
        <RollingFile name="InfoFile" fileName="${LOG_PATH}/info.log" 
                     filePattern="${LOG_PATH}/info-%d{yyyy-MM-dd}.log.gz">
            <Filters>
                <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
            </Policies>
            <!-- 保留30天日志文件 -->
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- 基于时间滚动的ERROR级别日志文件 -->
        <RollingFile name="ErrorFile" fileName="${LOG_PATH}/error.log"
                     filePattern="${LOG_PATH}/error-%d{yyyy-MM-dd}.log.gz">
            <Filters>
                <ThresholdFilter level="error" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy />
            </Policies>
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>
    </Appenders>

    <Loggers>
        <!-- 针对特定包调整日志级别 -->
        <Logger name="com.erow.common.dao" level="info" additivity="false">
            <AppenderRef ref="InfoFile"/>
            <AppenderRef ref="Console"/>
        </Logger>

        <!-- 对Spring框架日志级别进行控制,减少繁杂日志 -->
        <Logger name="org.springframework" level="warn" additivity="false">
            <AppenderRef ref="InfoFile"/>
        </Logger>

        <!-- Root Logger：默认输出到控制台和文件 -->
        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="InfoFile"/>
            <AppenderRef ref="ErrorFile"/>
        </Root>
    </Loggers>

</Configuration>
```

## 环境验证与调试

1. **启动应用**后查看控制台日志：是否已按照指定的格式和颜色输出日志？
2. **检查日志文件**：查看 `logs/app` 目录下是否自动生成 `info.log`、`error.log` 以及以日期切分的归档日志文件。
3. **测试不同日志级别与类包控制**：在代码中添加不同级别的日志（DEBUG、INFO、WARN、ERROR），确认它们的输出位置和是否受策略控制。

## 进阶与扩展

* **多环境配置**：利用 Spring Profiles 可以为开发、测试、生产环境提供不同的日志级别和策略配置文件。例如：
  `log4j2-spring-dev.xml`、`log4j2-spring-prod.xml`，并通过 `application-dev.yml` 与 `application-prod.yml` 自动加载相应的日志配置。
* **异步日志**：若对性能要求更高，可使用 `AsyncAppender` 提升日志写入效率。此时需谨慎调整参数，确保在高负载下日志不会过度堆积。
* **JSON 格式日志**：若想与 ELK（Elasticsearch、Logstash、Kibana）或其他日志分析工具更好整合，可将日志输出为 JSON 格式，方便被日志采集与分析系统解析。