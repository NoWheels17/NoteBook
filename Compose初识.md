## Thinking In Compose

学习笔记的第一节取这个名字，首先因为官方文档的第一节也叫这个。其次Compose并不是单纯的将原来的UI换了一种写法，可以理解为一种新的UI渲染方式，通过@Compose这个特殊的编译时注解定义每个微件，然后组合它们完成整个页面渲染。在学习代码之前，这里有一些概念要先消化掉。

### 声明式UI

可以理解为之前学习Flutter的UI编码形式，说的更直白一点，就是通过函数表达式将UI和与其关联的数据一起声明出来。个人理解好处是UI和数据在微件中关联在一起方便理解数据和UI之间的关系，不必像传统的编码那样通过id找对应关系；坏处就是容易嵌套，嵌套的代码非常难阅读，虽然有@Compose注解便于拆分微件。

### 动态内容

```kotlin
@Composable
fun Greeting(names: List<String>) {
    for (name in names) {
        Text("Hello $name")
    }
}
```

官方给出的例子是这样的，实际上就是UI会跟着数据更新，不需要我们再set、notify之类的操作

### 重组

关于重组官方花了大篇幅去介绍，个人总结之后大概是如下几个特征：

- 【最小化】当数据变化时，只是与之相关的UI组件会重组，例如下面的例子，只有Text会重组

  ```kotlin
  @Composable
  fun ClickCounter(clicks: Int, onClick: () -> Unit) {
      Button(onClick = onClick) {
          Text("I've been clicked $clicks times")
      }
  }
  ```

- 【无序性】例如下面part1～3，可组合函数声明的顺序并不一定是其绘制的顺序

  ```kotlin
  @Composable
  fun ButtonRow() {
      MyFancyNavigation {
          StartScreen() // part1
          MiddleScreen() // part2
          EndScreen() // part3
      }
  }
  ```

  所以写代码的时候要非常注意，千万不能让3者之间有时序上的强依赖。举个🌰：

  ```kotlin
  var a = 0
  fun StartScreen() {
    a == 1
    // 渲染顶部内容
  }
  
  fun MiddleScreen() {
    if (a == 1) {
      // 渲染中间内容
    }
  }
  ```

  ❌上面这种写法，很可能翻车，千万别尝试。

- 【高效率】可组合函数可能会使用多个线程并行执行，而且只会重组需要的部分，数据频繁变化时，Compose可能会抛弃做到一半的重组动作，用最新的数据重新开始

- 【不稳定】可组合函数再是原来UI框架里每个帧时执行一次了，可能会执行很多次，所以千万不要在里面写耗时操作

  （其实就算不是Compose，我也就见过1次有人在UI绘制的生命周期里读SP）

对重组的大篇幅说明，其实就是官方希望我们编码时要注意这些误区。

### CompositionLocal

这个是最诡异的...官方文档竟然把这个放在前面，类似的东西Provider在Flutter其实是放在后面的，说白了声明式UI的最大问题就是数据和UI的关系，传统的AndroidUI可以“定点爆破”，指定到某个UI设置值。Compose里面要么依赖全局的变量，要么就得跟随UI树向下传递，声明式的UI永远逃不掉的噩梦就是UI限制了参数传递。

```kotlin
import androidx.compose.runtime.compositionLocalOf

var LocalString = compositionLocalOf { "Using Compose" }

@Composable
fun getContent() {
    Column {
        CompositionLocalProvider(LocalString provides "Hello World") {
            Text( // 文案1
                text = LocalString.current,
                color = Color.Green
            )
          	Text( // 文案2
                text = "123",
                color = Color.Cyan
            )
        }
        Text( // 文案3
            text = LocalString.current,
            color = Color.Red
        )
    }
}

```

按照顺序文案1、2、3的现实内容为：

<font color=#00FF00>Hello World</font>

<font color=#00FFFF>123</font>

<font color=#FF0000>Using Compose</font>

上面就是基础的用法，跟Flutter的Provider，简直是一模一样

