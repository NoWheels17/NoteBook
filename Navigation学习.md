## Navigation

也是官方Jetpack组件，学习Navigation主要是看中这个组件对返回栈的管理，在使用fragment的时候不需要手动设置切换逻辑，在使用Activity时不需要考虑过多的模式设计问题。

### 快速开始

不得不说，AndroidStudio对Jetpack的支持越来越积极，总是会有新的插件和快捷方法快速支持组件功能。

#### 创建导航资源

如需向项目添加导航图，请执行以下操作：

1. 右键点击 `res` 目录，然后依次选择 **New > Android Resource File**。此时系统会显示 **New Resource File** 对话框。
2. 在 **File name** 字段中输入名称，比如"**navi_study**"。
3. 从 **Resource type** 下拉列表中选择 **Navigation**，然后点击 **OK**。

```tex
res
|-navigation
	|-navi_study.xml
```

创建完之后大概就是这样的，navigation文件夹会自动创建，要说的是这个文件并不能直接使用，还需要指定宿主，让宿主来承载这个导航资源。

#### 指定宿主

如果是新的demo，在layout_main.xml中加一下如下控件就好了

```xml
    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"

        app:defaultNavHost="true"
        app:navGraph="@navigation/navi_study" />
```

- `android:name` 属性包含 `NavHost` 实现的类名称。

- `app:navGraph` 属性将 `NavHostFragment` 与导航图相关联。导航图会在此 `NavHostFragment` 中指定用户可以导航到的所有目的地。注意，navi_study就是我们创建的导航资源。

- `app:defaultNavHost="true"` 属性确保您的 `NavHostFragment` 会拦截系统返回按钮。请注意，只能有一个默认 `NavHost`。如果同一布局（例如，双窗格布局）中有多个宿主，请务必仅指定一个默认 `NavHost`。

  这里需要注意，如果在这个值为true

#### 导航链接

这里直接先代码再解释

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/demo_nav"
    app:startDestination="@id/page_1">


    <fragment android:id="@+id/page_1"
        android:name="com.kevin.demo.Page1Fragment"
        android:label="Page1"
        tools:layout="@layout/fragment_blank">
        <action
            android:id="@+id/action_one_to_two"
            app:destination="@id/page_2" />
    </fragment>

    <fragment android:id="@+id/page_2"
        android:name="com.kevin.demo.Page2Fragment"
        android:label="Page2"
        tools:layout="@layout/fragment_blank2">
        <action
            android:id="@+id/action_two_to_three"
            app:destination="@id/page_3" />
    </fragment>

    <fragment android:id="@+id/page_3"
        android:name="com.kevin.demo.Page3Fragment"
        android:label="Page3"
        tools:layout="@layout/fragment_blank3"/>

</navigation>
```

反正我看完之后是秒懂，主要有3个关键点：

- 根标签的app:startDestination="@id/page_1"，指定了导航一开始的默认fragment

- android:name="com.kevin.demo.Page2Fragment" 指定导航页面的fragment，可以跨组件，支持组件化

- <action标签，id是这个跳转动作的唯一id，destination是目标页面，同时action在官网说明上其实还支持动画，例如：

  ```xml
  <action android:id="@+id/action_a_to_b"
                  app:destination="@id/b"
                  app:enterAnim="@anim/nav_default_enter_anim"
                  app:exitAnim="@anim/nav_default_exit_anim"
                  app:popEnterAnim="@anim/nav_default_pop_enter_anim"
                  app:popExitAnim="@anim/nav_default_pop_exit_anim"/>
  ```

#### 执行导航

要执行导航的动作的跳转，需要2步：

```java
// 1、获取NavigationController
NavHostFragment.findNavController(Fragment);
Navigation.findNavController(Activity, @IdRes int viewId);
Navigation.findNavController(View); 
// 2、通过controller执行跳转动作
controller.navigate(R.id.action_one_to_two)
```

那上面3个获取NavigationController有什么区别呢？

首先需要知道的是NavigationController对象在NavHostFragment的onViewCreated方法中使用View.setTag()设置给了fragment的根视图。

- NavHostFragment.findNavController(Fragment)

  这里的fragment要求是NavHostFragment，否则会向上寻找父层级的NavHostFragment，然后从fragment的根视图中获取controller

- Navigation.findNavController(Activity, @IdRes int viewId) 和 Navigation.findNavController(View)

  这2个函数的操作基本是一致的，只是前者多了一步通过id先获取到View的步骤，后面的步骤则是ViewParent逐层向上查找视图，然后通过getTag获取到NavigationController为止

#### 导航回退

```java
controller.navigateUp();
controller.popBackStack();
```

### 进阶操作

#### 跳转传参

##### 发送方

创建 `Bundle` 对象并使用 navigate(id, bundle)将它传递给目的地，如下所示：

```java
Bundle bundle = new Bundle();
bundle.putString("param", param);
Navigation.findNavController(view).navigate(R.id.confirmationAction, bundle);
```

##### 接收方

```java
TextView tv = view.findViewById(R.id.textViewAmount);
tv.setText(getArguments().getString("param"));
```

虽然Navigation在路由跳转的时候可以传递参数，但是官方却不是很推荐这么做，特别是当数据量特别大的时候。在fragment之间传输数据最好是使用ViewModel。

#### 特殊场景管理

##### 循环

导航创建的时候，无法避免循环的情况，例如有3个页面A、B、C构成导航路径为A->B->C->A，如果我们不做任务操作，那么栈中的页面会越来越多，那么怎么实现从C->A的时候，B、C都退出栈呢？其实很简单

```xml
<action
    android:id="@+id/action_c_to_a"
    app:destination="@id/aFragment"
    app:popUpTo="@id/aFragment"
    app:popUpToInclusive="true" />
