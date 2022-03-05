## APT学习笔记

### 学习背景

Annotation Processing Tool注解处理工具，在实际的框架搭建和流程化功能处理时都需要使用这个技术，例如组件化项目中跨组件的路由框架，简化代码量的工具类注解。需要注意的是，当使用注解框架完成一项功能有性能影响时，需要权衡注解的使用。本次对注解的学习主要侧重在源码即注解的路由框架实现上，同时了解一些附带的开发流程和知识。

### 学习目标

自定义注解，通过注解参数生成被注解类与注解参数的配置文件

### 项目结构

注解功能在项目结构上通常分为2部分，annotation和processor，使用AS开发时直接通过创建JavaLibrary创建2个Module来实现功能。

```
|-my-annotation // 单纯作为定义@Interface的的lib
|-my-processor // 处理不同的@Interface逻辑的库
```

为什么要分开为2个库呢？

看上面的注释就知道，分开库利于维护，需要修改解释器逻辑的时候可以只修改processor，活着重新定义新的processor来对注解功能进行不同的诠释。

### 项目配置

#### my-annotation

没有特殊的配置要求，可以把它视为一个接口库

#### my-processor

```groovy
dependencies {

    annotationProcessor 'com.google.auto.service:auto-service:1.0.1'
    implementation 'com.google.auto.service:auto-service:1.0.1'
    implementation project(path: ':my-annotation')

}
```

- 为什么auto-service再annotationProcessor依赖之后还需要implementation？

  如果不使用implementation在编译时会报错：`错误: 程序包com.google.auto.service不存在`

- auto-service作用？

  注解处理器实际上是需要注册的，最原始的方式就是手动注册，auto-service注解就是用来替代手写的过程，在注解处理器的类上加上类注解`@AutoService(Processor.class)`即可，最后生成的jar包中就会有这么一个文件：

  ```
  my-processor
  |-com
  |-META-INF
  	MANIFEST.MF
  	|-services
  		javax.annotation.processing.Processor
  ```

  javax.annotation.processing.Processor中的内容其实就是被`@AutoService`注解的全路径名

### 注解类

```java
@Inherited
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface KLRouter {
    String[] value() default {};
}
```

下面逐条分析一下这个注解类里的知识点

#### 元注解

**@Inherited（继承）：**@KLRouter注解了一个类A后，类A的子类B会自动继承被@KLRouter注解这一特性。需要注意的是A和B都不能是接口类，否则extends 或 implements后@Inherited将失效

**@Retention（保留）：**指定了注解的三种时机类型

- RetentionPolicy.SOURCE  源代码注解，参与编译，但是编译后会被VM去除
- RetentionPolicy.CLASS  字节码注解，参与编译，但是编译后不会被VM去除
- RetentionPolicy.RUNTIME  运行时注解，VM将在运行期也保留注释，可以通过反射机制读取注解的信息

**@Target（目标）：**注解应用的对象，可以多选：对类、方法、变量等等。一般来说不建议选太多，不同的处理逻辑可以选择新建一个注解处理，但尽量不要全部混在一起使用

#### 注解参数

注解在使用的时候都是下面的使用方式

```java
@KLRouter({"1","2"})
// 或者
@KLRouter("1")
```

如果注解变为

```java
public @interface KLRouter {
    String[] tag() default {};
}
```

那么使用时就必须带上`tag=`，匿名参数只对value()可用，其他的必须声明参数名

```java
@KLRouter(tag = {"1","2"})
```

最后需要注意的是，对于所有参数的声明最好是全部有默认值，尽量保证参数不会为null

### 处理器

```java
@AutoService(Processor.class)
@SupportedOptions({KLProcessor.MODULE_NAME, KLProcessor.BUILD_DIR})
@SupportedAnnotationTypes({"com.kevin.annotation.KLRouter"})
public class KLProcessor extends AbstractProcessor {
  	public static final String MODULE_NAME = "moduleName";
    public static final String BUILD_DIR = "buildDir";
}
```

#### 元注解

**@AutoService：**这个上面已经解释过了，这里就不再重复记录

**@SupportedOptions：**用来接收参数，这里一般是跟gradle配合使用的，下面晒一段代码就能完全理解了

```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [moduleName: project.getName(),
                             buildDir: project.getBuildDir().getAbsolutePath()]
            }
        }
    }
}
```

在注解运行时，便可以通过一下方式获取到传入的参数：

