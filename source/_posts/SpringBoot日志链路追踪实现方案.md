---
title: SpringBoot日志链路追踪实现方案
date: 2020-05-30 23:05:53
tags: SpringBoot
categories: 
- Java
---

### MDC

MDC ( Mapped Diagnostic Contexts )，它是一个线程安全的存放诊断日志的容器。

Logback设计的一个目标之一是对分布式应用系统的审计和调试。在现在的分布式系统中，需要同时处理很多的请求。如何来很好的区分日志到底是那个请求输出的呢？我们可以为每一个请求生一个logger，但是这样子最产生大量的资源浪费，并且随着请求的增多这种方式会将服务器资源消耗殆尽，所以这种方式并不推荐。

一种更加轻量级的实现是使用MDC机制，在处理请求前将请求的唯一标示放到MDC容器中如sessionId，这个唯一标示会随着日志一起输出，以此来区分该条日志是属于那个请求的。并在请求处理完成之后清除MDC容器。底层是采用ThreadLocal实现的。

### TraceID工具类

```java
public class TraceIdUtil {
    private static final String TRACE_ID = "traceId";
    /**
     * 当traceId为空时，显示的traceId。随意。
     */
    private static final String DEFAULT_TRACE_ID = "0";

    /**
     * 设置traceId
     */
    public static void setTraceId(String traceId) {
        //如果参数为空，则设置默认traceId
        traceId = StringUtils.isBlank(traceId) ? DEFAULT_TRACE_ID : traceId;
        //将traceId放到MDC中
        MDC.put(TRACE_ID, traceId);
    }

    /**
     * 获取traceId
     */
    public static String getTraceId() {
        //获取
        String traceId = MDC.get(TRACE_ID);
        //如果traceId为空，则返回默认值
        return StringUtils.isBlank(traceId) ? DEFAULT_TRACE_ID : traceId;
    }

    /**
     * 判断traceId为默认值
     */
    public static boolean defaultTraceId(String traceId) {
        return DEFAULT_TRACE_ID.equals(traceId);
    }

    /**
     * 生成traceId
     */
    public static String genTraceId() {
        return UUID.randomUUID().toString();
    }
}
```

### 增加过滤器

```java
@WebFilter(urlPatterns = "/*", filterName = "traceIdFilter")
@Order(1)
public class TraceIdFilter extends GenericFilterBean {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        initTraceId((HttpServletRequest) servletRequest);
        filterChain.doFilter(servletRequest, servletResponse);
    }
    /**
     * traceId初始化
     */
    private void initTraceId(HttpServletRequest request) {
        //尝试获取http请求中的traceId
        String traceId = request.getParameter("traceId");

        //如果当前traceId为空或者为默认traceId，则生成新的traceId
        if (StringUtils.isBlank(traceId) || TraceIdUtil.defaultTraceId(traceId)) {
            traceId = TraceIdUtil.genTraceId();
        }
        //设置traceId
        TraceIdUtil.setTraceId(traceId);
    }
}
```

## logback-spring.xml配置

