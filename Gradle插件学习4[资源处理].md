## assets资源处理和实战

### 学习背景

因为项目是组件化的，服务和路由功能都是跨组件的能力，为了解耦很多时候都希望能在不依赖核心逻辑处理库的情况下实现这些功能。这些自动化的配置基本都是由注解完成了，但是注解的产物最终需要在打包时做整理，这就需要学习如果在打包时处理资源，这里先从入门的assets资源处理入手。

### 学习目标

了解assets文件处理时机，完成文件合并，掌握多层Extension的设置方式，支持自定义合并功能

### 书接上回

在APT学习的尾章，我只是把生成的json配置文件放在了`xxx/build/tmp/klrouter/`下，接下来的问题就交给gradle去做了。那么这里就有个为什么需要注意。

- 为什么不把生成的json放在项目的assets目录下，这样岂不是就不需要挪文件了？

  先解释可不可行，其实是可行的，但是配置是在第一次编译时后才能生成，需要在git记录中提交变更。如果是把生成的配置放在`xxx/build/`下，配置文件会打入aar中，但是不会有其他可感知的变更，相对于前一种json放入项目assets中，这里选择学习的时候增加一点难度，选择后者。

### Aar配置处理

在注解生成完配置文件放在`xxx/build/`下后，我发现其实json并不会被打到aar中。其实在打包的时候，需要被打包到aar或者apk中的文件都有固定的路径，当处理完所有的资源和字节码后，这些文件都会被一起打包到，所以不管是aar还是apk，如果在编译期间新增assets文件，则需要把这些文件想办法放到正确的目录下。此前在学习task的时候，就知道了可以自定义task然后放在正确的时机去执行，那么问题来了。

#### 拷贝时机

```
[release]
> Task :gradle-study1:bundleReleaseAar

[debug]
> Task :gradle-study1:bundleDebugAar
```

以打release为例，在打包aar的时候，其实可以发现有bundleReleaseAar任务，每次都会执行，所以时机最好是选在bundleReleaseAar之前把文件拷贝到正确的目录下，分2步实现（此前已经全部掌握）：

```groovy
static def copyFile(String src, String dest) {
    def s = new File(src)
    if (!s.exists()) {
       return
    }
    def d = new File(dest)
    if (d.exists()) {
        d.delete()
    }

    s.withReader {reader->
        def lines = reader.readLine()
        d.withWriter {writer->
            lines.each {
                writer.append(it + "\n")
            }
        }
    }
}

afterEvaluate {
    tasks.findByPath("bundleReleaseAar").doFirst {
        copyFile("xxx","yyy")
    }
}
```

这样就能在bundleReleaseAar执行前把文件拷贝到目标目录了

#### 目标目录

其实这一步也很简单，通过实际添加一个assets文件，然后打包再搜索build目录下同名文件，就能知道文件在：

[release]  `build/intermediates/library_assets/release/out`

[debug]   `build/intermediates/library_assets/debug/out`

这里折腾完了，开发aar的时候就能搞定复制文件的功能了，剩下的就是apk打包时的aar合并了

### 风味与变体

```groovy
flavorDimensions('store', 'v')

productFlavors {
    xm{
        dimension 'store'
    }
    hw {
        dimension 'store'
    }
    v1 {
        dimension 'v'
    }
    v2 {
        dimension 'v'
    }
}
```

如果配置了上面的风味，将发生一个恐怖的事情，apk在合并aar中assets文件时会出现以下不同的8种task，而且输出的目录也有8个

```
> Task :gradle-study1:mergeXmV1ReleaseAssets
> Task :gradle-study1:mergeXmV2ReleaseAssets
> Task :gradle-study1:mergeHwV1ReleaseAssets
> Task :gradle-study1:mergeHwV2ReleaseAssets
> Task :gradle-study1:mergeXmV1DebugAssets
> Task :gradle-study1:mergeXmV2DebugAssets
> Task :gradle-study1:mergeHwV1DebugAssets
> Task :gradle-study1:mergeHwV2DebugAssets
```

这就意味着按照上面aar配置处理的方式，脚本逻辑必须跟着变体更新，这也太麻了。（aar可以这么写是因为aar不支持变体）

#### 通配逻辑

面对变体带来的问题，有没有什么办法能获取到所有的assets合并任务，以及assets资源存放位置呢？

有的，而且专门对assets资源合并任务对外提供了api：

```kotlin
val androidExt = it.extensions.findByName("android") as AppExtension
androidExt.applicationVariants.all { variant ->
		val variantTask = variant.mergeAssetsProvider.get()
    val taskName = "${variantTask.name}Custom"
    val workPath = variantTask.outputDir.asFile.get().absolutePath
    val mergeTask = project.tasks.create(taskName, MergeAssetFilesTask::class.java, workPath, handler)
    variantTask.finalizedBy(mergeTask)
}
```

通过获取`android{}`这个Extension，然后拿到变体集合`applicationVariants`，通过.all遍历所有的变体集合，这样就能获取到2个关键的对象：

- 每个变体对应的assets文件合并task
- 每个变体将会被打到apk中assets的临时存放目录

有上面2个关键信息之后，接下来需要做的就是：

- 将目录传入给合并文件的`mergeTask`
- 安排顺序，使`mergeTask`优先于mergeAssets对应的task执行

### 开放设计

文件合并的过程比较简单，这里的重点是如何设计合并task的工作流以及支持能力的开放，先看下面几行代码：

