### 背景

​	在做项目的时候，原本只是跟着使用手册按部就班地实现了一个ModuleApp——A，然后按照先完成功能后打理结构的思维方式又实现了一个ModuleApp——B，紧接着就出问题了。非常尴尬，只有一个ModuleApp生效了，另外一个失效了（这个判断后来被去掉了），于是我点进XXRouter进去看了一下：

```java
// CODE_1
@Inherited
@Retention(RetentionPolicy.SOURCE) // <-- ??? 源码级注解
@Target(ElementType.TYPE)
public @interface XXRouter {
    String[] value() default {};
}
```

好奇心就上来了：

- 源码级的注解怎么实现的路由？这玩意儿都不参与编译
- 组件化怎么保证其他组件的ModuleApp路由类能被注解器管理到？

### 链路分析

#### 配置和触发

先收集一下目前的有效信息，从使用的流程分析其大致原理，然后从注解的实现原理上完成完整链路的分析。先看3段代码：

```java
// CODE_2
public abstract class ModuleApp {
    public abstract void route(Context context, String target, Bundle bundle, int requestCode);
}
```

```java
// CODE_3
@XXRouter({"custom"}) // 关联target和处理
public class CustomApp extends ModuleApp {
    @Override
    public void route(final Context context, String target, final Bundle bundle, final int requestCode) {
        if (TextUtils.equals(target, "custom")) {
          // jump to custom activity
          // context.startActivity...
        }
    }
}
```

```java
// CODE_4
// 发起跳转动作
UrlRouter.execute(UrlRouter.makeBuilder(context, "custom", bundle));
```

1、从使用上从使用上来看，注解起到的作用是关联路由target字符串和负责这个target处理的ModuleApp

2、通过UrlRouter来触发跳转动作，从已知信息看

#### Scheme处理

```java
// CODE_5
// UrlRouter
public static void execute(UrlBuilder urlBuilder) {
    SchemeService schemeService = MicroServiceManager.getInstance().findServiceByInterface(
                SchemeService.class.getName());
    schemeService.execute(urlBuilder);
}
```

这里主要是把这个urlBuilder交给了Scheme的服务实现类来处理，这里跳过Flutter和Redirect的拦截，直接走到的地方在下面：

```java
// CODE_6
// SchemeServiceImpl
private void go(UrlBuilder urlBuilder) {
  // 关键点在这里
    ModuleApp app = ModuleAppManager.getInstance().getModuleAppByTarget(urlBuilder.target);
    // ...
    app.route(urlBuilder.context, urlBuilder.target, urlBuilder.params, urlBuilder.requestCode);
}
```

这个地方就是体现实现原理的关键点。首先app.route明显就是对应了CODE_3处的回调方法，即触发了Activity跳转动作。而ModuleApp的具体实现者是通过target（"custom"）从ModuleAppManager中获取的，说明这个Manager处理和管理了通过源码注解关联的信息，即target->xxModuleApp，那再追一下数据的来源。

#### ModuleAppManager

##### 入口逻辑

```java
// CODE_7
private LruCache<String, ModuleApp> moduleApps = new LruCache<>(20);
private Map<String, String> moduleMap = new HashMap<>();

public ModuleApp getModuleAppByTarget(String target) {
  //...先获取target对应的全路径类名
    String moduleClass = moduleMap.get(target);
  //...然后通过类名获取到ModuleApp
    return getModuleAppByClass(moduleClass);
}

public ModuleApp getModuleAppByClass(String moduleClass) {
    ModuleApp app = moduleApps.get(moduleClass);
    if (app != null) {
        return app;
    }
  // 通过类加载器加载对应的ModuleApp
  	ModuleApp moduleApp = (ModuleApp) AppUtils.getClassLoader()
                .loadClass(moduleClass).newInstance();
    moduleApps.put(moduleClass, moduleApp);
    return moduleApp;
}
```

从主要的成员变量可以看出：

1、设定了一级LruCache最多缓存20个，用来直接存储ModuleApp实例（如果模块划分比较细，可能经常需要初始化缓存），同时也说明再使用ModuleApp的时候，不能在这个类里读写需要缓存的数据

2、有全量的target到类名的映射关系，moduleMap数据的来源便是分析链路的下一个关键点

##### 构造初始化

```java
// CODE_8
private ModuleAppManager() {
    Map<String, List<String>> moduleAppMap = ModuleConfigLoader.getInstance().getModuleApps();
    if (moduleAppMap != null) {
        Set<Map.Entry<String, List<String>>> entrySet = moduleAppMap.entrySet();
      //...
        for (Map.Entry<String, List<String>> entry : entrySet) {
          // ...
            for (String target : targets) {
                moduleMap.put(target, app);
            }
        }
    }
    //...
}
```

从构造方法里可以看到，moduleMap的值是ModuleConfigLoader提供的，loader提供的原始数据为Map，key为类的完整路径，value则是target的集合。

#### ModuleConfigLoader

