---
title: Log4j如何记录多线程
date: "2021-09-02T13:17:00.000Z"
description: "记录一下从1.x到2.x的升级过程"
---

有一个多线程程序，我需要按照时间顺序把每个线程的log如实地记录在一个文件里。也就是说必须是同步的。可以有多种方法，我这里使用的是Log4j，就介绍一下在新老两个版本中的实现。

## Log4j 1.x
在log4j 1.x版本中，是如下设计的。首先启动一个log4j Server：

```bash
CLASSPATH="$REPO"/log4j-1.2.17.jar:"$REPO"/apache-log4j-extras-1.2.17.jar
EXTRA_JVM_ARGUMENTS="-Dlog.dir=$LOG_DIR"

exec "$JAVACMD" $JAVA_OPTS \
	$EXTRA_JVM_ARGUMENTS \
	-cp "$CLASSPATH" org.apache.log4j.net.SimpleSocketServer 4712 "$BASEDIR"/etc/log4j-server.properties
```
也就是开了一个Socket，这样不管多少线程，大家都可以向这个Socket里面发送log。可以保证同步。

log4j-server.properties的内容是：

```property
log4j.rootLogger=INFO, logfile

# LOGFILE is set to be a File appender using a PatternLayout.
# Use log4j-extra appender to void log lost: 
# https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/DailyRollingFileAppender.html suggest log lost
# https://bz.apache.org/bugzilla/show_bug.cgi?id=36384
# https://stackoverflow.com/a/6037141
log4j.appender.logfile = org.apache.log4j.rolling.RollingFileAppender
log4j.appender.logfile.File=${log.dir}/xxx-extra/xxx.log
log4j.appender.logfile.rollingPolicy = org.apache.log4j.rolling.TimeBasedRollingPolicy
log4j.appender.logfile.rollingPolicy.FileNamePattern = ${log.dir}/xxx-extra/xxx.log.%d{yyyyMMdd}.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
```
通过上面的注释可以看到，log4j自身的org.apache.log4j.DailyRollingFileAppender是有问题的，会丢数据，所以必须要使用apache-log4j-extras这个jar中的org.apache.log4j.rolling.RollingFileAppender。

在发送端，配置是

```properties
# Root logger option
log4j.rootLogger=INFO, stdout, server

# Direct log messages to stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

log4j.appender.server=org.apache.log4j.net.SocketAppender
log4j.appender.server.Port=4712
log4j.appender.server.RemoteHost=localhost
log4j.appender.server.ReconnectionDelay=10000
```

## Log4j 2.x

现在必须要在2.x上实现这个功能。分成两步，第一步是去掉1.x的依赖，增加2.x的依赖，同时尽量不更改代码。我们可以使用Bridge包来实现，这样一共就是三个包就可以了。

```xml
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.14.1</version>
    </dependency>         
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.14.1</version>
    </dependency>  
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-1.2-api</artifactId>
        <version>2.14.1</version>
    </dependency> 
```

第二步是用2.x版本中的SocketServer来代替之前SimpleSocketServer中的实现。可是在新版本中并没有server的实现，查阅后才知道在2.10版本之后这部分内容被踢出去了，在如下位置[https://github.com/apache/logging-log4j-tools/tree/release-2.x](https://github.com/apache/logging-log4j-tools/tree/release-2.x)，下载之后利用assembly打出一个fat jar。这样我们就可以利用它来启动log4j Server了。

```xml
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id> <!-- this is used for inheritance merges -->
            <phase>package</phase> <!-- bind to the packaging phase -->
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin> 
```

启动的命令如下：

```bash
CLASSPATH="$BASEDIR"/lib2/log4j-server-2.14.1-jar-with-dependencies.jar
EXTRA_JVM_ARGUMENTS="-Dlog4j2.configurationFile="$BASEDIR"/etc/log4j2-server.properties"

exec "$JAVACMD" $JAVA_OPTS \
	$EXTRA_JVM_ARGUMENTS \
	-cp "$CLASSPATH" org.apache.logging.log4j.server.TcpSocketServer --wire-format SERIALIZED -a localhost -p 4712
```
上面这个命令中，我们是通过-D把log4j的配置文件传入进去的，也曾尝试使用TcpSocketServer的“-c”参数，发现不行。


而log4j2-server.properties 的内容如下:

```properties
appender.console.type = Console
appender.console.name = LogToConsole
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

appender.rolling.type = RollingFile
appender.rolling.name = LogToRollingFile
appender.rolling.fileName = ${env:LOG_DIR}/xxx-extra/xxx.log
appender.rolling.filePattern = ${env:LOG_DIR}/xxx-extra/xxx.log.%d{yyyyMMdd}.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = %d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy

# Log to console and rolling file
rootLogger.level = info
rootLogger.appenderRef.stdout.ref = LogToConsole
rootLogger.appenderRef.rolling.ref = LogToRollingFile
```

同样发送端的配置也有所更改：

```properties
appender.console.type = Console
appender.console.name = LogToConsole
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n

appender.socket.type = Socket
appender.socket.name = LogToSocket
appender.socket.port = 4712
appender.socket.host = localhost
appender.socket.layout.type = SerializedLayout

rootLogger.level = info
rootLogger.appenderRef.stdout.ref = LogToConsole
rootLogger.appenderRef.socket.ref = LogToSocket
```

这中间有个比较有趣的地方是新版本的log4j竟然不能支持从虚拟机"-D"拿变量，查阅[Lookup](https://logging.apache.org/log4j/2.x/manual/lookups.html)后才知道要加个前缀`env:`才可以拿声明的环境变量。
