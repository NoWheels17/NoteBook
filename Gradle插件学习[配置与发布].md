## Gradle插件学习2
### 学习目标

这篇学习笔记主要的学习目标就是掌握入门的工程创建、工程中每个文件的作用、工程配置的含义、以及如何打包发布项目，最好会涉及到某个博主推荐的方便调试的项目结构

### 预备知识

想起来，上次学习gradle还是上次（我是想说很久以前），此前学gradle都是从写task的角度开始的，task是gradle脚本逻辑的最小单元，其实也是3中自定义方式里最简单的一种，剩下两种分别是：

- buildSrc

可以将插件的源代码放在`my-gradle/src/main/java`目录（或`my-gradle/src/main/groovy`或`my-gradle/src/main/kotlin`中，具体取决于你使用的语言）。 Gradle将负责编译插件，并使其在构建脚本的类路径中可用。 该插件对整个项目里的每个构建脚本都是可见的， 但是，它在项目外部不可见，因此不能其他项目中复用该插件。

上面这句话是官方的翻译，这里解释一下，用Studio新建项目MyProject后，再新建一个module名字为`my-gradle`。这样上面的目录结构就能理解了，其实就是正常的App开发项目，根据使用的语言区分一下而已

- 独立项目

可以为插件创建一个单独的项目，将项目打包成一个JAR包，然后可以在多个项目中复用，独立项目必须要会发布。

### 独立项目

很多博客都不适用于初学者，就是因为少了这一块内容。一个新独立项目的开发，首先就需要知道项目的结构以及各个文件的大致作用，不然就算学会了也无法举一反三。下面就以最正式的独立项目开发来学习整个插件的开发，使用的语言还是沿用第一次学习的Kotlin。

#### 创建模版项目

最开始接触的文章都是以创建一般的Android项目为基础，然后删除各种文件来满足开发Gradle插件开发的项目结构，这样方式还需要自己手动修改和配置gralde中的内容，实际上我个人觉得是走了弯路，最好的办法是使用gradle的指令来生成项目。

首先新建文件夹MyPlugin，然后在 `kevinlee@KevinLeedeMacBook-Pro MyPlugin %`下键入指令gradle init（我用的是7.0.2的版本，需要java11） ：

```
Select type of project to generate:
  1: basic
  2: application
  3: library
  4: Gradle plugin
  
# 1、✅这里选4 Gradle 插件
Enter selection (default: basic) [1..4] 4

# 2、✅实现语言选择，最近kotlin生疏了，选Kt好了
Select implementation language:
  1: Groovy
  2: Java
  3: Kotlin
Enter selection (default: Java) [1..3] 3

# 3、✅DSL也就是我们编译脚本语言，同样选Kt
Select build script DSL:
  1: Groovy
  2: Kotlin
Enter selection (default: Groovy) [1..2] 2

# 4、✅这里其实是在填写包名
Project name (default: hello):
Source package (default: hello): com.kevin.study
```

#### 项目结构分析

这样就能得到一个kotlin编写的gradle插件项目，主要结构如下：

```
MyPlugin // 创建的文件夹的名字
|-plugin // 默认生成的module名称
	｜-src
		｜-main
			|-kotlin
				|-com.kevin.study // 上面第4步生成的包名
					MyStudyPlugin // 自定义差极差实现类
	 build.gradle.kts // 与src同级
```

gradle中内容我做了对test部分的裁剪，内容如下：

```kotlin
plugins {
    // 这句话很重要，表示这个module是一个gradle插件组件
    `java-gradle-plugin`
    // 因为我用的是kt写插件，所以必须导入jvm对kt的支持插件
    id("org.jetbrains.kotlin.jvm") version "1.4.31"
}

repositories {
    // 仓库依赖
    mavenCentral() // 默认有的
    mavenLocal() // 我自己添加的
}

dependencies {
    // Align versions of all Kotlin components
    implementation(platform("org.jetbrains.kotlin:kotlin-bom"))
    // Use the Kotlin JDK 8 standard library.
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
}
// ==========往上都跟Android的差不多========

// 这里则开始定义了
gradlePlugin {
    // 定义插件，这个studyDemo是我随便取的名字，目前没有发现有什么作用
    val studyDemo by plugins.creating {
        id = "com.kevin.study"
        implementationClass = "com.kevin.study.MyStudyPlugin"
    }
}
```

以上就是完整的项目结构了，在模版项目中，自定义插件里面的其实只是往project中注册了一个greeting的task然后打印了一句话，就这么简单而已。

### 项目发布

