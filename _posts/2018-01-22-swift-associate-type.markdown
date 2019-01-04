---
layout: post
title: "为什么 Swift 关联类型的协议需要做为泛型约束使用(译)"
subtitle: ''
author: "RandomJ"
header-img: "img/background.jpeg"
tags:
  - Swift
  - iOS
---

#### 一、OC 协议：发消息

`OC` 的协议本质是消息的集合。例如，`UITableViewDataSource` 协议有请求 `sections` 个数和一个 `sections` 中的 `rows` 的个数。

#### 二、Swift 协议：消息 + 关联类型

`Swift` 协议也是消息的集合。`Swift` 的协议同时也包含关联类型。协议中的关联类型是类型占位符，实现协议的时候来填充具体的类型。

#### 三、关联类型：简化协议的实现

关联类型是一个强大的工具，让我们更方便的实现协议。

##### 例子：Swift Equatable 协议

例如，`Swift Equatable` 协议有一个比较两个值是否相等的函数。

```swift
static func ==(lhs: Self, rhs: Self) -> Bool
```

这个函数使用了 `Self` 这个关联类型。`Self` 的具体类型是由协议的具体实现来确定。假设有一个类型下面的结构体，那么这个时候 `Self` 就是 `Name`。

```swift
struct Name: Equatable {
	let value: String
}
```

这个时候，协议的具体实现如下：
```swift
static func ==(lhs: Name, rhs: Name) -> Bool
```
综上：`Equatable` 使用关联类型 `Self` 来约束比较相同类型的值的 `==` 函数。

#### 四、对比：OC NSObjectProtocol

`NSObjectProtocol` 也有一个 `isEqual(_:)` 方法，但是因为是 `OC` 的协议，不能用 `Self` 类型。具体的定义如下：

```swift
func isEqual(_ object: Any?) -> Bool		
```

`OC` 的协议无法约束参数类型为指定关联类型，因此所有遵守协议的类型都能用来比较。通常在实现中会先检测参数的类型与消息的接收者类型是否一致。
```swift
func isEqual(_ object: Any?) -> Bool {
	guard let other = object as? Name else { return false } 
	
	// 开始检查值是否相等
}
```
每一个 `isEqual(_:)` 的实现，在每次调用的时候都要做这样的检查。`Equatable` 协议不会做这样的检测，通过 `Self` 关联类型保证了对象满足条件。

#### 五、代价

> Protocol ‘SomeProtocol’ can only be used as a generic constraint because it has Self or associated type requirements.
> 
> 因为需要实现 Self 或者其他关联类型，该协议只能用做泛型约束。

关联类型是个强大的工具。所付出的代价就是

> error: protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements

**协议中使用关联类型，必须用做泛型。**

`泛型`也是一个占位符，调用泛型函数的时候，必须填充对应的类型。

从`泛型`和`关联类型`的调用和实现两个方面对比来看：

*   关联类型：实现方指定类型，调用方不指定。当你实现一个使用关联类型的函数的时候，你需要填充对应的类型，所以你知道实际的类型。调用方不知道你具体采用的类型。

*   泛型：调用方指定类型，实现方不指定。当你实现一个使用泛型的函数，不需要知道调用方具体采用的类型。你可以使用约束限制类型，但是你必须处理所有满足约束的类型。调用方指定具体的类型，但是你的代码必须处理传递的任意类型。

##### 例子： 调用 Equatable == 强制使用泛型

```swift
func checkEquals(left: Equatable, right: Equatable) -> Bool {
	return left == right
}
```

Swift 编译器调出如下错误：

```swift
error: MyPlaygroundSwift.playground:27:24: error: protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements
func checkEquals(left: Equatable, right: Equatable) -> Bool {
                       ^
```

##### 为什么不使用泛型，checkEquals 不起作用？

如果 `swift` 允许这样做会怎么样，我们来做个实验。

假如有两个遵守 `Equatable` 的类型，`Name` 和 `Age`。代码可能会是这样。

```swift
let name = Name(value: "")
let age = Age(value: 0)
let isEquals = checkEquals(name, age)
```

这样做没有任何意义，从以下两个方面来看：

