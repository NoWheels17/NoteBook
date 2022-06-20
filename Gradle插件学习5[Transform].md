## 自定义Transform

### 学习背景
开发过程中开发自测阶段经常需要修改某些组件的某些类的部分代码，或者是插入测试代码检测部分问题，但是这些测试或者调试使用的代码是不能进入正式包的。为了适应这种情况，最好得办法是使用插件在编译时完整代码插装的操作。

### 学习目标
了解相关基础概念，掌握开发流程和关键操作原理。

### Transform？
首先了解Transform的作用，`从class文件到dex文件装换期间`用来`修改class文件`一套标准API，这个流程是通过多个Tranform串联执行完成的，每个Transform处理完之后会把结果传给下一个Tranform去执行，混淆和语法糖的装换都是通过Transform去操作的。Tranform最终会被转换成一个一个的Task。
自定义的Trandform会放在标准化流程Tranform的最前面。
Tranform可以处理jar、aar、asset文件，这里多留意，可以留意asset这一点，说明插桩的地方不仅限于代码。

### 输入和输出
代码插桩的基本思想就是获取并识别目标文件，更改输出产物，所以输入和输出是最关键的2个概念
#### TransformInput（输入）
- DitectoryInput 是以源码形式参与编译的所有文件夹和文件，这里要注意文件夹也在集合中 
- JarInput 是以jar包和aar参与项目编译的所有三方库
#### TransformOutputProvider（输出）
编译后产物文件输出文件路径和相关信息

### 方法介绍
``` kotlin
package com.kevin.study

import com.android.build.api.transform.*

class TestTransform : Transform() {

    override fun getName(): String {
	
	}

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> {
     
    }

    override fun getScopes(): MutableSet<in QualifiedContent.Scope> {
       
    }

    override fun isIncremental(): Boolean {
        
    }

    override fun transform(transformInvocation: TransformInvocation?) {
        super.transform(transformInvocation)
    }

}
```
上面是kotlin写的自定义的Tranform类，上面的5个方法是必须实现的，下面对前4个方法的作用做一些说明

#### getName方法

返回tranform的名称，这里的名称并不是全名，与自定义task时设置名称一样，只是名称的一部分，作为名称的前缀。假如返回`kevin`，那么执行的时候transform转换成的task的名称将为`transformKevinxxxx`

#### getInputTypes方法

文件类型纬度，有两种枚举类型

- CLASSES 代表处理的 java 的 class 文件
- RESOURCES 代表要处理asset资源文件

这个其实就是选择需要处理的文件类型，因为是返回Set，所以支持多选

#### getScopes方法

文件范围纬度，官方文档 Scope 有 7 种类型：

1. EXTERNAL_LIBRARIES ： 只有外部库
2. PROJECT ： 只有项目内容
3. PROJECT_LOCAL_DEPS ： 只有项目的本地依赖(本地jar)
4. PROVIDED_ONLY ： 只提供本地或远程依赖项
5. SUB_PROJECTS ： 只有子项目
6. SUB_PROJECTS_LOCAL_DEPS： 只有子项目的本地依赖项(本地jar)
7. TESTED_CODE ：由当前变量(包括依赖项)测试的代码

如果要处理所有的class字节码，返回SCOPE_FULL_PROJECT

#### isIncremental方法

表示是否支持增量编译

- 返回true，表示支持增量编译，那么此时input的状态会有changed、removed、added、notchanged四种，除了notchanged，其他三种都有需要执行的对应动作

​		added、changed：重复执行自定义的操作，然后输出文件

​		removed：移除outputProvider获取路径对应的文件

- 返回false，则需要删除所有的output的文件，全部重新生成

### 注册Transform

```kotlin
class MyPluginPlugin: Plugin<Project> {
    override fun apply(project: Project) {
        val android = project.extensions.findByType(AppExtension::class.java)
        android.registerTransform(TestTransform())
    }
}
```

注册的流程很简单，主要功能基本都在TestTransform中实现

### transform方法

为什么这个方法单独拿出来说明呢，因为严格来说，这个方法才是自定义功能起点，上面在说明增量编译的的时候提及过几个状态，这里我们掏出一套模板代码

