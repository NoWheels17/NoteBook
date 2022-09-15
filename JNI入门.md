## 学习初衷

-> 了解一下，知道有这么个东西

-> 面试拓展知识，也不知道啥时候会用到

-> 飞利浦项目需要一个加密隐私信息的策略

-> 较系统学习，掌握一个小技巧

## 开始第一个JNI

### 配置

-> 没那么多文件，java、cpp、gradle、cmake

![目录](./目录.png)

cpp、java：负责代码逻辑

build.gradle：配置CMakeLists.txt路径，so库架构

CmakeLists.txt：so编译配置

-> gradle配置解释

```groovy
android {
......
    defaultConfig {
	......
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++11" // 选择c++版本，有一些新特性在低版本没有，比如auto类型，nullptr
                abiFilters "arm64-v8a", "armeabi-v7a", "x86_64"
            }
        }

    }

    externalNativeBuild {
        cmake {
            path "CMakeLists.txt" // 相对路径，这里表示这个文件gradle同级
        }
    }
......

}
```

【问题】

nullptr：为什么用nullptr，主要是为了统一关于NULL的定义，C语言中NULL是一个指针，地址为0；C++就是定义的整数0



```c++
// c
#define NULL ((void*)0)
// c++
#define NULL 0
```

-> CMakeLists.txt配置

```shell
# 选定把 src/main/cpp/code-util.cpp 打成libcode-util.so
add_library( demo
             SHARED
             src/main/cpp/demo.cpp )
# 下面2句话就是表示code-util依赖了log库，log库的作用其实就是输出Android的log日志的
find_library( log-lib
              log )
target_link_libraries( demo
                       ${log-lib} )
```

-> Java文件

```java
public class JNILibrary {

    // 加载so库
    static {
        System.loadLibrary("demo");
    }

    // 定义native方法
    public static native String demo(Context context);

}
```

-> cpp文件

```c++
#include <jni.h>
#include <android/log.h>
extern "C" {
#include "custom.h"
}

extern "C" { // 括号里面的内容按照C语言的方式编译和链接

    JNIEXPORT jstring JNICALL Java_com_kevin_demo_JNILibrary_demo
    (JNIEnv *env, jclass clazz, jobject context) {
        __android_log_print(ANDROID_LOG_INFO, "DemoJni" , "1122332 %s", "");
        return env->NewStringUTF("778899");
    }

}
```
【问题】
c++对c文件的引用，在导入头文件custom.h时需要用extern "C"{}给括起来，同样的，c代码也是

动态注册，推荐使用

```c++
#include <jni.h>
#include <android/log.h>

jstring demo(JNIEnv *env, jclass clazz, jobject context) {
        __android_log_print(ANDROID_LOG_INFO, "DemoJni" , "1122332 %s", "");
        return env->NewStringUTF("778899");
}

// 动态加载模块
jint jniRegisterNatives(JNIEnv *env) {
    jclass clz = env->FindClass("com/kevin/demo/JNILibrary");
    if (clz == nullptr) {
        return JNI_FALSE;
    }
    JNINativeMethod gMethods[] = {
            {"demo", "(Landroid/content/Context;)Ljava/lang/String;", (jstring *)demo}};

    if(env->RegisterNatives(clz, gMethods, 1) < 0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}

jint JNI_OnLoad(JavaVM *vm, void *unused) {
    JNIEnv *env = nullptr;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    if (env == nullptr) {
        return JNI_ERR;
    }

// 注册native方法
    if (!jniRegisterNatives(env)) {
        return JNI_ERR;
    }

    return JNI_VERSION_1_6;
}
```



### 输出

![输出err](./输出err.png)

看到很多博客都说编译之后的so库从这个路径里获取，那么对吗？

不对，其实正确的路径是这里：

![输出right](./输出right.png)

【问题】

1、为什么这个目录输出是正确的？有啥区别？

stripped目录下的so库是压缩过的

![目录对比](./目录对比.png)

那么删掉了哪些东西？

SYMBOL TABLE，符号表。有啥用？具体的定义大家可以去搜，大致可以理解为记录了源码和编译后文件的对应关系。

那AS生成未压缩的文件干嘛？

未压缩的文件，保留了符号表，可以用来debug断点调试。

