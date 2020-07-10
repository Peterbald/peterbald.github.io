---
layout: post
title: "Flutter 初探"
subtitle: ""
description: "iOS, 实践, Flutter, 跨平台"
author: "RandomJ"
header-img: "img/morocco.jpeg"
tags:
  - Flutter
  - iOS
  - 跨平台
---

本文所使用的 Flutter 环境如下：

```
> $flutter doctor --verbose

- Flutter (Channel stable/v1.17.3, v1.17.0-4.0.pre.12, on Mac OS X 10.15.5 19F101, locale zh-Hans-CN)
- Flutter version 1.17.0-4.0.pre.12 at /Users/random/flutter/flutter
- Framework revision 638c4d70f6 (3 weeks ago), 2020-06-12 13:35:25 +0800
- Engine revision 30b0ea8906
- Dart version 2.8.4
```

## 一、常用概念梳理：

## 1.1 Widget 是什么

Flutter 的组件分为两种，分为有状态组件 StatefulWidget 和无状态组件 StatelessWidget。

### 1.1.1 StatelessWidget

> Stateless widgets receive arguments from their parent widget, which they store in final member variables. When a widget is asked to build(), it uses these stored values to derive new arguments for the widgets it creates.

Stateless 组件从父组件接收参数，并将参数存储在成员变量中。当组件 build 方法被调用的时候，使用存储变量计算出它创建的子组件参数。

### 1.1.2 StatefulWidget

StatefulWidget，组件通过 createState 方法创建并持有 State。每次 setState 之后都会调用一次 build 方法构造新的 Widget 树。

````dart
abstract class StatefulWidget extends Widget {
  /// Initializes [key] for subclasses.
  const StatefulWidget({ Key key }) : super(key: key);

  /// Creates a [StatefulElement] to manage this widget's location in the tree.
  ///
  /// It is uncommon for subclasses to override this method.
  @override
  StatefulElement createElement() => StatefulElement(this);

  /// Creates the mutable state for this widget at a given location in the tree.
  ///
  /// Subclasses should override this method to return a newly created
  /// instance of their associated [State] subclass:
  ///
  /// ```dart
  /// @override
  /// _MyState createState() => _MyState();
  /// ```
  ///
  /// The framework can call this method multiple times over the lifetime of
  /// a [StatefulWidget]. For example, if the widget is inserted into the tree
  /// in multiple locations, the framework will create a separate [State] object
  /// for each location. Similarly, if the widget is removed from the tree and
  /// later inserted into the tree again, the framework will call [createState]
  /// again to create a fresh [State] object, simplifying the lifecycle of
  /// [State] objects.
  @protected
  State createState();
}
````

