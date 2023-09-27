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

### 二、Save&Load

#### 2.1 Call Back

在菜单栏目添加与文件<font color=#66ff66>Save 和 Load</font>相关的UI，添加一段文本作为文件名来执行<font color=#bc8df9>Graph View </font>，添加一个新的文本作为默认的<font color=#4db8ff>描述 & Name</font>

```c#
private string _fileName = "New Narrative"; 

private void GenerateToolbar()
{
    var toolbar = new Toolbar();

    var fileNameTextField = new TextField("File Name");
    //设置为新的描述
    fileNameTextField.SetValueWithoutNotify(_fileName);
 ...
}
```

随后将文本标记为<font color="red">Dirty</font>（脏标记），以此去修改该文件，当文本发生改变的时候，会调用内部函数重新绘制，否则文本字段不会使用新的值。随后添加一个回调函数，以此去改变<font color=#4db8ff>File Name</font>

其中<font color=#bc8df9>Variable</font>来自于<font color=#66ff66>Editor</font>中获取，可以将其添加到回调函数中

```c#
private string _fileName = "New Narrative"; 

private void GenerateToolbar()
{
var toolbar = new Toolbar();

var fileNameTextField = new TextField("File Name");
//设置为新的描述
fileNameTextField.SetValueWithoutNotify(_fileName);
//设置为Dirty 脏标记
fileNameTextField.MarkDirtyRepaint(); 
fileNameTextField.RegisterValueChangedCallback(evt =>_fileName = evt.newValue);
toolbar.Add(fileNameTextField);

toolbar.Add(new Button( ()=>SaveData()){text = "Save Data"});
toolbar.Add(new Button( ()=>LoadData()){text = "Load Data"}); 

...
}
```

![image-20230927111215037](assets/image-20230927111215037.png)

#### 2.2 NodeData

创建新的文件夹<font color=#4db8ff>Runtime</font>用于存放<font color=#66ff66>序列化的C# Class</font>，作为没有数据的对话框，会传递一些属性，并且保存单个节点的数据

保存节点的<font color=#bc8df9>GUID、Text、Position</font>，最后将这个类序列化

```c#
using System;
using System.Numerics;
[Serializable]
public class DialogueNodeData
{
    public String GUID;
    public string DialogueText;
    public Vector2 Position;
}
```

#### 2.3 LinkData

保存链接之间的数据，需要保存<font color=#4db8ff>Base Node、Target Node 、Port</font>，并且序列化

```c#
using System;
[Serializable]
public class NodeLinkData
{
    public string BaseNodeGuid;
    public string PortName;
    public string TargetNodeGuid;
}
```

#### 2.4 Container

创建一个容器脚本，继承<font color=#bc8df9>ScriptableObject</font>，方便实例出<font color=#4db8ff>Asset文件</font>，里面会存储所有的节点<font color=#4db8ff>Node</font>以及对象的<font color=#66ff66>Link</font>

需要创建两个容器去进行存储，并且这个类需要序列化

```c#
using System;
using System.Collections.Generic;
using UnityEngine;

[Serializable]
public class DialogueContainer : ScriptableObject
{
   public List<NodeLinkData> Nodelinks = new List<NodeLinkData>();
   public List<DialogueNodeData> DialogueNodeData = new List<DialogueNodeData>();
}

```

#### 2.5 Utility