2、为啥你有2个so？怎么配置的？

```shell
add_library(
            code-util
             SHARED
            src/main/cpp/code-util.cpp )

add_library(
        demo
        SHARED
        src/main/cpp/demo.cpp )

find_library(
              log-lib
              log )

target_link_libraries(
                       code-util
                       ${log-lib} )

target_link_libraries(
        demo
        ${log-lib} )
```

同样的，可以把个cpp打成1个库

```shell
add_library(
            code-util
             SHARED
            src/main/cpp/code-util.cpp src/main/cpp/demo.cpp )

find_library(
              log-lib
              log )

target_link_libraries(
                       code-util
                       ${log-lib} )

```

## JNI调用java

为啥要在在JNI里面调java？举一个很简单的场景，获取assets中的文件，c玩不了。

### 常用操作

```java
new String().toCharArray();
```

```c++
// 准备阶段
jclass clz_str = env->FindClass("java/lang/String"); // 加载class类
jmethodID new_obj = env->GetMethodID(clz_str, "<init>", "()V"); // 找到构造方法的直接引用
jmethodID to_char = env->GetMethodID(clz_str, "toCharArray", "()[C"); // 找到toCharArray方法的直接引用
// 执行阶段
jobject str = env->NewObject(clz_str, new_obj); // new 对象
jobject char_array = env->CallObjectMethod(str, to_char); // 执行方法
// 内存回收
env->DeleteLocalRef(clz_str);
env->DeleteLocalRef(str);
```

### 类型转换

因为网上能快速找到，这里就举个例子说明一下什么是类型转换

```c++
const char *nativePkg = env->GetStringUTFChars(packName, JNI_FALSE); // jstring -> char*
env->NewStringUTF("jstring"); // char* -> jstring
```

但是需要注意的是，java和c互转的时候，一定要保证规格一致，比如java的char是16位的，无法直接转到C语言中8位的char

### 局限性(目前我自己实战发现的)

```java
public interface IGunAdapter {
    void onOpenFire();
}

public class Silencer implements IGunAdapter {
    void onOpenFire() {
        // low voice
    }
}

public class Gun {
    private IGunAdapter mAdapter;
    public Gun(IGunAdapter adapter) {
        mAdapter = adapter;
    }
}
```

```c++
jmethodID new_obj = env->GetMethodID(clz_gun, "<init>", "(Lcom/kevin/demo/IGunAdapter;)Lcom/kevin/demo/Gun");
env->CallObjectMethod(clz_gun, new_obj, silencer); // silencer 为 Silencer实例，这里编译时会报错
```



## 实用案例

### so使用校验和防止重新打包APP

原理介绍

1、调用java的方法获取SHA1签名信息，这个签名信息时打包时赋予的，关键点就是校验打包者赋予的信息

2、获取so中获取包名后，经过一定的处理（比如取反，高低位交换等），将处理后的包名与已经保存的字符串做对比，匹配上了则继续，匹配不上则杀死当前进程，退出APP

【问题】

1、上来就问个问题，反正是调Java代码，为啥不直接在java层做？

因为包被反编译之后，得到的smali文件相当好修改，只要对照class文件把校验的代码处理一下直接重新打包就能绕过去了，成本比较低，在so里就要麻烦很多

2、那这个处理后的SHA1公版保存在哪？

用ida看了一下公版的jnimain，没有看到任何跟获取SHA1相关的操作，怀疑是放在安全图片里了



继续讲原理，先看对照用的java代码，理解调用流程

```java
public static String getPackageSign(Context context) {
    String signStr = "";
    if (context != null) {
        //获取包管理器
        PackageManager packageManager = context.getPackageManager();
        PackageInfo packageInfo;
        //分版本获取SHA1签名信息
        Signature[] signatures = null;
        try {
            if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
                packageInfo = packageManager.getPackageInfo(context.getPackageName(), PackageManager.GET_SIGNING_CERTIFICATES);
                SigningInfo signingInfo = packageInfo.signingInfo;
                signatures = signingInfo.getApkContentsSigners();
            } else {
                packageInfo = packageManager.getPackageInfo(packageName, PackageManager.GET_SIGNATURES);
                signatures = packageInfo.signatures;
            }
        } catch (Exception e) {
            e.printStackTrace();

        }
        if (null != signatures && signatures.length > 0) {
            signStr = signatures[0];
        }
    }
    return signStr;
}
```