文档链接：[https://api.flutter.dev/flutter/widgets/StatefulWidget-class.html](https://api.flutter.dev/flutter/widgets/StatefulWidget-class.html)

### 1.1.3 Widget 文档

- [https://flutter.dev/docs/development/ui/widgets-intro](https://flutter.dev/docs/development/ui/widgets-intro)
- [https://api.flutter.dev/flutter/widgets/widgets-library.html](https://api.flutter.dev/flutter/widgets/widgets-library.html)

## 1.2 State 是什么

### 1.2.1 setState 如何驱动 UI

首先看下 setState 是如何驱动 UI 的，官方文档的阐述如下。

> When handling the onCartChanged callback, the \_ShoppingListState mutates its internal state by either adding or removing a product from \_shoppingCart. To signal to the framework that it changed its internal state, it wraps those calls in a setState() call. Calling setState marks this widget as dirty and schedules it to be rebuilt the next time your app needs to update the screen. If you forget to call setState when modifying the internal state of a widget, the framework won’t know your widget is dirty and might not call the widget’s build() function, which means the user interface might not update to reflect the changed state. By managing state in this way, you don’t need to write separate code for creating and updating child widgets. Instead, you simply implement the build function, which handles both situations.

在 setState 调用时通知 frameWork 改变内部状态。调用 setState 方法标记组件为 dirty 并且在下一次 app 刷新屏幕的时候重新构建组件。如果更改组件内部状态但是忘记调用了 setState 方法，framework 不知道组件是 dirty 的，不会调用组件的 build 方法，意味着 UI 不会更新来响应状态变化。通过这种方式管理状态，不需要写单独的代码来创建和更新子组件。只需要实现 build 函数就可以。

### 1.2.2 State 和 Widget 为什么分开实现

> You might wonder why StatefulWidget and State are separate objects. In Flutter, these two types of objects have different life cycles. Widgets are temporary objects, used to construct a presentation of the application in its current state. State objects, on the other hand, are persistent between calls to build(), allowing them to remember information.

大家可能会疑惑 StatefulWidget 和 State 为什么是分开实现的，因为这两种类型的对象有不同的生命周期。Widget 是临时对象，用来构造当前 state 下应用的展示。另一方面，State 对象在两次调用 build 方法之间是持久对象，可以用来记录相关信息。

### 1.2.3 State 生命周期

- widget.createState() 生命周期内仅调用一次。
- state.initState() 初始化 State，生命周期内仅调用一次。
- [state.build](http://state.build)() 构建组件树
- state.setState() 更新状态
- state.didUpdateWidget() 组件已经更新，参数是 oldWidget
- state.dispose() State 销毁

文档链接：[https://api.flutter.dev/flutter/widgets/State-class.html](https://api.flutter.dev/flutter/widgets/State-class.html)

## 1.3 Keys 作用

> Use keys to control which widgets the framework matches up with other widgets when a widget rebuilds. By default, the framework matches widgets in the current and previous build according to their runtimeType and the order in which they appear. With keys, the framework requires that the **two** widgets have the same key as well as the same runtimeType.

当一个组件重新构建的时候，使用 keys 来控制框架如何匹配组件。通常情况下，框架在匹配当前构建组件和先前构建的组件时是按照 widgets 的 runtimeType 和出现的顺序来匹配的。使用 keys 时，则需要组件有相同的 key 和 runtimeType。

文档链接: [https://api.flutter.dev/flutter/foundation/Key-class.html](https://api.flutter.dev/flutter/foundation/Key-class.html)

## 二、Flutter 启动链路详解

- Dart 里面入口函数 main 实现如下

```dart
void main() => runApp(MyApp());
```

- runApp 的实现

![runApp](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdkvpcej30se084jrl.jpg)

首先看下 .. 是什么，这是 Dart 里面的级联注解(Cascade notation)，在同一个对象上面进行一系列操作。参见文档 [https://dart.dev/guides/language/language-tour#cascade-notation-](https://dart.dev/guides/language/language-tour#cascade-notation-)

上面的 runApp 可以翻译成下面的代码：

```dart
void runApp(Widget app) {
	// 初始化 WidgetsBinding
	WidgetsBinding widgetsBinding = WidgetsFlutterBinding.ensureInitialized();
	// 附加根组件
	widgetsBinding.scheduleAttachRootWidget(app);
	// 绘制 frame
	widgetsBinding.scheduleWarmUpFrame();
}

// 附加根组件
@protected
void scheduleAttachRootWidget(Widget rootWidget) {
		// 为什么要放在 Timer 里面执行：异步方法的 callback 被放到了 pool 里面来安排调用，通常情况下，比同步方法调用慢。
    Timer.run(() { // 立即执行的 timer
      attachRootWidget(rootWidget);
    });
}

// 拿到 Widget，附加到 renderViewElement 上面
void attachRootWidget(Widget rootWidget) {
    _readyToProduceFrames = true;
		// 从 RenderObject 到 Element Tree 的桥接
    _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget,
		// 膨胀组件设置 RenderObject 为容器子节点
    ).attachToRenderTree(buildOwner, renderViewElement as RenderObjectToWidgetElement<RenderBox>);
  }

void _invokeFrameCallback(FrameCallback callback, Duration timeStamp, [ StackTrace callbackStack ]) {
    assert(callback != null);
    assert(_FrameCallbackEntry.debugCurrentCallbackStack == null);
		// 是在 debug 下才执行 callback 中代码
    assert(() {
      _FrameCallbackEntry.debugCurrentCallbackStack = callbackStack;
      return true;
    }());
}
```

上面 assert + callback 的使用，是为了方便在 debug 下执行 callback 的逻辑。

- 只有 debug mode 开启 assert
- assert 第一个参数是 true 时才接着执行，false 时报错

## 三、Flutter 如何布局

> The core of Flutter’s layout mechanism is widgets. In Flutter, almost everything is a widget—even layout models are widgets. The images, icons, and text that you see in a Flutter app are all widgets. But things you don’t see are also widgets, such as the rows, columns, and grids that arrange, constrain, and align the visible widgets.

Flutter 布局机制的核心就是 widget。在 Flutter 中，万物皆组件，甚至是布局模型也是组件。Flutter app 中能看到的 Image、Icons 和 Text 都是组件。有些不可见的，例如用来排列、约束和对齐可见组件的 rows、columns 和 grids 等也是组件。

常用的布局组件见 [https://flutter.dev/docs/development/ui/widgets/layout](https://flutter.dev/docs/development/ui/widgets/layout)

## 四、Flutter 如何做渲染

### 4.1 Flutter 架构解析

先看下 Flutter 的架构图：
![Flutter 架构](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdnrfyfj30i309raam.jpg)
再重新理解下涉及到的概念：Widget、Element 和 RenderObject

Widget：

> A widget is an immutable description of part of user interface. Describes the configuration for an [Element]. Widget 就是一个对 UI 的不可变的描述。描述 Element 的配置。简单来说就是用来配置 UI 的。

Element:

> An instantiation of a Widget at a particular location in the tree. 组件树指定位置的 Widget 实例。简单来说就是方便管理 Widget 和 RenderObject 对象。

主要的职责：

- 更新和改变 UI
- 管理 Widget 的生命周期

RenderObject：

> When Flutter draws your UI, it does not look at the tree of Widgets, but looks at the Render Objects’ one which controls all of the sizes, layouts and keeps all logic for painting the actual widget. That’s the reason why Render Object is very expensive to instantiate.

> 当 Flutter 绘制 UI 的时候，不会依赖 Widget 树，但是会依赖 Render Object 树，Render Object 控制尺寸、布局以及实际组件的绘制逻辑，正因如此 Render Object 实例化逻辑很重。简单来说就是控制尺寸、布局和绘制，负责绘制到屏幕上。

Widget、Element、RenderObject 的职责见下图：
![三棵树](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdn4gv4j311y0lemzg.jpg)

![三棵树职责](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdm68daj311y0liju4.jpg)

### 4.2 Element 和 RenderObject 为什么要分开

RenderObject 和 Widget 是同构的，RenderObject 树是 Element 树的子集。一种简单的实现方式是把两种树合成一个，但是，将两种树分开会有如下好处。

- 性能好：当布局变化的时候，布局树上只有相关的部分才会被遍历。合成的时候，Element 树有许多可以跳过的附加节点。
- 清晰：概念的分离，widget 协议和 render object 协议能更好的满足自己的需求，简化 API，减少 Bug 和测试负担。
- 类型安全：Render object 树能够保证运行时子节点类型合适，更加的类型安全。因为合成的组件在布局的时候无法感知所在的坐标系，所以校验 element 树中 render object 的类型需要遍历整棵树。

注：每个坐标系中都有对应的 Render object 类型，在 App Model 中相同的组件可以使用 box 布局，也可以使用 sliver 布局。具体参见 [https://stackoverflow.com/questions/53590842/whats-the-difference-between-box-layout-model-and-sliver-layout-model-boxconst](https://stackoverflow.com/questions/53590842/whats-the-difference-between-box-layout-model-and-sliver-layout-model-boxconst)
总结：Widget 是描述 UI 的树，Element 是 Widget 的实例，用来管理 Widget 和 RenderObject，RenderObject 树用来做渲染。

### 4.3 渲染 pipeline

![Pipeline](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdlcpvxj30ih09c0t3.jpg)
Widget build 完成后，会调用 widget 的 createElement 创建对应 Element，然后 Element 调用 createRenderObject 创建对应的 RenderObject 对象。

CreateElement 的调用栈如下：
![调用栈](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggmbdmmbkxj30gx0gfwf9.jpg)
组件更新的时候会调用 Widget 的 canUpdate 方法来判断组件是否需要更新，需要判断 Widget 的 runtimeType 和 key 均一致才可以。

```dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
}
```

如果 canUpdate 为 true，代表 RenderObject 可以复用。RenderObjectElement 会调用 updateRenderObject 方法，更新已有的 RenderObject 的属性值。

```dart
void updateRenderObject(BuildContext context, covariant RenderObject renderObject) { }
```

如果 canUpdate 为 false。Element 会重新创建 RenderObject 和 Element 对象，不复用已有对象。

总结：有三棵树的存在，主要是为了当有变化发生的时候，更多的复用，更少的创建新的对象，所以性能表现更好。

参考：

- [https://www.youtube.com/watch?v=996ZgFRENMs](https://www.youtube.com/watch?v=996ZgFRENMs)
- [https://www.youtube.com/watch?v=UUfXWzp0-DU](https://www.youtube.com/watch?v=UUfXWzp0-DU)

## 五、引用

- [Tips to use timer in dart and flutter](https://fluttermaster.com/tips-to-use-timer-in-dart-and-flutter/)
- [Dart language keywords](https://dart.dev/guides/language/language-tour#keywords)
- [Features to a class mixins](https://dart.dev/guides/language/language-tour#adding-features-to-a-class-mixins)
- [Dart assert usage](https://dart.dev/guides/language/language-tour#assert)
- [光栅化 Rasterize](https://www.computerhope.com/jargon/r/rasterize.htm)
- [How flutter renders widgets](https://medium.com/manabie/how-flutter-renders-widgets-fd6eca945a04)