<font color=#4db8ff>特殊文件夹 Link：</font>https://docs.unity3d.com/cn/current/Manual/SpecialFolders.html

 我们可以利用静态的方法，返回<font color=#66ff66>Graph State Utility</font>的实例，并且我们需要先获取保存的<font color=#66ff66>Graph view</font>

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class GraphSaveUtility : MonoBehaviour
{
   private DialogueGraphView _targetGraphView;
   
   public static GraphSaveUtility GetInstance(DialogueGraphView targetGraphView)
   {
      return new GraphSaveUtility
      {
         _targetGraphView = targetGraphView
      };
   }
    public void SaveGraph(string fileName)
   {}
   public void LoadGraph(string fileName)
   {}
}
```

将<font color=#66ff66>Save & Load</font>方法链接到<font color=#4db8ff>Button</font>上，并且判断文件名是否为空

```c#
private void RequestDataOperation(bool save)
{
    //检查文件名是否为空
    if (string.IsNullOrEmpty(_fileName))
    {
        EditorUtility.DisplayDialog("Invalid file Name!", " Please Enter a valid file name","OK");
        return;
    }
}
```

传递<font color=#bc8df9>Graph View</font>到<font color=#bc8df9>GraphSaveUtility</font>进行序列化存储数据，随后判断我们是否存储数据

```c#
private void RequestDataOperation(bool save)
{
    //检查文件名是否为空
    if (string.IsNullOrEmpty(_fileName))
    {
        EditorUtility.DisplayDialog("Invalid file Name!", " Please Enter a valid file name","OK");
        return;
    }
    
    //GraphSaveUtility
    var SaveUtility = GraphSaveUtility.GetInstance(_graphView);
    if (save)
    {
        SaveUtility.SaveGraph(_fileName);
    }
    else
    {
        SaveUtility.LoadGraph(_fileName);
    }
}
```

#### 2.6 Save

<font color=#4db8ff>Edge</font>的信息可以存储为<font color=#bc8df9>float List</font>

<font color=#4db8ff>Node</font>信息可以投射为<font color=#bc8df9>Dialogue Node</font>

```c#
using UnityEditor.Experimental.GraphView;
public class GraphSaveUtility : MonoBehaviour
{
//Graph view 中的所有View 和 Node
//TODO:Edge to List 
// private List<Edge> Edges => _targetGraphView.edges.ToList();
private List<Edge> Edges => _targetGraphView.edges.ToList().Cast<Edge>().ToList();
private List<DialogueNode> Nodes => _targetGraphView.nodes.ToList().Cast<DialogueNode>().ToList();
...
}
```

<font color=#4db8ff>UQueryState .ToList</font>：将所有满足选择规则的元素添加到列表中。

<font color=#4db8ff>Link：</font>https://docs.unity3d.com/ScriptReference/UIElements.UQueryState_1.ToList.html

<font color=#4db8ff>Enumerable.Cast<TResult>(IEnumerable) Method：</font>[将IEnumerable](https://learn.microsoft.com/en-us/dotnet/api/system.collections.ienumerable?view=net-7.0)的元素转换为指定类型。

<font color=#4db8ff>Link：</font>https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.cast?view=net-7.0



随后将内容都序列化为数据，存储到新的<font color=#4db8ff>Dialogue Container</font>中

获取连接的部分，以便了解每个阶段应该有一个输出<font color=#bc8df9>Port</font>，只保存输出<font color=#bc8df9>Output Choice Ports</font>，这意味着只要将<font color=#bc8df9>Output Choice Ports</font> 连接到<font color=#bc8df9>Input Choice Ports</font>我们可以将其视为有效连接的<font color=#bc8df9>Port</font>

```c#
public void SaveGraph(string fileName)
{
  if (!Edges.Any()) return;  //no edges( no connections ) then return

  //创建一个新的对话容器
  var dialogueContainer = ScriptableObject.CreateInstance<DialogueContainer>();
  //过滤一个序列
  var connectedPorts = Edges.Where(x => x.input.node != null).ToArray();
}
```

<font color=#4db8ff>Enumerable.Where Method：</font>to filter a sequence

<font color=#4db8ff>Link：</font>https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.where?view=net-7.0

随后利用一个循环，将连接的<font color=#bc8df9>Input Choice Ports</font>、<font color=#bc8df9>Output Choice Ports</font> 将他们转化为<font color=#66ff66>Dialogue node</font>，随后将相关数据保存到<font color=#bc8df9>ScriptableObject 容器里</font>

这里先将<font color=#FFCE70>Port Link</font>信息保存，需要保存两个<font color=#4db8ff>Node</font>，以及输出<font color=#66ff66>Port Name</font>

```c#
//过滤一个序列
  var connectedPorts = Edges.Where(x => x.input.node != null).ToArray();

  //Edge Link Data
  for (int i = 0; i < connectedPorts.Length; i++)
  {
     var outputNode = connectedPorts[i].output.node as DialogueNode;
     var inputNode = connectedPorts[i].input.node as DialogueNode;

     //保存标识符
     dialogueContainer.Nodelinks.Add(new NodeLinkData
     {
        BaseNodeGuid = outputNode.GUID,
        PortName = connectedPorts[i].output.portName,
        TargetNodeGuid = inputNode.GUID
     });
  }
```

<font color=#FFCE70>Node</font>同理

```c#
//Node Data Nodes Data
  foreach (var dialogueNode in Nodes.Where(node=>!node.Entrybool))
  {
     //传递到容器中
     dialogueContainer.DialogueNodeData.Add(new DialogueNodeData
     {
        Guid = dialogueNode.GUID,
        DialogueText = dialogueNode.DialogueText,
        Position = dialogueNode.GetPosition().position
     });
  }
```

随后将<font color=#bc8df9>ScriptableObject</font>保存到资源里，利用<font color=#4db8ff>AssetDatabase</font>

```c#
if (!AssetDatabase.IsValidFolder("Assets/Resources"))
     AssetDatabase.CreateFolder("Assets", "Resources");