```
popUpToInclusive为false和true的时候，从C跳转到A的情况如下：

```txt
|-C-|             |   |                  |-C-|            |   |
|-B-|    false    |-A-|                  |-B-|    true    |   |
|-A-|     >>>     |-A-|                  |-A-|     >>     |-A-|
|---|             |---|                  |---|            |---|
```

##### 不保留跳转

先解释一下这个场景

- A页面是主页面
- B页面是登录页
- C页面是首次登录欢迎页

```xml
<action
    android:id="@+id/action_b_to_c"
    app:destination="@id/cFragment"
    app:popUpTo="@id/aFragment"
    app:popUpToInclusive="false" />
```

popUpToInclusive为false和true的时候，从C跳转到A的情况如下：

```txt
|   |             |   |                  |   |            |   |
|-B-|    false    |-C-|                  |-B-|    true    |   |
|-A-|     >>>     |-A-|                  |-A-|     >>     |-A-|
|---|             |---|                  |---|            |---|
```

这里看了别人的一个总结，总结的比较抽象，这里自己也总结一下，关于这2个标签：

- popUpTo

  指定跳转动作发生之后，栈中destination指定页面将在哪个页面上

- popUpToInclusive

  指定跳转动作发生之后，栈中是否保留destination指定的页面

#### 深层链接

##### 显示深层链接

假设有点击通知栏然后跳转到某个Activity的某个Fragment上这种要求，那么必定需要一个PendingIntent作为点击通知栏之后的跳转意图，显示的意图代码大致如下：

```java
PendingIntent pendingIntent = new NavDeepLinkBuilder(context)
        .setGraph(R.navigation.nav_graph) // 导航资源文件
        .setDestination(R.id.des_fragment) // fragment的id，一定要在导航资源文件中声明了才生效
        .setArguments(args) // 跳转带的参数
        .setComponentName(DestinationActivity.class) // 承载fragment的Activity
        .createPendingIntent();
```

##### 隐式深层链接

隐式的调用也很简单分为2个步骤

- 在navigation资源文件的fragment中声明<deepLink标签

  ```xml
  <fragment
      android:id="@+id/des_fragment"
      android:name="com.kevin.demo.DeepLinkDemoFragment"
      tools:layout="@layout/fragment_deep_link_demo">
  
      <!-- 为destination添加<deepLink/>标签 -->
      <deepLink app:uri="www.kevinlee.com" />
  
  </fragment>
  ```

  这里需要说明的是，如果uri没有指明前缀，那么默认匹配的是http或者https，如果想指明前缀，可以使用

  ```xml
  <deepLink app:uri="com.kevin://www.kevinlee.com" />
  ```

- 在AndroidManifest.xml中，对应目标所属的Activity下声明<nav-graph标签

  ```xml
  <activity android:name="nav.DestinationActivity">
          <intent-filter>
              <action android:name="android.intent.action.VIEW" />
              <category android:name="android.intent.category.DEFAULT" />
              <category android:name="android.intent.category.BROWSABLE" />
          </intent-filter>
  
          <!-- 为Activity设置<nav-graph/>标签 -->
          <nav-graph android:value="@navigation/nav_graph" />
      </activity>
  ```

