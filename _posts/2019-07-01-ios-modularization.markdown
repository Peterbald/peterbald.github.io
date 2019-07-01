---
layout: post
title: "组件化踩坑实践"
subtitle: ""
description: "iOS, 组件化, 实践, 坑"
author: "RandomJ"
header-img: "img/scotland.jpeg"
tags:
  - 组件化
  - iOS
---

#### 什么是组件化

在平时开发中，我们经常接触模块和组件两个概念，在维基百科中的定义如下：

> 软件模块（Software Module）是一套一致而互相有紧密关连的 [软件](https://zh.wikipedia.org/wiki/%E8%BB%9F%E9%AB%94) 组织。包含了 [程序](https://zh.wikipedia.org/wiki/%E7%A8%8B%E5%BC%8F) 和 [数据结构](https://zh.wikipedia.org/wiki/%E8%B3%87%E6%96%99%E7%B5%90%E6%A7%8B) 两个部分。软件模块是现代 [软件开发](https://zh.wikipedia.org/wiki/%E8%BB%9F%E4%BB%B6%E9%96%8B%E7%99%BC) 往往利用模块作合成的单位。模块的 [接口](https://zh.wikipedia.org/wiki/%E4%BB%8B%E9%9D%A2) 表达了由该模块提供的功能和调用它时所需的元素。模块是可能分开地被编写的单位，能允许广泛人员同时协作、编写及研究不同的模块。

> 软件组件，定义为自包含的、可编程的、可重用的、与语言无关的 [软件](https://zh.wikipedia.org/wiki/%E8%BD%AF%E4%BB%B6) 单元。软件组件可以很容易被用于组装 [应用程序](https://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F) 中 。

这两个概念从定义上很难分清，这里引用下李运华老师在《从 0 开始学架构》的定义：

> 从逻辑的角度来拆分系统后，得到的单元就是“模块”；从物理的角度来拆分系统后，得到的单元就是“组件”。划分模块的主要目的是职责分离；划分组件的主要目的是单元复用。

那具体什么是组件化开发，我理解的就是将系统拆分成职责明确的可复用组件，每个组件足够高内聚、低耦合，方便维护和替换。

#### 为什么要做组件化

从上面组件和模块的定义可以看出，模块划分是为了职责分离，组件划分是为了单元复用。其实我们在做组件化的时候两方面都有，随着项目的开发人员逐渐增加，大家在一个工程中同时开发需求的时候不得不面对各种各样的冲突和依赖，一方面我们希望通过拆分不同的模块来隔离开发时的影响，另一方面我们希望在不同的工程中使用更多的可复用组件，减少 CV 操作和开发维护成本。

#### 如何决定组件化拆分粒度

组件化拆分应该是渐进式的，拆分的粒度也应该是由粗到细的。常见的分层思想应该是划分成业务组件、基础组件。美团在自己的 iOS 复用过程中抽象出了平台适配层(也就是 Adapter)，主要职责是在复用的时候处理各个平台的调用差异化，个人认为这一层可以根据需要来决定是否采用，如果是刚开始做组件化拆分，可以将这部分逻辑直接放在业务层处理(可能导致业务代码中有 CV)。

#### 组件化常用方式及优劣

业内的组件化方式常用的有以下几种：

- `URLRouter`
- `CTMediator`
- `Protocol + Class`

这方面业内分析的人很多，我就不再多赘述了，简单做个对比如下：

- `URLProtocol`：将 `URL` 和具体的模块进行映射，优点是比较方便做运营配置。缺点是没有区分本地调用和远程调用，将本地调用包装成 `URL` 的形式不是很合理。
- `CTMediator`：通过 `Runtime` 的方式解耦，包装成 `Target-Action` 形式方便调用，由组件提供方维护 `Category` 接口方便业务方调用。优点是调用类型分为本地调用和远程调用。缺点是在 `Target` 里面和 `Category` 中有两层硬编码，可能维护起来稍显不友好。
- `Protocol + Class`：通过注册 `Protocol` 和 `Class` 的映射关系，模块都依赖 `Protocol` 接口，不依赖具体的实现。比较有代表性的是阿里的 `BeeHive`，基于 `Spring` 的 `Service` 理念，使模块间的具体实现与接口解耦，但无法避免对接口类的依赖关系。

#### 如何选择适合自己的方式

基于上面对组件化实现方案的调研，因为我们是 `Swift` 项目，社区还没有特别成熟的方案。所以我们最终基于 `Protocol` 实现了自己的 `DDRouter` 来实现本地调用。并且在 `DDRouter` 上面加了一个 `URLRouter` 来解析远程调用的参数。

#### 组件化中需要注意什么

组件化我们最后的要抽出可以独立运行的组件，组件内部包含代码和资源，资源方面主要包含图片、`Lottie` 动画资源和多语言 `LocalizedStrings` 等。主要需要注意的有两点：

- 组件解耦的时候依赖的代码该如何处理：这部分代码我们最开始是直接将一些依赖下沉到项目的 `Common` 仓库，然后在后续整理拆分。工具类等下沉到具体的基础 `Kit`。`Adapter` 的部分，暂时先放到业务代码中，后续可以下沉到平台层。
- 组件依赖的资源访问路径需要注意：直接使用 `cocoapods` 的 `resourceBundle` 来管理，组件内部维护统一访问 `Bundle` 的分类。

#### 组件化中遇到了什么坑

1. 组件化中的资源如何管理：
   > `Cocoapod` 推荐使用 `resource_bundle` 来管理，会默认生成一个 `.bundle` 文件
2. Asset 里面的名字和图片本身的名字不一样：

   > 不使用图片本身的名字来访问，直接使用 `xcassets` 来访问，现在 `Cocoapods` 支持 `Assets.car` 的生成

   ![Assets.car 文件访问](http://ww2.sinaimg.cn/large/006tNc79gy1g4kf72g4h0j30v606pwfd.jpg)

3. 图片资源如何批量替换

   > 图片因为之前有好多通过 `imageLiteral` 访问的，所以直接通过 `Xcode` 的正则替换更换图片资源访问方式。`imageNamed` 方式访问也是同理。

   ```swift
   #imageLiteral\(resourceName: "(.*)"\)
   UIImage.talentImageNamed\(“$1”\)
   ```

4. 图片资源和通用的 `View` 有耦合

   > 工程的组件化是渐进式的，可以在主工程中保留所有图片资源，等其他模块的耦合解掉的时候统一去掉该资源。

5. 国际化的语言路径不对

   > 国际化 `localizedString` 的访问路径是 `bundle`下面具体选择的的 `lproj` 文件的路径，只给 `bundle` 会采用默认的语言。

6. 涉及到被多个模块依赖，但是调用有差异
   - 暂时放到 Common 中处理调用
   - 直接拷贝出来一份处理：暂时这样处理，但是后期要抽象出平台层的抽象，或者直接给到默认配置。

### 引用

感谢下面文章和源码的作者大佬：

- [什么叫组件化开发? - 知乎](https://www.zhihu.com/question/29735633)
- [蘑菇街 App 的组件化之路 - Limboy’s HQ](https://limboy.me/tech/2016/03/10/mgj-components.html)
- [蘑菇街 App 的组件化之路·续](https://limboy.me/tech/2016/03/14/mgj-components-continued.html)
- [iOS 组件化实践(一)：简介 - 简书](https://www.jianshu.com/p/568e875abd48)
- [iOS 应用架构谈 组件化方案 - Casa Taloyum](https://casatwy.com/iOS-Modulization.html)
- [关于 Pod 库的资源引用](http://zhoulingyu.com/2018/02/02/pod-resource-reference)
- [Xcode 使用正则表达式替换 - 简书](https://www.jianshu.com/p/98c96f7c1509)
- [iOS 组件化方案探索 « bang’s blog](http://blog.cnbang.net/tech/3080/)
- [iOS 组件化方案 - 知乎](https://zhuanlan.zhihu.com/p/22565656)
- [美团外卖 iOS 多端复用的推动、支撑与思考 - 美团技术团队](https://tech.meituan.com/2018/06/29/ios-multiterminal-reuse.html)