*   行为：怎样运行这段代码？怎样实现 checkEquals 中的 == 调用。只有 (Names, Names) 和 (Age, Age) 才有意义，因为 Equatable 定义为 ==(Self, Self)。只调用 Name 或者 Age 的 == 会打破类型安全。

*   意义：有什么意义? Equatable 协议不仅仅是个类型，它还和另一个类型 Self 有关。如果你写`checkEquals(left: Equatable, right: Equatable)`，只是在讨论 `Equatable`，它的关联类型`Self`被忽略。不能只是明确`Equtable`，必须明确`Equtable where Self is (some type)`

这非常蛋疼，但是很重要，`checkEquals` 看起来会工作。你想比较两个 `Equtable` 类型。但是 `Equtable` 是一个不完整的类型，它其实是`equtable for some type`

`checkEquals(left: Equatable, right: Equatable)`中左边是一个`equtable for some type`，右边也是一个`equtable for some type`。左右两边并不是`equatable for the same type`。`Equatable ==`需要左右同一个类型，所以上面的情况`checkEquals`不起作用。

##### 使 checkEquals 处理所有的 Equatable + Self

`checkEquals`在`Equatable where Self is (some type)`不知道具体的`some type`。所以，必须处理所有的`Equatable 和 Self 类型`，`checkEquals`所有的类型`T`，`T`是`Equatable和它的关联类型`

具体的代码如下：

```swift
func checkEquals<T: Equatable>(left: T, right: T) -> Bool {
	return left == right
}
```

现在，类型 `T`是一个`Equatable`类型，`T`的关联类型`Self`，实现了`checkEquatable`方法。使用 Swift 的泛型为具体的类型实现了一个`配方`，不需要写具体的`checkEquals(left: Name, right: Name)`和`checkEquals(left: Age, right: Age)`。最后需要提取泛型函数来重构代码。

##### 例子：调用 NSObjectProtocol 的 isEqual(_:) 不需要泛型

使用 `NSObjectProtocol` 的 `checkEquals` 不需要使用泛型。

```swift
import Foundation

func checkEquals(
  left: NSObjectProtocol,
  right: NSObjectProtocol
) -> Bool {
  return left.isEqual(right)
}
```

写起来很简单，这种情况下还是允许我们以下调用：

```swift
let isEqual = checkEquals(name, age)
```

`Name` 可以和 `Age` 比较？当然不能，`isEqual`结果是`false`。`Name.isEqual(_:)` 会判断对象是不是 `Name` 类型。不像`Equabable`的`==`，所以每个`isEqual(:)`方法必须处理类型一致性。

#### 六、权衡

关联类型使 `Swift` 的协议比 `OC` 的更加强大。

`OC` 的协议捕获了对象和它的调用者间的关系。调用者能够使用协议发消息，消息的具体行为由遵守协议的对象来实现。

`Swift` 协议也能够捕获一个类型和多个关联类型之间的关系。`Equatable`协议通过`Self`关联一个类型。`SetAlgebra`协议将实现关联到类型`Element`。

通过对比`Equatable`的`==`和`NSObjectProtocol`的`isEqual(:)`，能够发现 Swift 实现协议比较简单，但是在使用协议的时候比较复杂。强大的实现也会使代码变得很复杂，所以在使用协议的时候，需要权衡使用关联类型的价值和处理他们的复杂程度。

希望通过这篇文章帮助你认识使用的协议和 `API`。如果你觉得有用，可以关注我们的`高级 Swift 训练营`。

#### 七、刨根问底：Self 是关联类型么？

`Self` 的行为很像关联类型，不同于其他关联类型，不需要指定 `Self` 关联的类型，`Self` 会自动关联到实现协议的类型。

但是协议中的错误提示是“有 `Self` 或者关联类型”，听起来很像是不同的类型。

为了找到答案，我找到了 `Swift` 编译器中 `AST` 的源码，具体的文档在[这里](https://github.com/apple/swift/blob/ebd701c4b72e29ea3e42c6a906bc46e136ebaaad/include/swift/AST/Decl.h#L2589-L2590)

> 每个协议有一个隐式构造的关联类型 Self，描述了遵守协议的类型。

**所以，Self 是一个关联类型**

#### 八、感谢原作者`Jeremy Sherman`，原文地址
> https://www.bignerdranch.com/blog/why-associated-type-requirements-become-generic-constraints/