check.cpp

```c++
// 这个示例里为了方便，SHA1签名是直接定义在了文件里，一般不这么做，最好是学习公版放在安全图片里，当然原则还是客户端没有绝对的安全，只是多给有心人添堵
#include <jni.h>
#include <string>

const char CHARS[16] = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'};

const char *SHA1 = "1a16d1cbb01c8873d9c4609fc82c7b3e533a1196";

bool is_lower_P() {
    char ver[3] = {0};
    __system_property_get("ro.build.version.sdk", ver);
    return strtol(ver, nullptr, 10) < 28;
}

bool getSHA1SignAndCheck(JNIEnv* env, jbyteArray sign_byt_ary) {
    jclass md_clz = env->FindClass("java/security/MessageDigest");
    jmethodID md_get_mtd = env->GetStaticMethodID(md_clz, "getInstance",
            "(Ljava/lang/String;)Ljava/security/MessageDigest;");
    jobject md_obj = env->CallStaticObjectMethod(md_clz, md_get_mtd, env->NewStringUTF("SHA1"));
    jmethodID dig_mtd = env->GetMethodID(md_clz, "digest", "([B)[B");
    auto pb_key_ary = (jbyteArray) env->CallObjectMethod(md_obj, dig_mtd, sign_byt_ary);

    env->DeleteLocalRef(md_clz);
    env->DeleteLocalRef(md_obj);

    // 申请一片内存
    jbyte *key_ary = env->GetByteArrayElements(pb_key_ary, nullptr);
    // 获取数组的长度
    int len = env->GetArrayLength(pb_key_ary);
    char temp[len * 2];
    for (int i = 0; i < len; i++) {
        temp[i * 2] = CHARS[(key_ary[i] & 0xF0u) >> 4];
        temp[i * 2  + 1] = CHARS[(key_ary[i] & 0x0Fu)];
    }
    return strcmp(temp, SHA1) == 0;
}

jstring init_check(JNIEnv* env, jclass thiz, jobject context) {
    // 1.1、准备获取PackageManager的方法类、返回值类
    jclass cxt_clz = env->FindClass("android/content/Context");
    // 1.2、准备获取PackageManager的方法id
    jmethodID get_pm_mtd = env->GetMethodID(cxt_clz, "getPackageManager",
                                            "()Landroid/content/pm/PackageManager;");
    // 1.3、调用获得PackageManager对象
    jobject pm_obj = env->CallObjectMethod(context, get_pm_mtd);
    // 1.4、获取包名
    jmethodID get_pn_mtd = env->GetMethodID(cxt_clz, "getPackageName", "()Ljava/lang/String;");
    auto pkg_name = (jstring) env->CallObjectMethod(context, get_pn_mtd);

    env->DeleteLocalRef(cxt_clz);

    // 2.1 准备获取pkinfo的方法类和返回值类
    jclass pm_clz = env->FindClass("android/content/pm/PackageManager");
    jclass pi_clz = env->FindClass("android/content/pm/PackageInfo");
    // 2.2 准备获取pkinfo的方法id
    jmethodID get_pi_mtd = env->GetMethodID(pm_clz, "getPackageInfo",
            "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");

    // 2.3 获取签名返回类型
    jobjectArray sig_ary;
    bool is_lower_p = is_lower_P();
    if (is_lower_p) {
        jfieldID flag_id = env->GetStaticFieldID(pm_clz, "GET_SIGNATURES", "I");
        jint flag = env->GetStaticIntField(pm_clz, flag_id);
        jobject pi_obj = env->CallObjectMethod(pm_obj, get_pi_mtd, pkg_name, flag);
        if (env->ExceptionOccurred()) {
            env->ExceptionClear();
            return env->NewStringUTF("P以下的设备有点异常了");
        }

        jfieldID sign_ary_id = env->GetFieldID(pi_clz, "signatures", "[Landroid/content/pm/Signature;");
        sig_ary = (jobjectArray) env->GetObjectField(pi_obj, sign_ary_id);

        env->DeleteLocalRef(pi_obj);
    } else {
        jfieldID flag_id = env->GetStaticFieldID(pm_clz, "GET_SIGNING_CERTIFICATES", "I");
        jint flag = env->GetStaticIntField(pm_clz, flag_id);
        jobject pi_obj = env->CallObjectMethod(pm_obj, get_pi_mtd, pkg_name, flag);
        if (env->ExceptionOccurred()) {
            env->ExceptionClear();
            return env->NewStringUTF(">=P的设备有点异常了");
        }
        jclass sgi_clz = env->FindClass("android/content/pm/SigningInfo");
        jfieldID sgi_id = env->GetFieldID(pi_clz, "signingInfo", "Landroid/content/pm/SigningInfo;");
        jobject sgi_obj = env->GetObjectField(pi_obj, sgi_id);

        jmethodID get_sig_mtd = env->GetMethodID(sgi_clz, "getApkContentsSigners", "()[Landroid/content/pm/Signature;");
        sig_ary = (jobjectArray) env->CallObjectMethod(sgi_obj, get_sig_mtd);

        env->DeleteLocalRef(pi_obj);
        env->DeleteLocalRef(sgi_clz);
        env->DeleteLocalRef(sgi_obj);
    }

    env->DeleteLocalRef(pm_clz);
    env->DeleteLocalRef(pi_clz);

    bool checkPass = false;
    if (sig_ary) {
            jobject sig_obj = env->GetObjectArrayElement(sig_ary, 0);
            jclass sig_clz = env->FindClass("android/content/pm/Signature");
            jmethodID to_byt_mtd = env->GetMethodID(sig_clz, "toByteArray", "()[B");
            auto sign_byt_ary = (jbyteArray) env->CallObjectMethod(sig_obj, to_byt_mtd);
            checkPass = getSHA1SignAndCheck(env, sign_byt_ary);
    }

    if (checkPass) {
        return env->NewStringUTF("检查通过");
    } else {
        return env->NewStringUTF("杀死进程！");
    }
}


#define JNIREG_CLASS "com/ktdemo/kevinlee/AppInit" // 定义类名

static JNINativeMethod gMethods[] = {
    {"doInit", "(Landroid/content/Context;)Ljava/lang/String;", (jstring *)init_check}
};

jint jniRegisterNatives(JNIEnv *env) {
    jclass clz = env->FindClass(JNIREG_CLASS);
    if (clz == nullptr) {
        return JNI_FALSE;
    }
    if(env->RegisterNatives(clz, gMethods, 1) < 0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}

jint JNI_OnLoad(JavaVM *vm, void *unused) {
    JNIEnv *env = nullptr;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    assert(env != nullptr);

// 注册native方法
    if (!jniRegisterNatives(env)) {
        return JNI_ERR;
    }

    return JNI_VERSION_1_6;
}
```

