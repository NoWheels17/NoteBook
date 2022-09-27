## 学习背景

Android SDK Tool自带了至少200项lint检查规则，这些规则一般都比较基础和通用，但是并不能完全满足自有项目的要求，所以需要自定义一些规则来维护项目的稳定性。例如资源名称、类名、方法名等检查，方法返回值检查等等。
Lint检查能在编码环节就提示问题，并给出规则检查不通过的详细说明，相比人为review限制更加高效。

## Android Lint介绍

Android Lint是一个静态代码分析工具，它能够对你的Android项目中潜在的bug、可优化的代码、安全性、性能、可用性、可访问性、国际化等进行检查。Android Lint内置了很多lint规则，总共可以分为以下几类：  

- Correctness 正确性
- Security 安全性
- Performance 性能
- Usability 可用性
- Accessibility 可访问性
- Internationalization 国际化

下面列举一些常见的lint会检测的代码问题：

- 缺少翻译（和未使用的翻译）
- 布局性能问题（老的layoutopt工具会用于查找所有这样的问题，和除此之外更多的问题）
- 未使用的资源
- 不一致的数组大小（当在多个配置中定义数组）
- 可访问性和国际化问题（硬编码字符串，缺少contentDescription等）
- 图标问题 （如丢失密度、 重复图标、 错误尺寸等）
- 可用性问题 （如不在文本字段上指定输入的类型）
- AndroidManifest错误

lint检查作为流程的一个环节把控代码质量时，需要再在合适的实际执行，一般会有以下几种时机可选： 
1、编码阶段IDE实时检查，第一时间发现问题
2、本地编译时，及时检查高优先级问题，检查通过才能编译
3、提代码时，CI检查所有问题，检查通过才能合代码
4、打包阶段，完整检查工程，确保万无一失

## 需要实现的主要API  

- Issue：表示一个Lint规则。例如调用Toast.makeText()方法后，没有调用Toast.show()方法将其显示。

- IssueRegistry：用于注册要检查的Issue列表。自定义Lint需要生成一个jar文件，其Manifest指向IssueRegistry类。

- Detector：用于检测并报告代码中的Issue。每个Issue包含一个Detector。

- Scope：声明Detector要扫描的代码范围，例如Java源文件、XML资源文件、Gradle文件等。每个Issue可包含多个Scope。

- Scanner：用于扫描并发现代码中的Issue。每个Detector可以实现一到多个Scanner。自定义Lint开发过程中最主要的工作就是实现Scanner。

  在比较新的版本的中，Scanner主要有如下几个：

  `SourceCodeScanner`、`UastScanner`：主要负责对源码的扫描，分析源码中的语句

  `ClassScanner`：负责字节码class文件的扫描

  `BinaryResourceScanner`：扫描二进制资源文件(如.png)

  `ResourceFloderScanner`：扫描资源目录

  `XmlScanner`：扫描 xml 文件

  `GradleScanner`：扫描 Gradle 文件脚本文件

  `OtherFileScanne`r：扫描其他文件

## Android Lint实现基本步骤

### gradle配置

```groovy
plugins {
    id 'java-library'
}

java {
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8
}

dependencies {
    compileOnly 'com.android.tools.lint:lint-api:30.3.0'
    compileOnly 'com.android.tools.lint:lint-checks:30.3.0'
}

jar {
    manifest {
        attributes("Lint-Registry-v2": "com.kevin.lint.CustomIssueRegistry")
    }
}
```

gradle配置为`java library`，

### IssueRegistry注册中心

注册中心主要负责管理所有的lint issue，所有需要使用检查规则都需要在这里注册，每项规则的检查都通过一个Detector实现

``` java
public class CustomIssueRegistry extends IssueRegistry {
	private static final List<Issue> sIssues;
	static {
		sIssues = new ArrayList<>;
		sIssues.add(MyCustomDecetor.ISSUE);
	}
}
```
### 自定义Decetor检查规则

```java
public class MyCustomDecetor extends Detector implements SourceCodeScanner {

    private static final String ISSUE_ID = "CustomDecetor";

    public static final Issue ISSUE = Issue.create(
            ISSUE_ID, //独一无二的id即可
            "简述",
            "具体解释",     // 描述信息
            Category.MESSAGES,
            5, // 优先级 1~10, 数字越大，优先级越高
            Severity.WARNING,
            new Implementation(MyCustomDecetor.class, Scope.JAVA_FILE_SCOPE)
      			//文件类型意味着只扫描java文件
    );
}
```

Detector的实现主要是对某个问题的判定，然后报告错误，每个issue对应一个decetor，在继承`Detector`后需要实现对应的`Scanner`，这个`Scanner`只是一个接口，规范了一些回调方法，具体还需要针对不同的方法实现具体的内容。

### 项目引用和执行检查

```groovy
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
		// ...
    lintChecks project(':my-lint') // 本地项目引用
		// ...
}
```

执行只需要双击Module的gradle task即可，假设module是`app`,task路径为：`app->Tasks->verification->lint`

