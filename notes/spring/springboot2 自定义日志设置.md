# springboot2 自定义日志设置

## 本文目的：

1）、log分级，不同类型保存到指定路径，指定格式，指定滚动规则

2）、log按照环境选择不同策略；开发环境只输出console log ，其他环境输出其他类型log到文件。

3）、console log 采用springboot默认的pattern，其他log采用自定义pattern。

4）、视情况采用MDC。



## 措施与配置

1）、先上部分配置，`logback-spring.xml`

```xml
<!-- ConsoleAppender 控制台输出日志 -->
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>${CONSOLE_LOG_PATTERN}</pattern>
    </encoder>
</appender>
<!-- ERROR级别日志 -->
<appender name="ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.home_dir}/error/error.log</file>
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>ERROR</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <!-- rollover daily -->
        <fileNamePattern>${log.home_dir}/error/error-%d{yyyy-MM-dd}-%i.log.gz</fileNamePattern>
        <!-- each file should be at most 1MB, keep 60 days worth of history, but at most 20GB -->
        <maxFileSize>${log.maxFileSize}</maxFileSize>
        <maxHistory>${log.maxHistory}</maxHistory>
        <totalSizeCap>${log.totalSizeCap}</totalSizeCap>
    </rollingPolicy>
    <encoder encoder="UTF-8">
        <pattern>${patternStyle}</pattern>
    </encoder>
</appender>
<!--其他的和ERROR配置大同小异-->
<appender name="WARN" class="ch.qos.logback.core.rolling.RollingFileAppender" ...>
<appender name="INFO" class="ch.qos.logback.core.rolling.RollingFileAppender" ...>
<appender name="DEBUG" class="ch.qos.logback.core.rolling.RollingFileAppender" ...>
<appender name="TRACE" class="ch.qos.logback.core.rolling.RollingFileAppender" ...>
```

可以只采集部分日志。



2）、采用spring的profile机制，logback配置文件名字指定为`logback-spring.xml`。使profile机制生效。

配置如下：

```xml
<!--环境变量配置-->
<springProfile name="dev">
    <logger name="org.springframework" level="WARN" />
    <!--myibatis log configure-->
    <logger name="com.apache.ibatis" level="DEBUG"/>
    <logger name="java.sql.Connection" level="DEBUG"/>
    <logger name="java.sql.Statement" level="DEBUG"/>
    <logger name="java.sql.PreparedStatement" level="DEBUG"/>
    <!-- configuration to be enabled when the "dev" or "staging" profiles are active -->
    <!-- root级别 -->
    <root>
        <!-- 打印debug级别日志及以上级别日志 -->
        <level value="DEBUG"/>
        <!-- 控制台输出 -->
        <appender-ref ref="console"/>
    </root>
</springProfile>
<springProfile name="!dev">
    <logger name="org.springframework" level="ERROR" />
    <!--myibatis log configure-->
    <logger name="com.apache.ibatis" level="ERROR"/>
    <logger name="java.sql.Connection" level="ERROR"/>
    <logger name="java.sql.Statement" level="ERROR"/>
    <logger name="java.sql.PreparedStatement" level="ERROR"/>
    <!-- root级别 -->
    <root>
        <!-- 打印debug级别日志及以上级别日志 -->
        <level value="ERROR"/>
        <!-- 控制台输出 
            <appender-ref ref="console"/>-->
        <!-- 文件输出 -->
        <appender-ref ref="ERROR"/>
        <appender-ref ref="INFO"/>
        <appender-ref ref="WARN"/>
        <appender-ref ref="DEBUG"/>
        <appender-ref ref="TRACE"/>
    </root>
</springProfile>
```

除了在不同环境下指定日志输出方式，还指定了spring框架的log级别，指定了jdbc的log级别（这里是用mybatis，可以让它输出sql日志）。这些可以酌情配置。



3）、控制台日志输出格式使用springboot默认格式，优点是直接，美观。其他输出方式为自定义配置。

配置文件：

```xml
<!--自定义pattern-->
<property name="patternStyle" value="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} [%X{requestId}] [%X{requestSeq}] [%X{uri}] [%logger] %msg%n" />

<!--springboot默认pattern-->
<conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
<conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
<conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
<property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

```

springboot默认pattern来源于类`org.springframework.boot.logging.logback.DefaultLogbackConfiguration`的`CONSOLE_LOG_PATTERN` 属性。具体配置内容在：`org.springframework.boot.logging.logback.defaults.xml`中。

```xml
	<conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
	<conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
	<conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
	<property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

```

复制到`logback-spring.xml`中即可。



4）、待完善