AssetDatabase.CreateAsset(dialogueContainer,$"Assets/Resources/{fileName}.asset");
AssetDatabase.SaveAssets();      
```

<font color=#bc8df9>AssetDatabase：</font>https://docs.unity3d.com/ScriptReference/30_search.html?q=AssetDatabase

![image-20230927165532116](assets/image-20230927165532116.png)

#### 2.7 Load

先缓存容器，随后从资源文件夹中加载文件，反序列化文件，将数据读取出来

```c#
public void LoadGraph(string fileName)
{
  _ContainerCache = Resources.Load<DialogueContainer>(fileName);

  if (_ContainerCache == null)
  {
     EditorUtility.DisplayDialog("File Not Found", "Target Dialogue Graph File does not exists!", "OK");
     return;
  }
}
```

每次<font color=#66ff66>Load</font>数据，我们需要先清空<font color=#bc8df9>Graph View</font>，随后创建<font color=#FFCE70>Node</font>，再将它们<font color=#4db8ff>LInk</font>

所以我们创建三个函数方法

```c#
//清除、生成、连接 
ClearGraph();
CreateNodes();
ConnectNodes();
```

当我们打开<font color=#bc8df9>Graph View</font>时，我们会立刻获得<font color=#66ff66>Entry Point</font>，所以<font color=#66ff66>Container</font>中的第一个<font color=#4db8ff>Node</font>的连接始终是<font color=#66ff66>Entry Point</font>。因此可以沿着<font color=#66ff66>Entry Point</font>分配

因为我们链接的第一件事就是<font color=#66ff66>Entry Point</font>，我们可以得到<font color=#4db8ff>Node</font>建立的每个输出链接。

我们只保存<font color=#bc8df9>Output Port</font>，这样我们就可以验证，<font color=#bc8df9>Port</font>是否连接到另一个<font color=#FFCE70>Node Output</font>

检查是否存在有效连接，然后迭代所有的连接，然后在<font color=#bc8df9>Graph View</font>将他们都删除

```c#
private void ClearGraph()
{
  //set Entry points guid back from the save . Discard existing guid.
  Nodes.Find(x => x.Entrybool).GUID = _ContainerCache.Nodelinks[0].BaseNodeGuid;

  foreach (var node in Nodes)
  {
     if(node.Entrybool) return;

     //Remove edges that connected to this node;
     Edges.Where(x => x.input.node == node).ToList().
        ForEach(edge => _targetGraphView.RemoveElement(edge));

     //then remove the node
     _targetGraphView.RemoveElement(node);
  }
}
```

利用循环再次迭代，遍历缓存在<font color=#bc8df9>SOBJ</font>的数据，利用<font color=#bc8df9>Graph View</font>中创建节点的方法，传递文本内容作为参数去创建<font color=#FFCE70>Node</font>

随后赋予<font color=#66ff66>Guid</font>，再添加到<font color=#bc8df9>Graph View</font>

```c#
private void CreateNodes()
{
  foreach (var nodeData in _ContainerCache.DialogueNodeData)
  {
     var tempNode = _targetGraphView.CreateDialogueNode(nodeData.DialogueText);
     tempNode.GUID = nodeData.Guid;
     _targetGraphView.AddElement(tempNode);
  }
}
```

我们遍历已经缓冲的<font color=#bc8df9>Container</font>，利用<font color=#66ff66>Guid</font>进行判断是否匹配，将他们加入容器中。<font color=#4db8ff>Port</font>的选择会有问题

```c#
//port
var nodePorts = _ContainerCache.Nodelinks.Where(x => x.BaseNodeGuid == nodeData.Guid).ToList();
nodePorts.ForEach(x =>_targetGraphView.AddchoicePort(tempNode,x.PortName));
```

我们可以给<font color=#66ff66>Graph View</font>中的<font color=#4db8ff>AddchoicePort</font>方法中添加判断，增加一个<font color="red">String</font>参数，我们利用这个参数来显示给定的<font color=#FFCE70>Port</font>的名称。但是是要判断它是否为空

```c#
public void AddchoicePort(DialogueNode dialogueNode, string overriddenPortName = "")
{
    //port 
    var GeneratedPort = GeneratePort(dialogueNode, Direction.Output);

    //搜索端口名称指定为选择端口计数
    var outputPortCount = dialogueNode.outputContainer.Query("connector").ToList().Count;
    // GeneratedPort.portName = $"choice {outputPortCount}";

    var choicePortName = string.IsNullOrEmpty(overriddenPortName)
        ?$"choice {outputPortCount + 1}"
        :overriddenPortName;

    GeneratedPort.portName = choicePortName;
    
    dialogueNode.outputContainer.Add(GeneratedPort);
    dialogueNode.RefreshPorts();
    dialogueNode.RefreshExpandedState();
}
```

添加文本框，利用<font color=#66ff66>CallBack</font>函数，当<font color=#bc8df9>Graph View</font>和<font color=#4db8ff>Node</font>修改文本内容的时候，<font color=#66ff66>CallBack</font>函数将会更改<font color=#FFCE70>Port</font>函数

```c#
//添加文本
var textField = new TextField
{
    name = string.Empty,
    value = choicePortName
};
textField.RegisterValueChangedCallback(evt => GeneratedPort.portName = evt.newValue);
//空隙
GeneratedPort.contentContainer.Add(new Label(" "));
GeneratedPort.contentContainer.Add(textField);

GeneratedPort.portName = choicePortName;
```

添加一个<font color=#4db8ff>Button</font>从<font color=#FFCE70>Editor</font>中删除<font color=#4db8ff>Port</font>的按钮

```c#
var deleteButton = new Button(() => RemovePort(dialogueNode, GeneratedPort));
GeneratedPort.Add(deleteButton);
```