【问题】

3、这JNI里获取签名也太麻烦了吧，我能不能自己在工具类是把整个转换过程全部写好，然后jni调工具类获取到字符串直接与c中定义好的字符串对比？

原则上来说如果对字符串有一些乱七八糟的处理，这些处理不放java里面即可，其他代码倒是无所谓，但是最好不要。

## 破局

### 整锅端走

如果单单只是把上面检查签名的功能打成libcheck.so，这个是非常不可取的，因为这个功能比较单一，在java层对接的native方法又不能被混淆，只要“有心人”知道了java方法，就能参照java方法当场写一个一模一样的so库，并且掏空内部实现，然后重新打包，让你原地失业。所以一般这个校验功能是和其他必要的so库放一起使用的，一锅端会导致关键的方法完全无法执行，例如公版被一锅端就会导致请求异常。

什么？你的app只有这一个签名校验的so？

既然一直都这么“豪放”，说明app本身也没啥好值得防止重新打包的。

### 添点麻烦

```groovy
defaultConfig {
    applicationId "com.kevin.demo"
    minSdkVersion 23
    targetSdkVersion 30
    versionCode 1
    versionName "1.0"

    externalNativeBuild {
        cmake {
            cFlags "-fvisibility=hidden"
            cppFlags "-std=c++11 -fvisibility=hidden"
        }
    }

}
```

![压缩未混淆](./压缩未混淆.png)

![压缩且混淆](./压缩且混淆.png)

## 工具分享