```java
// CODE_9
public Map<String, List<String>> getModuleApps() {
    return mSchemeModel == null ? null : mSchemeModel.moduleMap;
}

private static final String APP_MODULE_FILE = "router_app.json";
private ModuleConfigLoader() {
  // ...
		InputStream is = MicroContext.getApplication().getAssets().open(APP_MODULE_FILE);
		reader = new JSONReader(new InputStreamReader(is, "utf-8"));
		mSchemeModel = reader.readObject(ModuleConfigModel.class);
  // ...
}
```

到这里就完全清晰了，原来是assets下面module_app.json写了所有的路由对应关系，到这里就分析完了。

完了？等等，我们的项目是组件化的，打包的用的aar已经没有了XXRouter这个源码注解了，那这个json文件怎么产生的？

### 组件化支持

通过寻找XXRouter.java的路径，我么发现在它xxxx-annotation:x.x.x这个依赖下，下面开始去注解的Processor中分析逻辑。这一节的主要内容就是探索router_app.json文件时如何产生的。

#### XXRouterProcess

```java
// CODE_10
TypeElement eType = (TypeElement) element;
String clazzName = eType.getQualifiedName().toString();
if (isAssignFromClass(eType, PARENT_CLASS)) {
    List<String> mActivies = new ArrayList<>();
    mConfigBean.getModuleMap().put(clazzName, mActivies);
    XXRouter router = getAnnotation(eType);
    final String keys[] = router.value();
    for (String key : keys) {
        if (!StringUtils.isEmpty(key.trim()))
            mActivies.add(key.trim());
    }
} 
```

从这里可以看到，获取到XXRouter注解作用的类之后，便读取注解的值。最终所有的target都会被放入keys[]中，这些key又被放在了mConfigBean.getModuleMap()这个map里面，key则是类的完整路径名称。到这里，这个@XXRouter注解的基本就发挥完其该有的作用了。下面继续分析被放入map中target信息与文件的关系。

#### ModuleProcessor

```java
// CODE_11
// AutoService会自动在META-INF文件夹下生成Processor配置信息文件
@AutoService(Processor.class)
// Options配置之后，表示需要从gradle那里承接需要的参数，gradle配置参见 CODE_12
@SupportedOptions({"moduleName", "moduleTmpPath")
@IncrementalAnnotationProcessor(ISOLATING)
// 指定JDK版本
@SupportedSourceVersion(SourceVersion.RELEASE_7)
// 执行需要处理的注解
@SupportedAnnotationTypes({"com.kevin.XXRouter"})
public class ModuleProcessor extends AbstractProcessor {
  	private ConfigUtil mConfigUtil;
    private ConfigBean mConfigBean;
  
  @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
      // ...
      mConfigUtil = ConfigUtil.create(options, messager);
      // ...
      List<BaseProcessor> ps = new ArrayList<>();
      ps.add(new XXRouterProcess(mConfigBean, mConfigUtil));
      for (BaseProcessor processor : ps) {
          processor.init(mEnv);
          processor.process(set, roundEnvironment);
      }
			// 把配置写成文件
      mConfigUtil.flushConfigBean(mConfigBean,mEnv.getFiler());

    }

}
```

```groovy
// CODE_12
// java配置
android {
  defaultConfig {
    javaCompileOptions {
      annotationProcessorOptions {
        arguments = [moduleName: project.getName(), moduleTmpPath: ""] // key:value, key2:value2...
      }
    }
  }
}
// kotlin 配置
apply plugin: 'kotlin-kapt'
kapt{
  arguments {
    // 参数1
    arg("moduleName": project.getName())
    // 参数2
    //arg("moduleTmpPath": project.getName())
  }
}

dependencies {
  // java对注解的依赖
  annotationProcessor 'abc:1.2.3'
  // kotlin对注解的依赖
  kapt 'abc:1.2.3'
}
```

从这里我们发现其实ModuleProcessor才是真正的注解处理器，而且把注解的逻辑分给了XXRouterProcess，等XXRouterProcess处理完之后会把结果保存在mConfigBean中，最后通过mConfigUtil写到文件里，那文件最终去哪里了呢？

#### 路由合并

实际上这个编译时注解产生的路由文件，被打到aar中了，在组件化打包的最后，会有gradle插件专门符合合并assets中的这些以

***_router_module.json***

结尾的文件，至于gradle插件合并assets中文件的操作，预计将分到gradle插件学习的计划中。

### 总结

整个跨组件的路由框架运作流程如下：

1、编译时通过源码注解生成json文件，放在assets中，json文件中存放是的类路径作为key，注解数组值作为value的数据。

2、在所有组件打包成apk时，由专门的gradle插件合并“_router_module.json”结尾的文件中的内容，并将合并得到的文件router_app.json打到apk的assets中

3、app运行起来之后，在路由的时候，会通过ModuleConfigLoader读取配置，获取到target对应的路由处理类xxxModuleApp，最终触发实际的Activity跳转动作

涉及到自己没有接触的新知识点：

1、APT给注解处理类传递参数的配置，koltin和java有差别

2、编译时新增assets文件的技巧