就像我们发布aar组件一样，gradle插件也是需要发布到仓库中方便多项目引用的。用kotlin写的话，大致就能猜到发布的东西就是字节码压缩包，例如jar包。

#### 配置发布

这一步的时候，也是先套用Android项目的思维方式，需要三方功能，则需要一个插件，以及相关的配置，试想一下项目中在使用上传插件时我们时怎么做的，这里也是类似。

```kotlin
plugins {
    `java-gradle-plugin`
    id("org.jetbrains.kotlin.jvm") version "1.4.31"
  	// 用来maven发布的gradle插件
    id("maven-publish")
}
```

然后就是定义我们需要发布的基础信息了，这里其实就是定义`maven-publish`需要的参数了：

```kotlin
publishing {
    publications {
        create<MavenPublication>("studyDemo"){
            groupId = "com.kevin.study" // 打出来的包的前缀
            artifactId = "study-demo" // 打出来的包的名称
            version = "0.0.1-LOCAL" // 版本
            from(components["java"]) // 表示是java组件
        }
    }
}
```

简直太眼熟了，Android打aar上传的时候，参数也是这几个，只不过要说明的是版本号的`-LOCAL`并不是指要上传到本地仓库，这个规则是我们自己定义的组件去实现的，并不是插件自己去实现的。

（*这里其实还需要注意一些点，往上很多博客晒的是groovy语言写的demo，很多时候还是的得去官网找demo*）

接下来才是去配置插件发布的位置：

要说的是其实就算没有配置，点击一下🐘同步一下，在右侧gradle的任务列表中已经出现了：

```
publishStudyDemoPublicationToMavenLocal
```

这个任务就是默认发布到本地，双击试了一下，确实发布到了`~/.m2/repository/com/kevin/study/study-demo/`这个目录下，~是使用了Linux的标记方式，表示在用户目录下。这个目录下，文件结构是：

```
study-demo
	|-0.0.1-LOCAL
		|-study-demo-0.0.1-LOCAL.jar
		|-study-demo-0.0.1-LOCAL.module
		|-study-demo-0.0.1-LOCAL.pom
	|-maven-metadata-local.xml
```

#### 私有Maven发布

一般自己公司的项目都会发在私有的Maven仓库里，如果需要使用`MavenCentral()`则需要一连串的申请手续，这里先学习如何推送到Nexus 私有服务。

这里找到一篇怎么在Centos下搭建Maven的教程（说来惭愧，Linux系统只用过Centos7）

[Centos下 Nexus 3.x 搭建Maven 私服](https://juejin.cn/post/6844903879910359047)

下面上一下完整配置：

```kotlin
repositories {
    maven {
        // 仓库名称
        name = "custom"
        //允许使用 http
        isAllowInsecureProtocol = true
        if (verCode.endsWith("-LOCAL")) {
            url = uri("/Users/kevinlee/Downloads/repo/LOCAL")
        } else if (verCode.endsWith("-SNAPSHOT")) {
            url = uri("/Users/kevinlee/Downloads/repo/SNAPSHOT")
        }
        // 配置私有maven的上传账号信息
        credentials {
            username = "userName"
            password = "passWord"
        }
    }
}
```

我这里为了测试这个配置是否正确，所以就整到本地路径下了，因为搞个Nexus服务临时没有这个条件，但是作为延伸我们区分了LOCAL和SNAPSHOT的两种方式发布。

### 使用验证

包已经发布了，下面就是验证体验一下，完成项目创建-开发-配置-发布-使用的闭环

- 先找一个小白鼠项目，在根项目里面的build.gradle配置脚本的仓库，包含本地仓库（一般默认都有了，确认一下就好了）

```groovy
buildscript {
  repositories {
    //...
    mavenLocal()
    //...
  }
}
```

因为我们此前打的插件默认放在本地默认仓库了，所以这里要引用这个路径

- 依赖我们的插件

```groovy
dependencies {
  //...
  classpath "com.kevin.study:study-demo:0.0.1-LOCAL"
  //...
}
```

- 在对应项目Module的build.gradle中应用我们的插件，通过id

  通过id？啥id？哪个id？

  [项目结构分析](#项目结构分析)

  这里配置gradlePlugin里面定义的id就是我们插件的id

  ```groovy
  plugins {
      id 'com.android.application'
      id 'com.kevin.study' // 我自己的插件
  }
  ```

​		这样就搞好了

- 最后点一下🐘，同步一下gradle配置，然后就能看到我们原始demo中自定义的task `greeting` 出现在了other中的任务列表里，双击执行一下：

  > Task :app:greeting
  > Hello from plugin 'study.greeting'
