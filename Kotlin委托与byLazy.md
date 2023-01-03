## 什么是委托？

对象A接收到某些请求后，将请求交由B去处理。在调用关系上，最常见的是外层调用A的方法，A再调用成员B的方法实际执行请求。这种模式就是委托。

``` java
interface Worker {
  void handleTask()
}

class Leader implements Worker {
  private TeamMember teammate;
  public Leader(Worker worker) {
    teammate = worker; 
  }
  
  @Override
  public void handleTask() {
    teammate.handleTask();
  }
}

class TeamMember implements Worker {
  @Override
  public void handleTask() {
    System.out.println("PPTing");
  }
}

// 调用
public static void bossMission() {
  TeamMember mermber = new TeamMember();
	Leader leader = new Leader(mermber);
	leader.handleTask();
}

```

## 委托分类

Kotlin的委托最常见分为两种，一种是类委托，一种是属性委托。顾名思义这两种分别是针对类和属性。

## 类委托

### 使用

```kotlin
interface Worker {
    fun handleTask()
}

class Leader(private val teammate: Worker): Worker by teammate {  //将Worker的实现委托给teammate实现

}

class TeamMember: Worker {
  override void handleTask() {
    println("PPTing");
  }
}

class TestClass() {
    fun bossMission() {
        val member = TeamMember()
      	//member作为Leader的Worker委托对象
        val leader = Leader(member)
      	//调用leader.handleTask()实际委托给member即TeamMember的handleTask方法，所以会打印出PPTing
        leader.handleTask()
    }
}
```

对比一下第一节委托模式解释的代码，可以发现单从代码层面看，Kotlin的委托其实就是在`Leader`类的实现上简化了一些代码。那么可以猜想一下，类委托的原理是不是就是帮忙实现了简化掉的代码呢？

### 字节码

```java
public final class Leader implements Worker {

  // access flags 0x12
  private final LWorker; teammate

  // access flags 0x1
  public <init>(LWorker;)V
    
  // 构造部分省略....
    
  public handleTask()V
   L0
    ALOAD 0
    GETFIELD Leader.teammate : LWorker;
    INVOKEINTERFACE Worker.handleTask ()V (itf)
    RETURN
   L1
    LOCALVARIABLE this LLeader; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1
    
}
```

可以看到，在`Leader`类中有属性`Worker teammate`，在`handleTask`方法中，`GETFIELD`获取到了`teammate`压入栈顶，然后执行了栈顶对象的`handleTask`方法，这里其实就跟Java的委托是一样的实现方式。

## by lazy

### 使用

```kotlin
val a: Int by lazy {
  1
}

val v: TextView by lazy {
  findViewById(R.id.text)
}
```

`by lazy`实际上是属于属性委托，看懂了它的实现原理之后，也就能知道自定义的属性委托怎么实现。

### 原理

先来看看lazy到底是个啥

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    // ...
}
```

1、从上面的源码我们可以发现lazy是一个区分平台(actual)实现的顶层高阶函数，返回一个Lazy类型的对象，通过`SynchronizedLazyImpl`构造方法返回的对象。

2、`initializer`函数返回值就是`委托者`最后被赋的值，从逻辑上可以看到，这个返回值没有直接给到外面，而是给了`value`变量，这是怎么回事？这个链路我们待会再分析，这里先看`get()`函数其实就是经典的DCL，这里就不分析了，现在只要分析`value`怎么赋值给`委托者`就好了

那我们再看Lazy

```java
public interface Lazy<out T> {

    public val value: T

    public fun isInitialized(): Boolean
}
```

3、这里看完之后，明白了`value`是一个属性，我们再梳理一下链路：

```
initializer()返回值 -----> value --xx--> 委托者
```

发现从`value`赋值给`委托者`链路的代码找不到，我们跟了顶层函数`lazy`，跟了`SynchronizedLazyImpl`类，而且`SynchronizedLazyImpl`就实现了2个接口`Lazy`和`Serializable`，那没了啊，委托好神奇，怎么关联的？

我一开始也是这么个想法，但我们细看一下`Lazy.kt`源码文件，就会发现：

```kotlin
public inline operator fun <T> Lazy<T>.getValue(thisRef: Any?, property: KProperty<*>): T = value
```

没错，就是这里，操作符重载，重载写了`getValue`方法，我们常见的是+、-、×、÷的重载，但是`getValue`很少有。简单点来说，这个函数的意思是，当能get`thisRef`这个对象的的值之前，就会调用这个方法，方法的返回值会赋值给`thisRef`。

## 自定义属性委托

了解了lazy的机制原理以后，是不是可以自己实现一下，其实是的：

1、新建类实现`ReadOnlyProperty`接口，这样不用手写需要重载的方法

2、实现顶层函数传递函数参数给`被委托者`，并返回`被委托者`

这样就能简单的仿制一个lazy了，代码如下：

```kotlin
import kotlin.properties.ReadOnlyProperty
import kotlin.reflect.KProperty

public fun <T> myLazy(valueGet: ()->T): MyLazyImpl<T> = MyLazyImpl(valueGet)

public class MyLazyImpl<T> (private val valueGet: ()->T) : ReadOnlyProperty<Any?, T> {
    
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return valueGet()
    }

}


// 新起一个类文件，方便看字节码
class MyKotlin {

    val a: Int by myLazy {
        1
    }

}
```

我们来看字节码

### 字节码

```kotlin
private final LMyLazyImpl; a$delegate

  public final getA()I
   L0
    ALOAD 0
    GETFIELD MyKotlin.a$delegate : LMyLazyImpl;
    ALOAD 0
    GETSTATIC MyKotlin.$$delegatedProperties : [Lkotlin/reflect/KProperty;
    ICONST_0
    AALOAD
    INVOKEVIRTUAL MyLazyImpl.getValue (Ljava/lang/Object;Lkotlin/reflect/KProperty;)Ljava/lang/Object;
    CHECKCAST java/lang/Number
    INVOKEVIRTUAL java/lang/Number.intValue ()I
    IRETURN
   L1
    LOCALVARIABLE this LMyKotlin; L0 L1 0
    MAXSTACK = 4
    MAXLOCALS = 1
```

这样瞬间就悟了，原来还是生成了一个MyLazyImpl的代理属性，然后调用`getValue`方法获取值。by lazy这个操作精髓主要在委托和懒加载两方面，少一个都不精髓了，所以kotlin好用的语法糖都是有具体的实现手段的，而且并不是没有代价的。