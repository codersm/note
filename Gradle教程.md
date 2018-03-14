---
title: Gradle教程
date: 2017-08-04 11:50:16
tags: Gradle
---
# Gradle教程

## Gradle构建脚本

Gradle构建脚本文件用来处理两件事情：一个是项目和另一个的任务。每个Gradle生成表示一个或多个项目。一个项目表示一个JAR库或Web应用程序，也可能表示由其他项目产生的JAR文件组装的ZIP。 简单地说，一个项目是由不同的任务组成。一个任务是指构建执行的一块工作。任务可能是编译一些类，创建一个JAR，产生的Javadoc或发布一些归档文件库。

Gradle使用Groovy语言编写脚本，使其更容易来形容和构建。Gradle 中的每一个构建脚本使用UTF-8进行编码保存，并命名为 build.gradle。

**使用示例**：

* Hello world

* 循环

* Groovy的JDK方法

## Gradle任务

**定义任务**

**任务依赖关系**

```gradle
task hello << {
    println 'Hello world!'
}
task intro(dependsOn: hello) << {
    println "I'm Gradle"
}
```

还有另一种方法来添加任务依赖，它就是通过使用闭包。在这种情况下，任务通过闭包释放如果您在构建脚本中使用闭包，那么应该返回任务对象的单个任务或集合。

**跳过任务**

如果用于跳过任务的逻辑不能用谓词表示，则可以使用StopExecutionException。 如果操作抛出此异常，则会跳过此操作的进一步执行以及此任务的任何后续操作的执行。

```gradle
task compile << {
    println 'We are doing the compile.'
}

compile.doFirst {
    // Here you would put arbitrary conditions in real life.
    // But this is used in an integration test so we want defined behavior.
    if (true) { throw new StopExecutionException() }
}
task myTask(dependsOn: 'compile') << {
   println 'I am not affected'
}
```

## Gradle依赖管理

Gradle构建脚本定义了构建项目的过程; 每个项目包含一些依赖项和一些发表项。依赖性意味着支持构建项目的东西，例如来自其他项目的所需JAR文件以及类路径中的外部JAR（如JDBC JAR或Eh-cache JAR）。发布表示项目的结果，如测试类文件和构建文件，如war文件。

Gradle负责构建和发布结果。 发布基于定义的任务。 可能希望将文件复制到本地目录，或将其上传到远程Maven或lvy存储库，或者可以在同一个多项目构建中使用另一个项目的文件。 发布的过程称为发布。

**声明依赖关系**

```gradle
apply plugin: 'java'

repositories {
   mavenCentral()
}

dependencies {
   compile group: 'org.hibernate', name: 'hibernate-core', version: '3.6.7.Final'
   testCompile group: 'junit', name: 'junit', version: '4.+'
}
```

**依赖关系配置**

依赖关系配置只是定义了一组依赖关系。 您可以使用此功能声明从Web下载外部依赖关系。这定义了以下不同的标准配置。

编译 − 编译项目的生产源所需的依赖关系。

运行时 - 运行时生产类所需的依赖关系。 默认情况下，还包括编译时依赖项。

测试编译 - 编译项目测试源所需的依赖项。 默认情况下，它包括编译的产生的类和编译时的依赖。

测试运行时 - 运行测试所需的依赖关系。 默认情况下，它包括运行时和测试编译依赖项。

**存储库**

在添加外部依赖关系时， Gradle在存储库中查找它们。 存储库只是文件的集合，按分组，名称和版本来组织构造。 默认情况下，Gradle不定义任何存储库。 我们必须至少明确地定义一个存储库。

```gradle
repositories {
    mavenCentral()
    maven { url "https://maven.eveoh.nl/content/repositories/releases" }
}
```

**发布文件**

依赖关系配置也用于发布文件。 这些已发布的文件称为工件。 通常，我们使用插件来定义工件。 但是需要告诉Gradle在哪里发布文件。可以通过将存储库附加到上传存档任务来实现此目的。

```gradle
apply plugin: 'maven'

uploadArchives {
   repositories {
      mavenDeployer {
         repository(url: "file://localhost/tmp/myRepo/")
      }
   }
}
```

## Gradle插件

插件只是一组任务，几乎所有的任务，如编译任务，设置域对象，设置源文件等都由插件处理。

Gradle中有两种类型的插件：脚本插件和二进制插件。 脚本插件是一个额外的构建脚本，它提供了一种声明性方法来操作构建，通常在构建中使用。 二进制插件是实现插件接口并采用编程方法来操作构建的类。二进制插件可以驻留在插件JAR中的一个构建脚本和项目层次结构或外部。

## Gradle多项目构建



[Gradle入门教程](http://blog.jobbole.com/tag/gradle/)