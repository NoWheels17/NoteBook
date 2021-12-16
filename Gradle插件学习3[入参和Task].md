## Gradle插件学习3[aar发布插件]

### 学习目标

学会使用插件干预或者定制打包流程，了解插件拓展参数的使用方法，学会给自定义插件定义的task单独分类，熟悉自定义插件Task的写法，从build.gradle文件的脚本阶段过渡到插件。（过了这一期之后，虽然实现了功能但是是直接开始实现，所以理论知识肯定很欠缺，后续的gradle学习将会着重于补强理论知识）

### 知识点

- 如何自定义gradle task的文件夹，而不是放在other中？
- 如何让插件能接收参数？
- 如何知道打包已经完成？
- 如何获取到输出路径？
- 如何发布aar到Maven？

### 准备

1、根据官网的说明，我们使用7.0.2的gradle开发时，build插件也要升级到7+，所以在插件module中gradle配置需要更新

```kotlin
repositories {
    mavenCentral()
    mavenLocal()
    google() // 这个仓库地址一定要加，否则无法引用下面的build插件，这个gradle官方有在文档中说明（藏的有点深）
}

dependencies {
    implementation(platform("org.jetbrains.kotlin:kotlin-bom"))
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")
  // 使用4.0.2的构建插件
    implementation("com.android.tools.build:gradle:4.0.2")
}
```

2、因为目前使用的是Gradle6在处理，所以会出现一些兼容性提示

> Deprecated Gradle features were used in this build, making it incompatible with Gradle 7.0.
> Use '--warning-mode all' to show the individual deprecation warnings.

​	如果有上面的提示，可以使用以下指令查看详细需要解决的点

> ./gradlew uploadDebug --warning-mode all --stacktrace

### 逐个击破

#### 参数接收

- 定义一个参数模型

  这里有2个知识点要着重记一下：

  1、class模型必须是open的

  2、成员变量必须是public或者protected

  因为在实际运行的时候，时动态创建了一个类来继承我们这个数据类，如果类时final，变量时private的，那么我们的配置将无法生效

  ```kotlin
  open class UpInfo {
  
      var groupId: String? = null // 打出来的包的前缀
      var artifactId: String? = null // 打出来的包的名称
      var verCode: String? = null // 版本
  
      companion object {
          fun getUpInfo(project: Project) :UpInfo {
              val info = project.extensions.findByType(UpInfo::class.java)
              return info?: UpInfo()
          }
      }
  
      fun isValid() :Boolean {
          return !groupId.isNullOrEmpty() && !artifactId.isNullOrEmpty() && !verCode.isNullOrEmpty()
      }
  
      override fun toString(): String {
          return "groupId:$groupId | artifactId:$artifactId | verCode:$verCode"
      }
  
  }
  ```

- 在Plugin中通过project定义参数

  ```kotlin
  class MyStudyPlugin: Plugin<Project> {
  
      companion object {
          const val UPLOAD_INFO = "upInfo"
      }
  
      override fun apply(project: Project) {
          // 1、定义扩展参数
          project.extensions.create(UPLOAD_INFO, UpInfo::class.java)
          
          project.tasks.register("greeting") { task ->
              task.doLast {
                  // 2、在task中使用参数
                  val info = UpInfo.getUpInfo(project)
                  println("MyStudyPlugin get upInfo: $info")
              }
          }
      }
  }
  ```

- 调试

  基于我们上次创建（学习2的Demo）的项目，发布本地组件，然后在其他Android项目中使用

  **builde.gradle**

  ```groovy
  upInfo {
      groupId = "com.kevin.demo"
      artifactId = "gdt-web"
      verCode = "0.0.1"
  }
  ```

  运行我们`greeting` Task，可以看到打印出了

  > Task :app:greeting
  > MyStudyPlugin get upInfo: groupId:com.kevin.demo | artifactId:gdt-web | verCode:0.0.1

