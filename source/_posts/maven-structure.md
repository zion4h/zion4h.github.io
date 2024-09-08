---
title: Maven 结构解析
date: 2023-05-28 22:29:57
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img015.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img015.jpg
categories: [IT]
tags:
    - maven
toc: true
---

本文将简单介绍如何解析 Maven 的 目录结构和 POM 文件结构。

<!--more-->

## 目录结构

Maven 有一套默认的目录结构，[这篇](https://www.cnblogs.com/now-fighting/p/4858982.html) 文章已经总结的很好了，我们当然不用硬记住每一项的名字用处，因为这本身就是约定俗成的。程序员最喜欢打破常规，即使是键盘也要自定义映射整成全平台统一。不过如果真有厂商 `ALL IN ONE` 了，我们又会出于安全考虑 DIY 一份。当然我也知道为了方便大家理解，名字就是用处，但是也有例外，比如 `it` 表示**集成测试**，这个我真是第一次见。言归正传，总之就是不要硬背了，但至少要明白 `target`、`src/main/java` 和 `src/test/java`。

## POM 结构

Maven 更重要的是配 **POM**（Project Object Model）文件，它利用 XML 将项目的**依赖项**、**构建方式**、**开发者信息**等组织起来。其中，最重要的属性是 `groupId`、`artifactId` 和 `version`。

不同 POM 文件之间的组织关系可以分为**项目聚合**和**项目继承**两类，也就是说两个 POM 之间既可以分上下级（父子关系），也可以并列（聚合模块）。

### 继承关系示例代码

`com.mycompany.app:my-module:1` 的 POM 文件

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <parent> 
        <groupId>com.mycompany.app</groupId> 
        <artifactId>my-app</artifactId> 
        <version>1</version> 
    </parent> 
    <groupId>com.mycompany.app</groupId> 
    <artifactId>my-module</artifactId> 
    <version>1</version>
</project>
```

如果和父 POM 的版本、组织绑定，那 `com.mycompany.app:my-module:1` 的 POM 文件也可以简写为

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <parent> 
        <groupId>com.mycompany.app</groupId> 
        <artifactId>my-app</artifactId> 
        <version>1</version> 
    </parent> 
    <artifactId>my-module</artifactId>
</project>
```

`com.mycompany.app:my-app:1` 的 POM 文件

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.mycompany.app</groupId> 
    <artifactId>my-app</artifactId> 
    <version>1</version>
</project>
```

### 聚集关系示例代码

`com.mycompany.app:my-module:1` 的 POM 文件

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.mycompany.app</groupId> 
    <artifactId>my-module</artifactId>
    <version>1</version>
</project>
```

`com.mycompany.app:my-app:1` 的 POM 文件

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.mycompany.app</groupId> 
    <artifactId>my-app</artifactId> 
    <version>1</version> 
    <packaging>pom</packaging> 
    <modules> 
        <module>my-module</module> 
    </modules>
</project>
```

### 同时整合聚集和继承

`com.mycompany.app:my-app:1` 的 POM

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <groupId>com.mycompany.app</groupId> 
    <artifactId>my-app</artifactId> 
    <version>1</version> 
    <packaging>pom</packaging> 
    <modules> 
        <module>../my-module</module> 
    </modules>
</project>
```

`com.mycompany.app:my-module:1` 的 POM

```xml
<project> 
    <modelVersion>4.0.0</modelVersion> 
    <parent> 
        <groupId>com.mycompany.app</groupId> 
        <artifactId>my-app</artifactId> 
        <version>1</version> 
        <relativePath>../parent/pom.xml</relativePath> 
    </parent> 
<artifactId>my-module</artifactId>
</project>
```

## 项目插值

项目插值可以通过形如 `${variable}` 来引用某个**变量**的值，减少代码冗余。这里的变量又分三类，一种很好理解，就是 **xml 标签对应的值**，比如 `${project.groupId}`、 `${project.version}`、 `${project.build.sourceDirectory}` 等。第二种是 `Properties`，需要我们自己定义，比如：

```xml
<project> 
    ... 
    <properties> 
        <mavenVersion>3.0</mavenVersion> 
    </properties> 

    <dependencies> 
        <dependency> 
            <groupId>org.apache.maven</groupId> 
            <artifactId>maven-artifact</artifactId>
            <version>${mavenVersion}</version> 
        </dependency> 
        <dependency>      
            <groupId>org.apache.maven</groupId> 
            <artifactId>maven-core</artifactId> 
            <version>${mavenVersion}</version> 
        </dependency> 
    </dependencies>  
    ...
</project>
```

第三种是**特殊变量**，包括：

1. `project.basedir` 当前项目所在的目录。
2. `project.baseUri` 当前项目所在的目录，表示为 URI
3. `maven.build.timestamp` 表示构建开始时的时间戳（使用协调世界时 UTC）。

我们甚至可以利用 **Properties** 指定 `maven.build.timestamp` 的格式：

```xml
<project> 
    ... 
    <properties> 
        <maven.build.timestamp.format>yyyy-MM-dd'T'HH:mm:ss'Z'</maven.build.timestamp.format> 
    </properties> 
    ...
</project>
```

## 参考

【1】[Introduction to the POM](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html)

【2】[Maven 学习 - 目录结构](https://www.cnblogs.com/now-fighting/p/4858982.html)
