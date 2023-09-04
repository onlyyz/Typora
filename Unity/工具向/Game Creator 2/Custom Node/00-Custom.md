## Creating an Instruction

我们可以创建模版代码： Create → Game Creator → Developer → C# Instruction

```C#
using System;
using System.Threading.Tasks;
using GameCreator.Runtime.Common;
using GameCreator.Runtime.VisualScripting;

[Serializable]
public class MyInstruction : Instruction
{
    protected override Task Run(Args args)
    {
        // Your code here...
        return DefaultResult;
    }
}
```

#### 1.1 Anatomy of an Instruction

指令是一个继承自 <font color=#66ff66>Instruction </font>超级类的类。抽象运行（...）方法是指令执行的入口点，当执行该指令时，该方法会被自动调用。

Run(...) 方法有一个 Args 类型的参数，Args 是一个辅助类，包含对启动调用的游戏对象（args.Self）和目标游戏对象（args.Target）（如果有）的引用。

#### 1.2 Yielding in Time

大多数指令将在一帧内执行完毕。不过，有些指令可能需要暂停执行一段时间后才能继续执行。最简单的例子就是 "<font color="DarkOrchid ">Wait for Seconds</font>"指令，它会暂停执行几秒钟，然后再继续执行。

<font color="DarkOrchid ">Instruction</font>超类包含一系列有助于时间管理的方法。

#### 1.3 Async

指令使用<font color="red"> async/await </font>方法来管理指令在一段时间内的流程。使用 <font color=#66ff66>await </font>符号要求 <font color=#66ff66>Run()</font> 方法的方法定义中包含 <font color=#4db8ff>async </font>符号：

```C#
protected override async Task Run(Args args) { }
```

#### 1.4  NextFrame

<font color=#4db8ff>NextFrame()</font> 方法会将指令的执行暂停一帧，然后继续执行。

```C#
protected override async Task Run(Args args)
{
    await this.NextFrame();
}
```

#### 1.5 Time

<font color=#66ff66>Time(float time)</font> 方法会在一定时间内暂停指令的执行。时间参数以秒为单位。

```C#
protected override async Task Run(Args args)
{
    await this.Time(5f);
}
```

#### 1.6 While

<font color=#66ff66>While(Func<bool> function) </font>方法暂停指令的执行，只要作为参数传递的方法的结果返回 true。该方法每帧执行一次，一旦返回 false，就会立即恢复执行。

```C#
protected override async Task Run(Args args)
{
    await this.While(() => this.IsPlayerMoving());
}
```

#### 1.7 Until

<font color=#66ff66>Until(Func<bool> function) </font>方法暂停指令的执行，只要作为参数传递的方法的结果返回 true。该方法每帧执行一次，一旦返回 true，执行将立即恢复。

```C#
protected override async Task Run(Args args)
{
    await this.Until(() => this.PlayerHasReachedDestination());
}
```

## Decoration & Documentation

强烈建议记录和<font color=#66ff66>decorate </font>指令，以便于查找和使用。可以使用类类型属性来完成，这些属性会告知游戏创建者该特定指令的特殊性。

例如，要将指令的标题设置为 "Hello World"，可使用类定义上方的 [Title(string name)] 属性：

#### 2.1 Title

指令的标题。如果不提供该属性，标题将是类名的美化版。

```C#
[Title("Title of Instruction")]
```

#### 2.2 Description

说明<font color=#66ff66>Instruction </font>指令的作用。这既用于浮动窗口文档，也用于向游戏创建者中心上传指令时的描述文本

```C#
[Description("Lorem Ipsum dolor etiam porta sem magna mollis")]
```

#### 2.3 Image 

<font color=#66ff66>[Image(...)]</font>属性可将指示的默认<font color=#FFCE70>icon </font>更改为其中一个默认<font color=#FFCE70>icon </font>。它由 2 个参数组成：

<font color=#66ff66>Icon [Type]：</font>一个 IIcon 派生类的 Type 类。虽然 Game Creator 自带了大量图标，但您也可以创建自己的图标。

<font color=#66ff66>Color [Color]：</font><font color=#FFCE70>icon </font>的颜色：<font color=#FFCE70>icon </font>的颜色。使用 Unity 的颜色类。

例如，"Solid Cube（实心立方体）"<font color=#FFCE70>icon </font>就是其中一个<font color=#FFCE70>icon </font>。要显示红色实心立方体作为指令图标，请使用以下属性：

```C#
[Image(typeof(IconCubeSolid), Color.red)]
```

#### 2.4 Version

<font color="red">version </font>，用于跟踪本指令的开发情况。需要注意的是，在游戏创建者中心更新指令时，版本号必须始终高于服务器上的版本号。

语义版本遵循标准的主要版本（Major Version）、次要版本（Minor Version）和补丁版本（Patch Version）。要进一步了解语义版本的工作原理，请阅读以下页面： https://semver.org。

```C#
[Version(1, 5, 3)]
```

#### 2.5 Example

> ```C#
> [Example(@"    This is the first paragraph.    This is also in the first paragraph, right after the previous sentence     This line is part of a new paragraph. )]
> ```
>
> 

#### 2.6 Dependency

```C#
[Dependency("gamecreator.inventory", 1, 5, 2)]
```