```kotlin
@Override
public void transform(TransformInvocation transformInvocation){
    Collection<TransformInput> inputs = transformInvocation.getInputs();
    TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
    boolean isIncremental = transformInvocation.isIncremental();
    //如果非增量，则清空旧的输出内容
    if(!isIncremental) {
        outputProvider.deleteAll();
    }	
    for(TransformInput input : inputs) {
        for(JarInput jarInput : input.getJarInputs()) {
            Status status = jarInput.getStatus();
            File dest = outputProvider.getContentLocation(
                    jarInput.getName(),
                    jarInput.getContentTypes(),
                    jarInput.getScopes(),
                    Format.JAR);
            if(isIncremental && !emptyRun) {
                switch(status) {
                    case NOTCHANGED:
                        break;
                    case ADDED:
                    case CHANGED:
                    	// 这2个方法需要单独实现
                        transformJar(jarInput.getFile(), dest, status);
                        break;
                    case REMOVED:
                        if (dest.exists()) {
                            // 这个方法需要单独实现
                            FileUtils.forceDelete(dest);
                        }
                        break;
                }
            } else {
                // 这个方法需要单独实现
                transformJar(jarInput.getFile(), dest, status);
            }
        }
        for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
            File dest = outputProvider.getContentLocation(directoryInput.getName(),
                    directoryInput.getContentTypes(), directoryInput.getScopes(),
                    Format.DIRECTORY);
            FileUtils.forceMkdir(dest);
            if(isIncremental && !emptyRun) {
                String srcDirPath = directoryInput.getFile().getAbsolutePath();
                String destDirPath = dest.getAbsolutePath();
                Map<File, Status> fileStatusMap = directoryInput.getChangedFiles();
                for (Map.Entry<File, Status> changedFile : fileStatusMap.entrySet()) {
                    Status status = changedFile.getValue();
                    File inputFile = changedFile.getKey();
                    String destFilePath = inputFile.getAbsolutePath().replace(srcDirPath, destDirPath);
                    File destFile = new File(destFilePath);
                    switch (status) {
                        case NOTCHANGED:
                            break;
                        case REMOVED:
                            if(destFile.exists()) {
                                // 这个方法需要单独实现
                                FileUtils.forceDelete(destFile);
                            }
                            break;
                        case ADDED:
                        case CHANGED:
                        	// 这2个方法需要单独实现
                            FileUtils.touch(destFile);
                            transformSingleFile(inputFile, destFile, srcDirPath);
                            break;
                    }
                }
            } else {
                // 这个方法需要单独实现
                transformDir(directoryInput.getFile(), dest);
            }
        }
    }
}

```

### ASM代码织入

到上面为止，基本上把自定义Transform的流程和代码书写的模版都已经有了，剩下的就是如何插入代码了，这里要牢记，我们修改的是编译之后的字节码，通过AndroidStudio看到的.class文件的内容并不是字节码真实的内容，这里需要**ASM Bytecode Viewer**插件来实现查看字节码。

顺着思路，如果我们需要把

```java
Public class Kevin {
    public static void showTips(Context context) {
        Toast.makeText(context, "Lee", Toast.LENGTH_SHORT).show();
    }
}
```

改为

```java
Public class Kevin {
    public static void showTips(Context context) {
        Log.i("TAG", "inser log code");
        Toast.makeText(context, "Lee", Toast.LENGTH_SHORT).show();
    }
}
```

那么就需要定位到`Kevin`这个类的`showTips`方法，然后把`Log.i("TAG", "inser log code");`代码插入到函数的第1行，但是这些操作需要在字节码层面进行，所以可以使用**ASM Bytecode Viewer**获取到目标的字节码，然后覆盖过去即可。

#### 流程模版

```groovy
static void insertLog(File inputFile, File outputFile) {
        def weavedBytes = inputFile.bytes
        ClassReader classReader = new ClassReader(bytes)
        ClassWriter classWriter = new ClassWriter(classReader,
                ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS)
		// 这里要自定义
        InsertLogClassVisitor classVisitor = new InsertLogClassVisitor(classWriter)
        try {
            classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES)
            weavedBytes = classWriter.toByteArray()
        } catch (Exception e) {
            println "Exception occurred when visit code \n " + e.printStackTrace()
        }

        outputFile.withOutputStream{
            it.write(weavedBytes)
        }
    }

```

其他的代码基本都是固定的，只是需要自己实现一个`InsertLogClassAdapter`。流程如下：通过文件的字节数组构造`ClassReader`，然后再通过它构造`ClassWriter`，`classReader.accept`用来触发写入操作，最终会得到新文件的字节数组，将其写入目标文件即可。

#### InsertLogClassVisitor

```java

public class InsertLogClassVisitor extends ClassVisitor implements Opcodes {

    public InsertLogClassAdapter(ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
        if ((ASMUtils.isPublic(access) && !ASMUtils.isStatic(access)) && 
                name.equals("showTips") && 
                desc.equals("(Landroid/content/Context;)V")) {
            methodVisitor = new InsertLogMethodVisitor(methodVisitor);
        }

        return methodVisitor;
    }
}
```

这里我们是通过`visitMethod`这个方法来监控我们需要插入代码的方法，判断满足：方法是public、为静态方法、方法名为showTips、方法描述为"(Landroid/content/Contextn;)V"。这个描述非常眼熟了，是此前在学习JNI的时候寻找java方法的描述方式，表示这个方法的入参为`android.content.Context`这个类，范围值为`void`。最后是把代码处理的这个活交给了InsertLogMethodVisitor这个自定义的方法观察者去实现。

#### InsertLogMethodVisitor

```java
public class InsertLogMethodVisitor extends AdviceAdapter {

    @Override
    protected void onMethodEnter() {
        super.onMethodEnter()
        // 这里写入插件显提示的代码语句
    }

}
```

### 其他字节码插桩框架

AspectJ：是一个代码生成工具，使用它定义的语法生成规则来编写，基本上要扫描所有的文件，当然AspectJx已经实现的非常的好。最主要的是有个坑，在抖音目前的多module的工程上，是有很多坑的，例如，和后面的Transform过程有一些冲突，导致代码一直织入失败，而且编译的时长也大大增加。
Javasist：直接操作修改编译后的字节码，而且可以自定义Transform，编译时长可以做很大空间的优化，就是织入代码的效率不如ASM。
