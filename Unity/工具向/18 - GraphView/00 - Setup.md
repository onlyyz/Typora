<font color=#4db8ff>Link：https://www.youtube.com/watch?v=7KHGH0fPL84</font>

在项目中 以这个为架构，接入Odin

 

### 一、Start

#### 1.1 Dialogue

首先创建三个脚本<font color=#FFCE70>DialogueGraph、DialogueGraphView、DialogueNode</font>，分别继承<font color=#bc8df9>EditorWindow、GraphView、Node</font>

<font color=#FFCE70>DialogueNode</font>需要存储数据，因此需要公布一些数据

<font color=#4db8ff>每个节点传递 的唯一ID</font>

<font color=#4db8ff>对话内容</font>

<font color=#4db8ff>入口Bool值</font>

```c#
using System;
using UnityEditor.Experimental.GraphView;

public class DialogueNode : Node
{
    public string GUID;
    public String DialogueText;
    public bool Entrybool = false;
}
```

<font color=#FFCE70>DialogueGraphView</font>，需要一个构造函数去添加鼠标相关控制器

内容拖动器、选择拖动器、矩阵选择器

```C#
using UnityEditor.Experimental.GraphView;
using UnityEngine.UIElements;

public class DialogueGraphView : GraphView
{
    public DialogueGraphView()
    {
         //  放大/缩小。
        SetupZoom(ContentZoomer.DefaultMinScale, ContentZoomer.DefaultMaxScale);
        
        this.AddManipulator(new ContentDragger());
        this.AddManipulator(new SelectionDragger());
        this.AddManipulator(new RectangleSelector());
    }
}
```

<font color=#FFCE70>DialogueGraph</font>利用一个静态方法打开<font color=#bc8df9>GraphView</font>，并且添加一个菜单栏属性。<font color=#4db8ff>以便在编辑器中静态调用它</font>

```C#
using UnityEditor;
using UnityEngine;

public class DialogueGraph : EditorWindow
{
   [MenuItem("Tool/18-Dialogue")]
   public static void OpenDialogueGraphWindow()
   {
      var Window = GetWindow<DialogueGraph>();
      Window.titleContent = new GUIContent("Dialogue Graph");
   }
} 
```

可以看到编辑器，但是看不到编辑器窗口

<img src="assets/image-20230925194603485.png" alt="image-20230925194603485" style="zoom: 67%;" />

#### 1.2 OnDisable

添加启用和禁用方法（<font color=#66ff66>OnEnable 和 OnDisable</font>），并且添加一个私有变量<font color=#FFCE70>GraphView</font>，如同启用方法一样，在里面获得一个新实例。

```C#
using UnityEditor;
using UnityEngine;

public class DialogueGraph : EditorWindow
{
   private DialogueGraphView _graphView;
   [MenuItem("Tool/18-Dialogue")]
   public static void OpenDialogueGraphWindow()
   {
      var Window = GetWindow<DialogueGraph>();
      Window.titleContent = new GUIContent("Dialogue Graph");
   }

   private void OnEnable()
   {
      _graphView = new DialogueGraphView()
      {
         name = "Dialogue Graph"
      };
   }
   private void OnDisable()
   {}
}
```

将整个<font color=#bc8df9>Graph View</font>填满<font color=#bc8df9>Editor Window</font>，随后利用<font color=#4db8ff>Root</font>加入<font color=#bc8df9>Editor Window</font>

```C#
private void OnEnable()
{
_graphView = new DialogueGraphView()
{
 name = "Dialogue Graph"
};

_graphView.StretchToParentSize();
rootVisualElement.Add(_graphView);
}
```

在禁用窗口的时候将其删除，但是此时的数据并没有保存 

```C#
private void OnDisable()
{
  rootVisualElement.Remove(_graphView);
}
```

![image-20230926101606899](assets/image-20230926101606899.png)

#### 1.3 Node

添加一个节点，首先创建一个生成条目的方法节点，创建一个<font color=#bc8df9>DialogueNode</font>节点实例，随后传递一系列参数。利用<font color=#FFCE70>C#</font> Gooood类，生成一个新的<font color=#4db8ff>GUID</font>

