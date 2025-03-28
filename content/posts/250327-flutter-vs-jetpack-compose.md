+++
title = "从对象到函数：Flutter 与 Jetpack Compose 的抽象范式对比"
description = "同为声明式 UI 框架，Flutter 和 Jetpack Compose 在设计理念上也存在一定差异。准确来说，Flutter 的「一切皆 Widget」和 Jetpack Compose 的「一切皆函数」是两种不同的抽象范式。"
date = 2025-03-27
updated =2025-03-27
draft = false

[taxonomies]
categories = ["android"]
tags = ["flutter", "jetpack compose"]

[extra]
mermaid = true
+++

同为声明式 UI 框架，Flutter 和 Jetpack Compose 在设计理念上也存在一定差异。准确来说，Flutter 的「**一切皆 Widget**」和 Jetpack Compose 的「**一切皆函数**」是两种不同的抽象范式。

# 1 抽象层面对比

| 维度          | Flutter                          | Jetpack Compose                  |
|---------------|----------------------------------|----------------------------------|
| **基本单元**   | Widget（不可变配置对象）         | Composable 函数（可重组函数）   |
| **核心抽象**   | 对象树（Widget → Element → RenderObject） | 函数调用树（基于 Gap Buffer 的智能重组） |
| **状态管理**   | 通过 StatefulWidget 与 State 对象的分离来管理状态 | 通过 remember 函数结合 mutableState 创建状态闭包 |
| **更新机制**   | 重建 Widget 树 + Diff 算法       | 标记重组范围 + 智能跳过           |

**关键区别**：

- Flutter 的 Widget 是**配置描述**（如 React 的 Virtual DOM），实际渲染由 RenderObject 处理。
- Jetpack Compose 的 Composable 函数是**执行指令**，直接生成渲染树节点（类似 Svelte 的编译时方案）。

# 2 Flutter 的 Widget 体系

首先，Flutter 基于 Dart，而 Dart 是「一切皆对象」的编程语言，Dart 中的每个值都是一个对象（包括数字、布尔值、函数，甚至 null 本身），这些对象都继承自顶层类 Object（在 Dart 2.12 引入空安全后，继承自 Object?）。这与 Flutter 所有 UI 组件都继承自 Widget 的特性相契合。

```dart
// 连布局约束也是 Widget
ConstrainedBox(
  constraints: BoxConstraints(maxWidth: 100),
  child: Text('Hi'),
)

// 动画也是 Widget
AnimatedOpacity(
  opacity: _visible ? 1.0 : 0.0,
  child: Button(),
)

// 手势检测还是 Widget
GestureDetector(
  onTap: () {},
  child: Container(),
)
```
本质上，Widget 是统一的配置单元，但实际运行时会被转化为不同实体：

- 布局 Widget → 创建 RenderObject
- 状态 Widget → 关联 State 对象
- 代理 Widget → 控制或修改子组件行为

Flutter 能在不同平台上保持 UI 的一致性，主要原因在于它实现了从 Widget 到渲染引擎的统一实现。与其他声明式框架类似，Flutter 也使用视图树（View Tree）来描述 UI 结构。不同的是，Flutter 对视图树的概念进行了扩展，将其抽象为三部分：Widget，Element 和 RenderObject。

- Widget 是由开发人员控制的对 UI 的结构化描述。Flutter 将 Widget 设计为不可变的，一旦创建就不能修改。因此当 UI 更新时，Flutter 会重建 Widget 树。
- Element 是 Widget 运行时的实体，它持有 Widget 的引用，同时还持有一个 RenderObject。也即是说，Element 是 Widget 和 RenderObject 之间的桥梁，承接了 UI 构建的上下文数据。
- RenderObject 是实际的渲染实体，它负责创建渲染对象，组成渲染对象树。渲染对象树经过布局、绘制，最后通过渲染引擎 Skia/Impeller 进行合成和渲染。

于是，Flutter 的视图树结构如下：

{% mermaid() %}
graph LR
  A[Widget] --> B[Element]
  B --> C[RenderObject]
  C --> D[Skia/Impeller]
{% end %}

# 3 Jetpack Compose 「一切皆函数」的本质

Jetpack Compose 的声明式是基于 Kotlin DSL 实现的，Composable 本质上就是一个 Kotlin 函数，通过 Kotlin 的尾 Lambda 语法特性让 Composable 之间能够嵌套，形成 Composable 的树形层级，实现不输于 XML 的结构化表达能力。

Jetpack Compose 的「一切皆函数」体现在：

- 一切组件都是函数，由于没有类的概念，因此不会有任何继承的层次结构，所有组件都是顶层函数，可以在 DSL 中直接调用。
- Composable 函数通过多级嵌套形成结构化的函数调用链，函数调用链经过运行后生成一棵 UI 视图树。
- 视图树一旦生成便不可随意改变。视图的刷新依靠 Composable 函数的反复执行来实现。
- Composable 函数必须幂等、无返回值，且执行顺序敏感。
- Composable 函数只能在 Composable 函数中调用。

```kotlin
// 约束布局
Box(Modifier.width(100.dp)) {
  Text("Hi")
}

// 动画
AnimatedVisibility(visible) {
  Button()
}

// 手势检测
Box(Modifier.clickable {}) 
```

因此，Composable 函数本质上是：

- 带记忆能力的函数（通过 Positional Memoization）。
- 编译期转换为节点操作指令。
- 无虚拟 DOM 概念，直接操作组合树。

# 4 设计理念总结

Flutter 的「一切皆 Widget」类似于用**乐高积木**搭建 UI：

- **标准化模块**：每个 Widget 都是预定义的积木块（如 `Text`、`Container`），通过固定接口（属性参数）组合。
- **不可变性**：乐高积木一旦拼装完成，修改需要替换整块（Widget 重建）。
- **显式组装**：必须手动连接积木（通过 `child`/`children` 嵌套），层级关系严格。

```dart
// 类似用乐高拼装一辆车
Column(
  children: [
    WheelWidget(size: 20), // 轮子
    BodyWidget(color: Colors.red), // 车身
    RoofWidget(shape: Rectangle()), // 车顶
  ],
)
```

其显著优势是**跨平台一致性**（同一套积木在任何平台拼出相同效果），局限则是灵活性较低（无法直接修改积木内部结构）。

而 Jetpack Compose 的「一切皆函数」更像用**粘土**塑造 UI：

- **自由塑形**：通过函数调用直接捏出 UI（如 `Box { Text(...) }`），无需预定义组件层级。
- **动态重组**：粘土可随时重塑（函数重组），只需修改受影响部分（智能跳过未变化区域）。
- **隐式连接**：通过作用域（如 `Modifier`) 传递属性，类似粘土的延展性。

```kotlin
// 类似用粘土捏一辆车
CarScope {
  Wheel(size = 20.dp)  // 轮子
  Body(color = Color.Red) // 车身
  Roof(shape = RectangleShape) // 车顶
}
```
其优势在于**极致灵活**（可任意组合/拆解逻辑）。局限则是比较依赖平台渲染能力（「粘土材质」由 Android 原生决定）。

总结两者的设计理念本质差异如下：

| 维度          | Flutter（乐高）                | Jetpack Compose（粘土）         |
|---------------|-------------------------------|-------------------------------|
| **抽象单元**   | 对象（Widget）                | 函数（Composable）            |
| **组合方式**   | 嵌套组装                      | 作用域内调用                  |
| **修改成本**   | 重建 Widget 树               | 局部重塑                      |
| **设计隐喻**   | 拼装标准化零件                | 自由雕刻材料                  |
