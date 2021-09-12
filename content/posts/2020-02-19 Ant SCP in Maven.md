---
title: Ant SCP in Maven
date: "2020-02-19T15:20:00.000Z"
description: "Ant SCP in Maven"
---

先放代码

```xml
<plugin>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.7</version>
    <executions>
        <execution>
        <phase>validate</phase>
        <configuration>
            <tasks>
            <mkdir dir="${project.build.directory}/yy" />		              
            <scp file="user:password@host:/home/xx/yy/*" todir="${project.build.directory}/yy" trust="yes"/>
            </tasks>
        </configuration>
        <goals>
            <goal>run</goal>
        </goals>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.apache.ant</groupId>
            <artifactId>ant-jsch</artifactId>
            <version>1.7.1</version>
        </dependency>
    </dependencies>
</plugin>	
```

* 第一个要注意的是这个plugin由于用到SSH，所以需要额外的ant-jsch包，需要另外加上这个依赖。

* 由于需要认证，所以，需加上`trust=true`, 否则会出如下错误：

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.7:run (default) on project icva-extractor: An Ant BuildException has occured: com.jcraft.jsch.JSchException: reject HostKey: xxx
```

* 由于scp不能自己在文件夹不存在的情况下创建文件夹，所以需要额外加上`mkdir`。 当然，如果是从本地拷贝到远程服务器，那就需要用到`sshexec`。

```xml
<sshexec
    host="${host}"
    username="${remote_user}"
    password="${remote_password}"
    command="mkdir -p ${remote_dir_path}"
    trust="true" />
```

* 到这里还有一个问题，就是如何处理用户名密码的问题。目前的办法就是定义在Maven的settings.xml里面。

```xml
<scp file="${scpUserName}:${scpUserPassword}@company.net:/home/xx/yy/*" todir="${project.build.directory}/yy" trust="yes"/>
```

然后你需要在pom.xml和settings.xml里面都要定义一个Profile，把ant的plugin放在里面，然后不要忘记在两边都激活它。 两种方式，一种是显示激活，就是在settings.xml里面加上

```xml
<activeProfiles>
    <activeProfile>unix</activeProfile>
</activeProfiles>
```
这样在pom.xml里面就不需要激活了。profile里面包含对应id即可。还有一种是隐式激活，比如在Unix操作系统下面才执行。这种情况下，两边的Profile里面都需要有`activation`。这样settings.xml里面的property才能注入到pom.xml里面去。 具体可以参考[这里](https://maven.apache.org/examples/injecting-properties-via-settings.html)

```xml
<profiles>
    <profile>
        <id>unix</id>
        <activation>
            <os>
                <family>unix</family>
            </os>			
        </activation>
        <properties>
            <scpUserName>xxx</scpUserName>
            <scpUserPassword>xxx</scpUserPassword>
        </properties>
    </profile>
</profiles>	
```

* 在stackoverflow提了个问题，根据里面的[建议](https://stackoverflow.com/questions/60325368/whats-the-best-way-to-store-username-password-for-ant-scp-task),更进一步的方式就是用ssh key来弄，可是考虑到每个teamcity的build agent都需要安装key...