#### compositionLocalOf 与 staticCompositionLocalOf 区别

我们在声明一个compositionLocal对象时，其实还有一种方式

```kotlin
var LocalString = staticCompositionLocalOf { "Using Compose" }
```

有啥区别呢？

区别就是，如果LocalString这个变量产生变化：

compositionLocalOf只会更新文案1

staticCompositionLocalOf会重组它层级下的所有试图，即文案1、文案2

## 布局

这一章主要是从Android的传统视角看Compose，所以学习的思路都是Android布局的思路

### 基础布局

#### 默认

```kotlin
@Composable
fun RawStudy() {
    Text(text = "String one")
    Text(text = "String two")
}
```

如果不添加布局方式，那么这种情况默认的就是**FrameLayout**

#### 线性布局

```kotlin
// 纵向
Column(horizontalAlignment = Alignment.Horizontally) {
    Text(text = "String one")
    Text(text = "String two")
}
// 横向
Row(verticalAlignment = Alignment.CenterVertically) {
    Text(text = "String one")
    Text(text = "String two")
}
```

##### 对齐

```kotlin
// 常用的就是2个方向上的对齐了，Alignment有多个值
verticalAlignment = Alignment.CenterVertically
horizontalAlignment = Alignment.Horizontally
```

##### 宽高约束

默认的原则都是wrap_content，如果想实现match_parent，需要使用修饰器，例如

```kotlin
// 宽度match_parent 高度 120dp，以此类推
Row (modifier = .fillMaxWidth().height(120.dp))
```

##### 权重

```kotlin
Row(verticalAlignment = Alignment.CenterVertically,
   modifier = Modifier.fillMaxWidth().height(120.dp)) {
    Text(text = "设备名", fontSize = 20.sp, modifier = Modifier.weight(1f)) // 高度为60dp
    Text(text = "设备id", fontSize = 14.sp, modifier = Modifier.weight(1f)) // 高度为60dp
}
```

#### 盒子

这个其实就是Framelayout，但是在Box中如果子组件需要match_parent则需要使用Modifier.matchParentSize()修饰

```kotlin
Box {
    Text(text = "设备名", fontSize = 20.sp)
    Text(text = "设备id", fontSize = 16.sp, Modifier.matchParentSize())
}
```

##### 特殊情况

当Box下面有Spacer时

- 如果Spacer使用matchParentSize，那么Box是wrap_content模式，宽高与Text一致

```kotlin
Row (modifier = Modifier.size(240.dp, 120.dp)) {
    Box(modifier = Modifier.background(Color.Blue)) {
      	// 注意这里
        Spacer(modifier = Modifier.matchParentSize().background(Color.Cyan))
        Text(text = "设备名", fontSize = 20.sp)
    }
}
```

- 如果Spacer使用matchParentSize，那么Box是wrap_content模式，宽高与外层Row一致 (240.dp, 120.dp)

```kotlin
Row (modifier = Modifier.size(240.dp, 120.dp)) {
    Box(modifier = Modifier.background(Color.Blue)) {
      	// 注意这里
        Spacer(modifier = Modifier.fillMaxSize().background(Color.Cyan))
        Text(text = "设备名", fontSize = 20.sp)
    }
}
```

需要注意的是Modifier.matchParentSize()只能在Box标签下使用

#### Scaffold

```kotlin
fun Scaffold(
    modifier: Modifier = Modifier,
    scaffoldState: ScaffoldState = rememberScaffoldState(),
    topBar: @Composable () -> Unit = {},
    bottomBar: @Composable () -> Unit = {},
    snackbarHost: @Composable (SnackbarHostState) -> Unit = { SnackbarHost(it) },
    floatingActionButton: @Composable () -> Unit = {},
    floatingActionButtonPosition: FabPosition = FabPosition.End,
    isFloatingActionButtonDocked: Boolean = false,
    drawerContent: @Composable (ColumnScope.() -> Unit)? = null,
    drawerGesturesEnabled: Boolean = true,
    drawerShape: Shape = MaterialTheme.shapes.large,
    drawerElevation: Dp = DrawerDefaults.Elevation,
    drawerBackgroundColor: Color = MaterialTheme.colors.surface,
    drawerContentColor: Color = contentColorFor(drawerBackgroundColor),
    drawerScrimColor: Color = DrawerDefaults.scrimColor,
    backgroundColor: Color = MaterialTheme.colors.background,
    contentColor: Color = contentColorFor(backgroundColor),
    content: @Composable (PaddingValues) -> Unit
)
```