```java
@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
	String moduleName = processingEnv.getOptions().get(MODULE_NAME);
	String buildDir = processingEnv.getOptions().get(BUILD_DIR);
}
```

这样就能获取到当前项目module的名称，以及build文件夹的绝对路径

**@SupportedAnnotationTypes：**说明注解处理器将会处理哪些注解

#### 主要代码

```java
@AutoService(Processor.class)
@SupportedOptions({KLProcessor.MODULE_NAME, KLProcessor.BUILD_DIR})
@SupportedAnnotationTypes({"com.kevin.annotation.KLRouter"})
public class KLProcessor extends AbstractProcessor {
  private static final String FILE_SUFFIX = "_module_router.json";
  public static final String MODULE_NAME = "moduleName";
  public static final String BUILD_DIR = "buildDir";

  private Messager messager;
  private String routerFilePath;
  private Map<String, String[]> clzRouteMap;
  
  @Override
  public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    messager = processingEnv.getMessager();
    // 首先获取配置
    Map<String, String> optionMap = processingEnv.getOptions();
    if (optionMap != null) {
    	String moduleName = processingEnv.getOptions().get(MODULE_NAME);
      String buildDir = processingEnv.getOptions().get(BUILD_DIR);
      if (checkString(moduleName) && checkString(buildDir)) {
         routerFilePath = buildDir + File.separator + "tmp" +
           File.separator + "klrouter" +
           File.separator + moduleName + FILE_SUFFIX;
      }
    }

    messager.printMessage(Diagnostic.Kind.NOTE, "file path:" + routerFilePath);
    clzRouteMap = new HashMap<>();
  }
  
  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
      messager.printMessage(Diagnostic.Kind.NOTE, "===== process =====");
      if (annotations == null || annotations.isEmpty() || !checkString(routerFilePath)) {
          return false;
      }
      messager.printMessage(Diagnostic.Kind.NOTE, "===== start =====");
      Set<? extends Element> els = roundEnv.getElementsAnnotatedWith(KLRouter.class);
      if (els == null || els.isEmpty()) {
          messager.printMessage(Diagnostic.Kind.NOTE, "KLRouter has no annotation elements");
          return false;
      }
      Map<String, String[]> addedMap = new HashMap<>();
      for (Element e : els) {
          if (e == null || e.getKind() != ElementKind.CLASS) {
              continue;
          }
          TypeElement typeEle = (TypeElement) e;
          // 类的全路径名作为key
          String clzFullName = typeEle.getQualifiedName().toString();
          if (!checkString(clzFullName) || clzRouteMap.containsKey(clzFullName)) {
              continue;
          }
          // 注解值的数组作为value
          KLRouter ann = typeEle.getAnnotation(KLRouter.class);
          String[] routes = ann.value();
          if (routes.length > 0) {
              messager.printMessage(Diagnostic.Kind.OTHER, "clz name:" + clzFullName);
              addedMap.put(clzFullName, routes);
          }
      }

      saveRouteConfig(addedMap);

      return true;
  }

  private void saveRouteConfig(Map<String, String[]> addedMap) {
      // 没有新增的路由配置
      if (addedMap.isEmpty()) {
          return;
      }
      File file = new File(routerFilePath);
      if (file.length() <= 0 || clzRouteMap.isEmpty()) {
          clzRouteMap.putAll(addedMap);
          writeFile(file, 0L, clzRouteMap);
      } else {
          writeFile(file, file.length() - 1, addedMap);
          clzRouteMap.putAll(addedMap);
      }
  }
  
  @Override
  public SourceVersion getSupportedSourceVersion() {
    return processingEnv.getSourceVersion();
  }
}
```

- 注解处理类需要继承AbstractProcessor
- init方法智慧调用一次，即为单例模式
- process方法可能会调用多次，因为在编译过程中可能会有新的类产生，新类也可能使用了注解，但再次调用这个方法时，如果新产生的类没有使用处理器处理的注解，annotations将会为空

#### 处理注解