​		打完收工！

#### 自定义Task Group

这个知识点，刚开始没实现的时候，觉得可能会需要乱起八糟配置啥的。其实很简单

```kotlin
project.tasks.register("greeting") { task ->
    task.group = "kevin" // 定义一个task的时候，说明一下group就好了
    task.doLast {
        // 2、在task中使用参数
        val info = UpInfo.getUpInfo(project)
        println("MyStudyPlugin get upInfo: $info")
    }
}
```

这样，我就不用在other中滚动一长条去找我自己的Task了，直接从kevin文件夹中寻找

#### 最后一个task——assemble

实际上，如果想知道aar是否构建完成，只要知道assemble是否执行完即可。于是我写了这么一出：

> ```
> val asReleaseTask = project.tasks.findByName("assembleRelease")
> val uploadReleaseTask = generateReleaseUploadTask(project)
> uploadReleaseTask.dependsOn(asReleaseTask)
> ```

然后同步完插件之后直接报错 `找不到 assembleRelease 这个task`

这个时候我突然醒悟起来，其实assembleRelease和assembleDebug是2个继承于assemble动态生成的任务，这样以来只能用动态的方式建立以来关系，正确的做法如下：

```kotlin
// 2、建立release上传的任务依赖关系
val uploadReleaseTask = generateReleaseUploadTask(project)
project.tasks.whenTaskAdded {
    if ("assembleRelease" == it.name) {
        uploadReleaseTask.dependsOn(it)
    }
}
// 2、建立Debug上传的任务依赖关系
val uploadDebugTask = generateDebugUploadTask(project)
project.tasks.whenTaskAdded {
    if ("assembleDebug" == it.name) {
        uploadDebugTask.dependsOn(it)
    }
}
```

要在task动态添加的时候与其绑定先后顺序关系，按照上面的代码：

- uploadReleaseTask会在assembleRelease之后执行
- uploadDebugTask会在assembleDebug之后执行

这样，就能在uploadReleaseTask和uploadDebugTask中写自己的逻辑了

#### aar输出路径

这一步也是，我以为也会很麻烦，但其实当知道是`assembleRelease`还是`assembleDebug`的时候其实首先就能确认后缀是`-release.aar`还是`-debug.aar`，剩下的基本就是拼接工作了

```kotlin
val debugPath = "${project.buildDir.absolutePath}${fileSe}outputs${fileSe}aar${fileSe}${project.name}-debug.aar"
val releasePath = "${project.buildDir.absolutePath}${fileSe}outputs${fileSe}aar${fileSe}${project.name}-release.aar"
```

#### 上传arr

上传这一步就更简单了，我们分2种情况讨论：

- 上传到自己定义的位置

  这里需要具体实现uploadDebugTask中@TaskAction注解的方法，使用Okhhtp上传文件到服务器即可（模版代码一大堆）

- 上传到maven

  这里就需要有一些讲究，但是后面想清楚之后也很简单

  1、首先需要对当前项目导入我们`学习2`中使用的插件`maven-publish`

  ```kotlin
  class MyStudyPlugin: Plugin<Project> {
    override fun apply(project: Project) {
      project.plugins.apply("maven-publish")
    }
  }
  ```

  2、配置仓库

  这里直接参考`学习2`中的仓库参数配置，然后再串一下task就好了
  ```kotlin
  val pubTask = project.tasks.findByName("publishAarPublicationToMavenLocal")
  uploadDebugTask.finalizedBy(pubTask)
  uploadReleaseTask.finalizedBy(pubTask)
  ```

### 总结

这一次的学习过程还是很曲折的，Gradle的版本更新非常的快，而且很多博客的内容都比较老，要么是已经无法实现了，要么就是实现的方式过于复杂，Gradle插件本身就是一个流水线化的流程清晰的工具，对于组件化项目，非常不推荐各自在module的build.gradle中写个性化的东西。