严格来说，Scaffold更像是一个设计结构，跟Flutter中的Scaffold几乎一模一样，有头部导航、底部导航、悬浮按钮，是标准的Material风格

## 自定义布局

这一段应该算进阶知识了，需要理解和熟练的API增加了一些，Compose中自定义布局的方式用原生的思想来说无非就是两种：

### 使用Modifier（类似自定义LayoutParams）

```kotlin
fun Modifier.fakeMeasure(tag: String) = layout { measurable, constraints ->
    // 关键点1: 完成对自身size的测量
    val placeable = measurable.measure(constraints)
    println("Kevin-- fakeM $tag - width: ${placeable.width}, height: ${placeable.height}")
    // 关键点2: 使用测量后的宽高设置视图在排布时实际占用的宽高空间大小
    layout(placeable.width, placeable.height) {
        // 关键点3: 相对预期位置进行视图排布
        placeable.placeRelative(0, 0)
    }
}

@Preview(showBackground = true)
@Composable
fun Test2() {
    Column (modifier = Modifier
        .height(300.dp)
        .fillMaxWidth()) {
        Text(text = "测试1", fontSize = 20.sp, modifier = Modifier.fakeMeasure("t1"))
        Text(text = "测试2", fontSize = 20.sp, modifier = Modifier.fakeMeasure("t2"))
        Image(
            painter = painterResource(id = R.drawable.ic_launcher_background),
            contentDescription = null,
            modifier = Modifier
                .size(104.dp)
        )
    }
}
```

这里有些问题需要说明一下：

fakeMeasure是Modifier的拓展函数，在Column布局中不影响子视图最终的位置。如果仔细拆开分析这个自定义Modifier，目前只能感叹，还好Java不支持高阶函数，不然可读性上还是有点折磨的

【关键点1】

解释下三个对象

**measurable**：子元素的测量句柄，通过提供的api完成测量与布局过程

**constraints**: 子元素的测量约束，包括宽度与高度的最大值与最小值

**placeable**：布局对象，承接测量后的结果，并负责对视图进行布局

【关键点2】

这里的layout是Modifier的一个函数，指定当前视图的布局范围，这里需要主要的是，如果范围小于实际内容，是不会被裁剪的，这一点与Android原始布局不一样。

【关键点3】

**placeRelative**：这个方法是相对布局范围x，y方向偏移多少像素开始布局

### 使用Composable方法（类似自定义ViewGroup）

```kotlin
@Composable
fun ColumnLike(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
  	// 关键点1
    Layout(modifier = modifier, content = content) { measurables, constraints ->
        // 关键点2                                            
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }
                                                    
        var yPosition = 0
        // 关键点3                                            
        layout(constraints.maxWidth, constraints.maxHeight) {
            placeables.forEach { placeable ->
                placeable.placeRelative(x = 0, y = yPosition)
                // 关键点4
                yPosition += placeable.height
            }
        }
                                                    
    }
  
}
```

看到这里的时候，简直内牛满面，想起了当初手写自定义ViewGroup的幸酸历史，这个步骤跟Android原生如出一辙。

【关键点1】

这里需要注意的是Layout和layout这俩方法不一样（注意大小写），Layout是Layout.kt文件中的内联方法，layout是Modifier的拓展方法

【关键2、3】

这2个点无法就是自定义Modifier的遍历版本

【关键点4】

从这里能看出，placeable.placeRelative不再是每个子视图相对于自己的位置了，而是他们相对父布局ColumnLike原点的位置