```java
Set<? extends Element> els = roundEnv.getElementsAnnotatedWith(KLRouter.class);
if (els == null || els.isEmpty()) {
    messager.printMessage(Diagnostic.Kind.NOTE, "KLRouter has no annotation elements");
    return false;
}
Map<String, String[]> addedMap = new HashMap<>();
for (Element e : els) {
    if (e == null || e.getKind() != ElementKind.CLASS) {
        continue;
    }
    TypeElement typeEle = (TypeElement) e;
    // 类的全路径名作为key
    String clzFullName = typeEle.getQualifiedName().toString();
    if (!checkString(clzFullName) || clzRouteMap.containsKey(clzFullName)) {
        continue;
    }
    // 注解值的数组作为value
    KLRouter ann = typeEle.getAnnotation(KLRouter.class);
    String[] routes = ann.value();
    if (routes.length > 0) {
        messager.printMessage(Diagnostic.Kind.OTHER, "clz name:" + clzFullName);
        addedMap.put(clzFullName, routes);
    }
}
```

##### 获取被注解的类

```java
Set<? extends Element> els = roundEnv.getElementsAnnotatedWith(KLRouter.class)
```

用来获取被`KLRouter`注解的类的集合，因为这个processor只处理这么一个注解，所以下面的代码在判断元素为空后就不用再处理了

##### TypeElement

```java
KLRouter ann = typeEle.getAnnotation(KLRouter.class);
if (e == null || e.getKind() != ElementKind.CLASS) {
    continue;
}
TypeElement typeEle = (TypeElement) e;
```

- 因为`KLRouter`是类注解，所以需要加必要的判断，否则下面直接强转`TypeElement`会崩溃

- 这里顺带记一个知识点，被注解的东西不同，Element的具体类型也不同，这里列举最常用的3个：

​		被注解类 -> TypeElement

​		被注解方法 -> ExecutableElement

​		被注解变量 -> VariableElement

- 通过getQualifiedName()获取被注解类的全名，例如：com.kevin.study.KLRouterTest

​		这里的`TestRouter`就是被注解的类

##### 提取配置

以下面的类为例：

```java
package com.kevin.study;

@KLRouter("firstPage")
public class KLRouterTest { }

@KLRouter({"r1","r2"})
public class KLTest { }
```

那么预期生成的配置就是：

```json
{"com.kevin.study.KLRouterTest":["firstPage"],"com.kevin.study.KLTest":["r1","r2"]}
```

即类全名作为key，注解入参作为value，那么如何获取呢？看完下面的代码想必就已经知道了

```java
TypeElement typeEle = (TypeElement) e;
// 类的全路径名作为key
String clzFullName = typeEle.getQualifiedName().toString();
// ...
KLRouter ann = typeEle.getAnnotation(KLRouter.class);
// 注解值的数组作为value
String[] routes = ann.value();
```

##### 保存配置

拿到key和value之后无非就是文件的写入逻辑了

```java
private void saveRouteConfig(Map<String, String[]> addedMap) {
      // 没有新增的路由配置
      if (addedMap.isEmpty()) {
          return;
      }
      File file = new File(routerFilePath);
      if (file.length() <= 0 || clzRouteMap.isEmpty()) {
          clzRouteMap.putAll(addedMap);
          writeFile(file, 0L, clzRouteMap);
      } else {
          writeFile(file, file.length() - 1, addedMap);
          clzRouteMap.putAll(addedMap);
      }
}
```

- clzRouteMap是一个全局变量，因为process会被调用多次，这个map就负责兜底，防止类被重复处理
- 如果有多个类被注解了，那么处理的的时候是把每个类的配置使用RandomAccessFile写入到文件后面，但它有一个非常难受的属性，虽然其能指定文件开始写入的位置，但是从这个位置开始，源文件后面所有的内容都会被擦除。所以还有一种方式，就是每次使用clzRouteMap写入配置，全量覆盖目标文件

最后文件会被写入到：`./build/tmp/klrouter/xxx_module_router.json`中，到这里注解的任务就完成了。

### 结束了？

这样放进去的文件会被打到aar中吗？会打到哪？

注解处理完之后，实际上文件无法被打到aar中，还需要在合适的时候交由gradle拷贝到正确的路径，然后被打入assets中，这一步的知识将在gradle学习笔记中进行记录。

### 反思

我写完这个注解之后，然后自己实际去使用了一下，发现遇到了一个大麻烦，由于使用的是源码级的注解，编译之后`KLRouterTest`变成了：

```java
package com.kevin.study;

public class KLRouterTest { }
```

我路由tag呢？我怎么知道想跳到`KLRouterTest`负责的全局页面使用的是`firstPage`呀？

这个问题其实只有实际使用的时候才会发现，只要把`@KLRouter`注解改为字节码级的就好了