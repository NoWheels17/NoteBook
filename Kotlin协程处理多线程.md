## Kotlin协程处理多线程

### 关于“协程”的定义

首先明确一个包含关系，一个进程可以有多个线程，一个线程可以有多个协程，所以协程其实比线程更小的单元，协程相对于线程而言，消耗的资源更少，且不需要在用户态和内核态之间互相切换。

但是协程并不是万能的，协程的工作是当线程有时间片的时候才有资源执行，相对的如果协程做了阻塞操作，其所在线程也是会被阻塞的，总之协程并不能比线程拥有更多的资源，撑死是充分利用线程资源。所以在使用协程的时候，耗时操作千万不要在主线程中创建协程执行？

### 为什么要学？

学习协程的初衷是想使用协程的简单写法替换RxJava复杂而恶心的回调。啥意思？直接说场景和上代码，假设首页需要请求tuya和fusion的batch接口，然后等2个数据都回来之后，更新UI。用java，这里有2个选择：

```java
// 1、RxJava的merge操作符, 然后折腾一堆回调，适合简单的数据合并
observable1 = Observable.create...
observable2 = Observable.create...
Observable.merge(observable1, observable2);

// 2、使用Handler，让状态在主线程中同步，适用与回调之后，需要处理麻烦而细节的业务
mHandler.post(...);
```

用协程实现的话，预期是这样的：

```kotlin
CoroutineScope(Dispatchers.IO).launch {
    val tuyaHome: Deferred = async { model.getTuyaHomeDetail(homeId) } // 其他线程异步获取
    val fusionHome: Deferred = async { model.getFusionHomeDetail(homeId) } // 其他线程异步获取
    val mergeHome = doMerge(tuyaHome.await(), fusionHome.await())
    doCallback(mergeHome)
}
```

就问这种跨线程的代码不依赖回调全部写在一起是不是看着很舒服，这么好的异步工具相当值得学习。

 <font color=#FF7D00>我们需要切换线程的时候，就可以用到协程</font>

### 学就完事儿了

#### launch() {}

先说明一下语法，这个launch并不能直接裸写，否则会报错。正确的示例如下：

```kotlin
fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        println("Kevin-- 1:" + Thread.currentThread().id)
        launch(Dispatchers.IO) { // 有了CoroutineScope作用域就可以直接写launch了
            delay(100)
            println("Kevin-- 2:" + Thread.currentThread().id)
            delay(200)
        }
        launch(Dispatchers.IO) {
            delay(100)
            println("Kevin-- 3:" + Thread.currentThread().id)
            delay(300)
        }
        println("Kevin-- 4:" + Thread.currentThread().id)
    }
}
```

launch的意思其实就是执行在某种线程中创建协程，从使用角度直接理解就是launch里面的代码需要切换另外一个线程去操作，含义就是这么简单，下面实操一下看下结果：

```tex
com.ktdemo.kevinlee I/System.out: Kevin-- 1:2
com.ktdemo.kevinlee I/System.out: Kevin-- 4:2
com.ktdemo.kevinlee I/System.out: Kevin-- 3:793
com.ktdemo.kevinlee I/System.out: Kevin-- 2:792
```

那如果需要跨2次主线程，改怎么写呢？

```kotlin
fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        launch(Dispatchers.IO) {
            ...
            launch(Dispatchers.Main) {
            	....
        	}
        }
    }
}
```

啊这......又开始嵌套了，体现不出来简洁的优势了。其实这种情况是有解的

#### withContext(){}

```kotlin
fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        val homeBean = withContext(Dispatchers.IO) {
            getHomeDetail()
        }
        doCallback(homeBean)
    }
}
```

或者这么写

```kotlin
fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        val homeBean = suspendGetHomeDetail()
        doCallback(homeBean)
    }
}

suspend fun suspendGetHomeDetail(): {
    return withContext(Dispatchers.IO) {
         getHomeDetail()
    }
}
```

#### async(){}

async关键字的使用在一开始已经给出来了，async和launch都是启动一个协程，不同的是async返回的是一个Deferred类型的对象，通过

```kotlin
Deferred.await()

// Deferred接口中的await方法
public suspend fun await(): T
```

能够获取到结果，这里就有能引出一个新的关键字

#### suspend

suspend意为“挂起”，用suspend修饰的函数必须间接或者直接地在协程中使用，当执行到suspend函数时，调用suspend函数的协程会被挂起，协程所在的线程会继续执行线程中剩下的代码；协程在指定线程中执行完代码后，会返回到调用suspend函数的线程中，这一点很像post。

```kotlin
Handler.post {
    ...
}


fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        val homeBean = suspendGetHomeDetail()
        doCallback(homeBean)
    }
}

suspend fun suspendGetHomeDetail(): {
    return withContext(Dispatchers.IO) {
         getHomeDetail()
    }
}
```

在上面的例子中挂起是的CoroutineScope.launch{}的部分，挂起的协程是其launch的在主线程中的协程。等suspend函数执行完后，协程会被resume，这个功能是协程才具有的。

这里在理解上有一个非常重要的点，suspend并不能挂起函数，只是作为一个标记，当执行到withContext时，withContext内部实现了挂起的功能，这个函数才能够被挂起，所以假设上面的代码是：

```kotlin
suspend fun suspendGetHomeDetail(): {
    return getHomeDetail()
}
```

这个函数实际上是不会被挂起的，最重要的一点是：

 <font color=#FF7D00>挂起函数是为了切换线程</font>

#### 什么时候需要使用suspend?

其实就是以前写啥代码需要开子线程，在使用协程的时候就需要使用suspend，规律很简单，这里也说明一个特殊场景：

```koltin
Handler.postDelay{()->{
    ...
}, 200L}
```

用协程写起来就是

```kotlin
fun learn() {
    CoroutineScope(Dispatchers.Main).launch {
        val homeBean = suspendGetHomeDetail()
        doCallback(homeBean)
    }
}

suspend fun suspendGetHomeDetail(): {
    delay(200)
   	...
}
```

### 最后的总结

关于Kotlin的协程，在学习完众多博客之后，发现了一个很严重的问题，Kotlin中的协程只是提供了线程池的API，支持切换线程，并且代码的风格是同步阻塞式的代码。挂起的操作实际上是切换了线程，回复则是把线程切了回来，Kotlin里的协程其实就是用漂亮的代码封装了线程池，实际上跟RxJava相比，在使用上并没有优越的地方。如果非得仔细严格地对比，其实协程甚至没有Java中直接使用线程池的效率高，只是写起来比Java线程池好看，没有了众多的回调而已。

最后的最后，在看完几篇原理性的说明后，本来以为Kotlin的协程是什么非常高深的东西，后来还是我的想法错了，毕竟kt和java最终编译都是class字节码，既然java都没有做到真正意义上的协程，kt就没有办法跨越这道门槛，除非kt直接对接native。

 <font color=#FF7D00>Kotlin的协程=漂亮的线程池</font>