## 自定义实战

最经典的问题：检查到`android.util.Log`时提示使用自定义的Log工具。接下来按照上面的接口实现

### 实现

- 新建JavaLibrary项目，配置gradle

  ``` groovy
  plugins {
      id 'java-library'
  }
  
  java {
      sourceCompatibility = JavaVersion.VERSION_1_8
      targetCompatibility = JavaVersion.VERSION_1_8
  }
  
  dependencies {
      compileOnly 'com.android.tools.lint:lint-api:30.3.0'
      compileOnly 'com.android.tools.lint:lint-checks:30.3.0'
  }
  
  // 这里manifest是最重要的，注册中心必须是正确全路径，这里跟自定义注解最初的时间很像
  jar {
      manifest {
          attributes("Lint-Registry-v2": "com.kevin.lint.KevinIssueRegistry")
      }
  }
  ```

- 新建`KevinIssueRegistry.java`继承`IssueRegistry`

  ```java
  package com.kevin.lint;
  
  import com.android.tools.lint.client.api.IssueRegistry;
  import com.android.tools.lint.detector.api.Issue;
  import org.jetbrains.annotations.NotNull;
  import java.util.ArrayList;
  import java.util.List;
  
  /**
   * Author: Klee  2022/9/20
   */
  public class KevinIssueRegistry extends IssueRegistry {
  
      private static final List<Issue> sIssues = new ArrayList<>();
  
      static {
        	// 自定义的Lint检测规则都可以在这里注册，建议自己的项目所有检查规则都在同一库里实现
          sIssues.add(UseAndroidLogIssue.ISSUE);
      }
  
      @NotNull
      @Override
      public List<Issue> getIssues() { // 通过这个回调，将所有问题检测都提供给注册中心
          return sIssues;
      }
  
  }
  ```

- 新建`UseAndroidLogIssue.java`继承`Detector`实现`SourceCodeScanner`，下面先晒源码，然后解释问题

  ``` java
  package com.kevin.lint;
  
  import com.android.tools.lint.detector.api.Category;
  import com.android.tools.lint.detector.api.Detector;
  import com.android.tools.lint.detector.api.Implementation;
  import com.android.tools.lint.detector.api.Issue;
  import com.android.tools.lint.detector.api.JavaContext;
  import com.android.tools.lint.detector.api.Scope;
  import com.android.tools.lint.detector.api.Severity;
  import com.android.tools.lint.detector.api.SourceCodeScanner;
  import com.intellij.psi.PsiMethod;
  import org.jetbrains.annotations.NotNull;
  import org.jetbrains.annotations.Nullable;
  import org.jetbrains.uast.UCallExpression;
  import java.util.Arrays;
  import java.util.List;
  
  /**
   * Author: Manolin.li@tuya.com  2022/9/21
   */
  public class UseAndroidLogIssue extends Detector implements SourceCodeScanner {
  
      private static final String ISSUE_ID = "UseLibLog";
  
      public static final Issue ISSUE = Issue.create(
              ISSUE_ID, //独一无二的id即可
              "不要直接使用Android Log", //描述信息
              "不要直接使用Android Log, 使用自定义的com.kevin.util.L", // 描述信息
              Category.MESSAGES,
              5,
              Severity.ERROR,
              new Implementation(UseAndroidLogIssue.class, Scope.JAVA_FILE_SCOPE)
      );
  
      @Nullable
      @Override
      //根据名称 去检查方法
      public List<String> getApplicableMethodNames() {
          return Arrays.asList("v", "d", "i", "w", "e");//Log.d()  Log.e() 等方法名
      }
  
      // 类似还有visitClass 包括Gradle的访问
      @Override
      public void visitMethodCall(@NotNull JavaContext context, @NotNull UCallExpression node, @NotNull PsiMethod method) {
          boolean isMemberInClass = context.getEvaluator().isMemberInClass(method, "android.util.Log");
          boolean isMemberInSubClassOf = context.getEvaluator().isMemberInSubClassOf(method, "android.util.Log", true);
          if (isMemberInClass || isMemberInSubClassOf) {
              context.report(ISSUE, node, context.getLocation(node), "不要直接使用Android Log");//report 上报提示信息给开发者
          }
      }
  
  }
  ```

  1、`getApplicableMethodNames`负责对源码中语句调用的方法进行监听，当发现有这个方法被调用时才触发`visitMethodCall`回调

  2、在`visitMethodCall`中得到的只是按照方法名过滤的代码，所以还需要

  ``` java
  boolean isMemberInClass = context.getEvaluator().isMemberInClass(method, "android.util.Log");
  boolean isMemberInSubClassOf = context.getEvaluator().isMemberInSubClassOf(method, "android.util.Log", true);
  ```

  判断是否确实是使用了android自带的log库

### 拓展

同样的方式，还可以针对`System.out.print`或者`System.out.println`进行检查。lint检查还可以协助对废弃api使用的提醒，让代码实现迁移到新的api上，不需要人工review的干预快速解决和定位问题。
