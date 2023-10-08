<font color=#4db8ff>Video Link：https://www.youtube.com/watch?v=yBM112uokM8&list=PL0yxB6cCkoWK38XT4stSztcLueJ_kTx5f&index=2</font>

<font color=#4db8ff>Git Link：</font>https://github.com/Wafflus/unity-dialogue-system



#### 二、Based

#### 2.1 Custom IManipulator

```C#
private IManipulator CreateNodeContextualMenu()
{
    ContextualMenuManipulator contextlMenuManipulartor = new ContextualMenuManipulator();
    return contextlMenuManipulartor;
}
```

<font color=#4db8ff>Link：</font>https://docs.unity3.com/ScriptReference/UIElements.ContextualMenuManipulator.html

参数为<font color=#FFCE70>Action</font>，利用<font color=#4db8ff>回调函数调用</font>，这是我们右键时将要填充的<font color="red">菜单</font>

```c#
menuEvet => menuEvet.menu.AppendAction();
```

其中<font color=#66ff66>AppendAction</font>接收两个参数，一个是<font color=#4db8ff>标题</font>、另一个是执行时的<font color=#4db8ff>操作</font>

操作为添加节点，位置为注册事件中的当前鼠标位置

```c#
private IManipulator CreateNodeContextualMenu()
{
    ContextualMenuManipulator contextlMenuManipulartor = new ContextualMenuManipulator
        (
        menuEvet => menuEvet.menu.AppendAction("Add Node", 
        actionEvent => AddElement(CreateNode(actionEvent.eventInfo.localMousePosition)))
    );
    return contextlMenuManipulartor;
}
```

<font color=#4db8ff>DropdownMenuEventInfo Link：https://docs.unity3d.com/ScriptReference/UIElements.DropdownMenuEventInfo.html</font>

这一部分可以取代旧代码中的在<font color=#FD00FF>SearchWindow</font>的位置设置

```c#
 //位置换算
            var worldMousePosition = graphWindow.rootVisualElement.ChangeCoordinatesTo(graphWindow.rootVisualElement.parent, context.screenMousePosition - graphWindow.position.position);
            var localMousePosition = graphView.contentViewContainer.WorldToLocal(worldMousePosition);
            node.SetPosition(new Rect(localMousePosition, new Vector2(100, 100)));
```

#### 2.2 Node

节点上使用继承机制

<font color=#4db8ff>Git Link：</font>https://github.com/onlyyz/Custom/commit/1102ab42ee76de91daa88ef6c4e484afee95d4c3

我们可以利用<font color=#66ff66>Type</font>，自动识别不同类型的<font color=#4db8ff>Node</font>，它根据给定的<font color=#FFCE70>Name</font>返回类型<font color=#bc8df9>type</font>，我们通过在传递一个前面待优<font color="red">（$）</font>符号的字符串，<font color=#bc8df9>C#</font>将变量放在括号<font color=#4db8ff> { } </font>中来将变量传递给字符串

其中字符串需要输入<font color="red">Class</font>的整个路径

这里通过传递<font color=#4db8ff>SingleChoice</font>枚举量，我们将得到<font color=#66ff66>DS.Elements.DSSingleChoiceNode</font>

```c#
Type nodeType = Type.GetType();
Type nodeType = Type.GetType($"{dialogueType}");
Type nodeType = Type.GetType($"DS.Elements.DS{dialogueType}Node");
```

<font color=#4db8ff>String：</font>https://learn.microsoft.com/en-us/dotnet/csharp/tutorials/string-interpolation