```kotlin
project.extensions.create(MergeHandler::class.java, "mergeHandler", DefaultMergeHandler::class.java)
val handler = project.extensions.findByType(MergeHandler::class.java)
project.tasks.create("taskName", MergeAssetFilesTask::class.java, "workPath", handler)
```

可以看到我们构建了一个拓展`mergeHandler`（类似此前学习的`upInfo`或者常见的`android`），然后将它作为作为参数传给MergeAssetFilesTask，那么`mergeHandler`就是设计中对外开放的入口了。这个task的内部反而没有太多的东西

```kotlin
open class MergeAssetFilesTask// Use the factory ...
@Inject constructor(workPath: String?, handler: MergeHandler?) : DefaultTask() {

    @Input
    var dirPath: String? = workPath

    @Input
    var mergeHandler: MergeHandler? = handler

    override fun getGroup(): String? {
        return "kevin"
    }

    @TaskAction
    fun mergeAssetsWithHandler() {
        if (dirPath.isNullOrEmpty()) {
            println("MergeAssetFilesTask- merge assets output path is null or empty")
            return
        }

        if (mergeHandler == null) {
            println("MergeAssetFilesTask- mergeHandler is null")
            return
        }

        val dirFile = File(dirPath!!)
        if (!dirFile.exists()) {
            println("MergeAssetFilesTask- merge assets output dir not exists: $dirPath")
            return
        }

        val files = dirFile.listFiles(mergeHandler?.getFileFilter())
        mergeHandler?.getFileMerger()?.doMerge(files, dirPath)

    }

}
```

其实可以看到主要的逻辑全部委托给mergeHandler来处理了，但是问题来了：

- mergeHandler不是一个可空对象吗？项目build.gradle中未配置的岂不是就无法完成逻辑了？

```kotlin
project.extensions.create(MergeHandler::class.java, "mergeHandler", DefaultMergeHandler::class.java)
```

并不是这样的，上面这行代码其实说明了，`mergeHandler`会默认创建一个DefaultMergeHandler，那么默认的逻辑就需要在这里实现了。

#### 嵌套Extension

设计的初期，预计Extension大概是下面这个样子

```groovy
mergeHandler {
  	filter = xxx
  	merger = yyy
}
```

这样就出现了嵌套参数，那如何实现这种嵌套呢？先上源码

```kotlin
open class DefaultMergeHandler: MergeHandler {
    @Input
    var filter: FileFilter = DefaultFileFilter("_router.json")
    @Input
    var merger: FileMerger = DefaultJsonMerger("app_router.json")
    override fun getFileFilter(): FileFilter {
        return filter
    }
    override fun setFileFilter(filter: FileFilter) {
        this.filter = filter
    }
    override fun fileFilter(configure: Action<in FileFilter>) {
        configure.execute(filter)
    }
    override fun getFileMerger(): FileMerger {
        return merger
    }
    override fun setFileMerger(merger: FileMerger) {
        this.merger = merger
    }
    override fun fileMerger(configure: Action<in FileMerger>) {
        configure.execute(merger)
    }
}
```

- @Input 注解入参
- 声明闭包和set两种自定义方式

#### 自定义

从上看可以看出，在设计的时候支持自定义的`FileFilter`和`FileMerge`，那么该如何使用呢？

```groovy
class CustomMerger implements FileMerger { ... }

def customMerger = new CustomMerger()
def customFilter = new FileFilter() {
    @Override
    boolean accept(File pathname) {
        return pathname.getName().endsWith("_router.json")
    }
}

mergeHandler {
    fileFilter = customFilter

    fileMerger = customMerger
}
```

演示到这里，其实代码无法执行，因为还需要一个import：

```groovy
import com.kevin.study.FileMerger
```

实际在使用的时候，其实发现体验不是很好，由于FileFilter是java库中的接口，所以在实现的时候，有自动补齐提示，但是FileMerger却不行，正确的代码只是能保证编译通过和正常运行，但是无法由编辑器提示完成编码。

### 总结

### 关于library中assets的copy

```kotlin
private fun library(project: Project) {
        val libraryExt = project.extensions.findByName("android") as LibraryExtension

        val args = HashMap<String, String>()
        args["moduleName"] = project.name
        args["buildDir"] = project.buildDir.absolutePath
        libraryExt.defaultConfig.javaCompileOptions.annotationProcessorOptions.arguments = args

        val fileName = project.name + "_module_router.json"
        val tmpDirPath = project.buildDir.absolutePath + File.separator + "tmp" + File.separator +
                "klrouter" + File.separator

        project.afterEvaluate {
            libraryExt.libraryVariants.all { variant ->
                val variantTask = variant.mergeAssetsProvider.get()
                val workPath = variantTask.outputDir.asFile.get().absolutePath
                val baseName = variant.baseName
                val bName = baseName[0].toUpperCase() + baseName.substring(1, baseName.length)

                project.tasks.findByName("bundle${bName}Aar")?.doFirst {
                    copyFile(tmpDirPath + fileName, workPath + File.separator + fileName)
                }
            }
        }
    }

    private fun copyFile(src: String?, des: String?) {
        if (src.isNullOrEmpty() || des.isNullOrEmpty()) {
            return
        }
        val srcFile = File(src)
        if (srcFile.exists()) {
            val desFile = File(des)
            if (desFile.exists()) {
                desFile.delete()
            }
            srcFile.copyTo(desFile, true)
        }
    }
```

写完上面的发现，其实单纯在AndroidLibrary中拷贝assets也是可以通过通用的方式去实现的，文件拷贝依旧是原逻辑。只是在获取目标文件夹和编译类型的时候不需要全路径手写。

