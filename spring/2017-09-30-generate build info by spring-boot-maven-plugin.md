---
title: Generate build info by spring-boot-maven-plugin
categories: java spring
author: forhot2000@qq.com
date: 2017/09/30
---

Spring Boot Actuator 为我们提供了很多调试信息，其中包含一个 /info ，用于显示应用程序的信息（在 `application.properties` 中以 `info.` 开始的配置项），通常我们可以在这里显示 `scm-url`, `build-url`, `version`, `description` 等信息，而这些信息一般又存放在 `pom.xml` 中的，我们希望仅修给一个地方就可以同时在 pom 和 actuator info 两个地方生效，幸运的是 `spring-boot-maven-plugin` 能为我们做到这些。

首先，我们在 `pom.xml` 中配置这些信息

```xml
  <artifactId>sprint-sleuth-test-admin</artifactId>
  <packaging>jar</packaging>
  <version>0.0.1-SNAPSHOT</version>
  <description>description for this application</description>
  <scm>
    <url>/d/git/spring-sleuth-test.git</url>
  </scm>
```

然后，在 `application.properties` 配置从 `pom.xml` 文件和环境变量中读取运行值

```
info.scm-url=@pom.scm.url@
info.build-url=http://jenkins.szewec.com/jobs/build/@env.BUILD_NUMBER@
info.name=@pom.artifactId@
info.version=@pom.version@
info.description=@pom.description@
```

配置好了后，我们设置本地环境变量 `BUILD_NUMBER` 模拟从 jenkins 编译

```sh
export BUILD_NUMBER=28
mvn clean package
```

运行程序

```sh
java -jar target/sprint-sleuth-test-admin-0.0.1-SNAPSHOT.jar
```

测试结果，打开 `localhost:8080/info` 可以得到如下结果：

```json
{
  "scm-url": "/d/git/spring-sleuth-test.git",
  "build-url": "http://jenkins.example.com/jobs/build/28",
  "name": "sprint-sleuth-test-admin",
  "version": "0.0.1-SNAPSHOT",
  "description": "description for this application"
}
```