```c#
public class DialogueGraphView : GraphView
{
    public DialogueGraphView()
    {
        this.AddManipulator(new ContentDragger());
        this.AddManipulator(new SelectionDragger());
        this.AddManipulator(new RectangleSelector());

        GenerateEntryPointNode();
    }

    private void GenerateEntryPointNode()
    {
        var node = new DialogueNode()
        {
                title = "start",
                GUID = Guid.NewGuid().ToString(),
                DialogueText = "ENTRYPOINT",
                // EntryPoint = true
        };
    }
}
```

设置节点出现的位置，并且将其加入<font color=#4db8ff>Graph View</font>

```c#
public DialogueGraphView()
    {
        this.AddManipulator(new ContentDragger());
        this.AddManipulator(new SelectionDragger());
        this.AddManipulator(new RectangleSelector());

        //添加到Graph View 基类
        AddElement(GenerateEntryPointNode());
    }

    private DialogueNode GenerateEntryPointNode()
    {
        var node = new DialogueNode()
        {
                title = "start",
                GUID = Guid.NewGuid().ToString(),
                DialogueText = "ENTRYPOINT",
                // EntryPoint = true
        };
        node.SetPosition(new Rect(100,200,100,150));
        return node;
    }
```

#### 1.4 port

增加一个生成<font color=#FFCE70>Port</font>的方法，参数为<font color=#4db8ff>Node，Direction，Capacity</font>。将这些信息填入<font color=#4db8ff>Node</font>中

```c#
private Port GenerateEntryPort(DialogueNode node, Direction portDirection, Port.Capacity capacity = Port.Capacity.Single)
{
    //不是在端口之间传递数据，并非是VFX shader，这里仅仅是检查是否有链接
    return node.InstantiatePort(Orientation.Horizontal, portDirection, capacity, typeof(float));
}

```

随后在生成<font color=#4db8ff>Node</font>的方法中调用它

```c#
var generatedPort = GeneratePort(node, Direction.Output);
generatedPort.portName = "Neat";
node.outputContainer.Add(generatedPort);
node.SetPosition(new Rect(100,200,100,150));
```

![image-20230926144048295](assets/image-20230926144048295.png)

但是此时节点的状态并不正确，添加一段更新节点的代码

```c#
node.outputContainer.Add(generatedPort);
//Refresh node Status
node.RefreshExpandedState();
node.RefreshPorts();

node.SetPosition(new Rect(100,200,100,150));
```

![image-20230926145403273](assets/image-20230926145403273.png)

#### 1.5 ToolBar

整理函数，随后增加一个创建工具栏的节点，并且利用<font color=#66ff66>button</font>去触发节点的生成

```c#
//DialogueGraph
private void OnEnable()
{
    //Tool Graph Virw
    GenerateToolbar();
    ConstructGraphView();
}
private void GenerateToolbar()
{
    var toolbar = new Toolbar();
    var nodeCreateButton = new Button(() =>{
        _graphView.CreateDialogueNode();
    });
}
private void ConstructGraphView()
{
    _graphView = new DialogueGraphView()
    {name = "Dialogue Graph"};
    //Graph View填满窗口 
    _graphView.StretchToParentSize();
    rootVisualElement.Add(_graphView);
}
```

![image-20230926151555219](assets/image-20230926151555219.png)

我们增加一个创建<font color=#bc8df9>CreateDialogueNode</font>的方法，创建一些信息给节点，并且在最后刷新节点信息

```c#
//没有加入Graph view
    public DialogueNode CreateDialogueNode(String nodeName)
    {
        var dialogueNode = new DialogueNode()
        {
            title = nodeName,  
            DialogueText = nodeName,
            GUID = Guid.NewGuid().ToString()
        };
        //Port the input and output
        var inputPort =  GeneratePort(dialogueNode, Direction.Input,Port.Capacity.Multi);
        inputPort.portName = "Input";
        dialogueNode.outputContainer.Add(inputPort);
        
        
        var outputPort = GeneratePort(dialogueNode, Direction.Output,Port.Capacity.Multi);
        outputPort.portName = "Output";
        dialogueNode.outputContainer.Add(outputPort);
       
        dialogueNode.RefreshExpandedState();
        dialogueNode.RefreshPorts();
        dialogueNode.SetPosition(new Rect(Vector2.zero, defaultNodeSize));

        return dialogueNode;
    }
```