> ```
> //控制台格式 [%X{ traceId }] - 为输出当前的TraceID 格式
> <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr( %d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} [%X{ traceId }] - %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
> ```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration  scan="true" scanPeriod="10 seconds">

    <!--<include resource="org/springframework/boot/logging/logback/base.xml" />-->

    <contextName>logback</contextName>
    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义变量后，可以使“${}”来使用变量。 -->
    <property name="log.path" value="./logs" />

    <!-- 彩色日志 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
    <!-- 彩色日志格式 -->
    <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr( %d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} [%X{ traceId }] - %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>


    <!--输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>info</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>


    <!--输出到文件-->

    <!-- 时间滚动输出 level为 DEBUG 日志 -->
    <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_debug.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志归档 -->
            <fileNamePattern>${log.path}/debug/log-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录debug级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>debug</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 时间滚动输出 level为 INFO 日志 -->
    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_info.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${log.path}/info/log-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录info级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>info</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 时间滚动输出 level为 WARN 日志 -->
    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_warn.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/warn/log-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录warn级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>


    <!-- 时间滚动输出 level为 ERROR 日志 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_error.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error/log-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!--
        <logger>用来设置某一个包或者具体的某一个类的日志打印级别、
        以及指定<appender>。<logger>仅有一个name属性，
        一个可选的level和一个可选的addtivity属性。
        name:用来指定受此logger约束的某一个包或者具体的某一个类。
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
              还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。
              如果未设置此属性，那么当前logger将会继承上级的级别。
        addtivity:是否向上级logger传递打印信息。默认是true。
    -->
    <!--<logger name="org.springframework.web" level="info"/>-->
    <!--<logger name="org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor" level="INFO"/>-->
    <!--
        使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
        第一种把<root level="info">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息
        第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：
     -->


    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        不能设置为INHERITED或者同义词NULL。默认是DEBUG
        可以包含零个或多个元素，标识这个appender将会添加到这个logger。
    -->

    <!--开发环境:打印控制台-->
    <!--<springProfile name="dev">-->
        <!-- 请更改此处包名！！！！！！或类 ！！！！！！-->
        <!--<logger name="com.nmys.view" level="debug"/>-->
    <!--</springProfile>-->

    <root level="info">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="DEBUG_FILE" />
        <appender-ref ref="INFO_FILE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>

    <!--生产环境:输出到文件-->
    <!--<springProfile name="pro">-->
    <!--<root level="info">-->
    <!--<appender-ref ref="CONSOLE" />-->
    <!--<appender-ref ref="DEBUG_FILE" />-->
    <!--<appender-ref ref="INFO_FILE" />-->
    <!--<appender-ref ref="ERROR_FILE" />-->
    <!--<appender-ref ref="WARN_FILE" />-->
    <!--</root>-->
    <!--</springProfile>-->

</configuration>
```

### 多线程打印 TraceID

```java
@Slf4j
public class LOGTestHandler {
    public void handle(){
        log.info("测试通过112321");

        ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
                .setNameFormat("demo-pool-%d").build();
        ExecutorService singleThreadPool = new ThreadPoolExecutor(3, 5,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(20), namedThreadFactory, new ThreadPoolExecutor.AbortPolicy());
        String traceId = TraceIdUtil.getTraceId();
        for (int i = 0; i < 20; i++) {


            singleThreadPool.execute(()-> {
                TraceIdUtil.setTraceId(traceId);
                log.info("测试多线程");
                System.out.println("asasdadad:"+traceId);

            });
        }
        singleThreadPool.shutdown();

    }
}
```

### 最终日志效果

```log
 2020-04-02 15:46:52.623  INFO 21276 --- [nio-9009-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : [] - Initializing Spring FrameworkServlet 'dispatcherServlet'
 2020-04-02 15:46:52.623  INFO 21276 --- [nio-9009-exec-1] o.s.web.servlet.DispatcherServlet        : [] - FrameworkServlet 'dispatcherServlet': initialization started
 2020-04-02 15:46:52.658  INFO 21276 --- [nio-9009-exec-1] o.s.web.servlet.DispatcherServlet        : [] - FrameworkServlet 'dispatcherServlet': initialization completed in 34 ms
 2020-04-02 15:46:52.693  INFO 21276 --- [nio-9009-exec-1] com.example.demo.TestIDSController       : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - name: cccssa
当前的tranceID是：7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.694  INFO 21276 --- [nio-9009-exec-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试通过112321
 2020-04-02 15:46:52.701  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.702  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.702  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.702  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.702  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.703  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.706  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
 2020-04-02 15:46:52.706  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.706  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [7a02efc5-0722-491c-92e2-1b713c54ebd9] - 测试多线程
asasdadad:7a02efc5-0722-491c-92e2-1b713c54ebd9
 2020-04-02 15:46:52.773  INFO 21276 --- [nio-9009-exec-3] com.example.demo.TestIDSController       : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - name: cccssa
当前的tranceID是：e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.774  INFO 21276 --- [nio-9009-exec-3] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试通过112321
 2020-04-02 15:46:52.775  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.777  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.778  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.778  INFO 21276 --- [    demo-pool-1] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
 2020-04-02 15:46:52.778  INFO 21276 --- [    demo-pool-2] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.778  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.780  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.781  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.781  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.781  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
 2020-04-02 15:46:52.781  INFO 21276 --- [    demo-pool-0] com.example.demo.IDP2.LOGTestHandler     : [e0bd73f6-1d33-4eda-909f-21ff94f4b763] - 测试多线程
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
asasdadad:e0bd73f6-1d33-4eda-909f-21ff94f4b763
```