但是该<font color=#4db8ff>Node</font>并不会出现在<font color=#4db8ff>Graph View</font>上，因为，我们并将它没有加入<font color=#bc8df9>Graph View</font>，我们可以利用<font color=#66ff66>AddElement</font>去添加，设计一个新的函数创建<font color=#4db8ff>Node</font>的同时将其添加到<font color=#FFCE70>Graph View</font>

```c#
//Dialogue Graph View
//添加节点到Graph view ，同时创建node
public void CreateNode(String nodeName)
{
    AddElement(CreateDialogueNode(nodeName));
}
```

并且该函数由<font color=#bc8df9>Dialogue Graph</font>中的函数<font color=#66ff66>GenerateToolbar</font>调用

```c#
private void GenerateToolbar()
{
    var toolbar = new Toolbar();
    var nodeCreateButton = new Button(() =>
    {
        _graphView.CreateNode("Dialogue Node");
    });
    nodeCreateButton.text = "Create Node";

    toolbar.Add(nodeCreateButton);
    rootVisualElement.Add(toolbar);
}
```

#### 1.6 Node Link

但是现在的节点之间并没有办法链接，因为他们之间的数据传递、判断没有一个规范，因此我们可以利用函数

<font color=#66ff66>GetCompatiblePorts</font>，对其覆盖，应用我们自己的连接规则

不能自己连接自己

```c#
//节点连接兼容问题，获取与给定端口兼容的所有端口
public override List<Port> GetCompatiblePorts(Port startPort, NodeAdapter nodeAdapter)
{
    //兼容端口
    var compatiblePorts = new List<Port>();

    ports.ForEach((port =>
    {
        if (startPort !=port && startPort.node != port.node)
        {
            compatiblePorts.Add(port);
        }
    }));
    return compatiblePorts;
}
```

#### 1.7 Output Branch

利用<font color=#66ff66>Button</font>来添加分支的数量，利用<font color="red">Lambda</font>表达式去调用一个新的函数方法

```c#
//Port the input and output
    var inputPort =  GeneratePort(dialogueNode, Direction.Input,Port.Capacity.Multi);
    inputPort.portName = "Input";
    dialogueNode.outputContainer.Add(inputPort);

    var button = new Button(() => {  AddchoicePort(dialogueNode); });


    dialogueNode.RefreshExpandedState();
    dialogueNode.RefreshPorts();
```

如同之前一样，生成一个<font color=#66ff66>Port</font>，我们想要更改每个<font color=#66ff66>Port </font>的名称，可以利用<font color=#4db8ff>Query</font>去搜索查找，该<font color=#bc8df9>Node</font>的<font color=#FFCE70>Container</font>中的<font color=#66ff66>Port</font>，随后将<font color=#66ff66>Port</font>的名字指定为输出<font color=#66ff66>Port</font>的技术



```c#
 private void AddchoicePort(DialogueNode dialogueNode)
    {
        
        var GeneratedPort = GeneratePort(dialogueNode, Direction.Output);
        //搜索端口名称指定为选择端口计数
        var outputPortCount = dialogueNode.outputContainer.Query("connector").ToList().Count;
        GeneratedPort.portName = $"choice {outputPortCount}";

        dialogueNode.outputContainer.Add(GeneratedPort);
        dialogueNode.RefreshPorts();
        dialogueNode.RefreshExpandedState();
    }
```

<font color=#66ff66>Query  </font><font color=#4db8ff>Link：</font>https://docs.unity3d.com/Manual/UIE-UQuery.html

随后将其添加到<font color=#4db8ff>Node</font>的<font color=#bc8df9>titleContainer</font>中

```c#
//输出port的数量
    var button = new Button(() => {  AddchoicePort(dialogueNode); });
    button.text = "New Choice";
    dialogueNode.titleContainer.Add(button); 
```

![image-20230926182034871](assets/image-20230926182034871.png)

#### 1.8 USS

添加背景格式

```c#
//  读取 uss 文件并将其添加到样式中
this.styleSheets.Add(Resources.Load<StyleSheet>("20-GraphView/Uss/GraphViewBackGround"));
//  在图层最底层添加背景
this.Insert(0, new GridBackground());
```
