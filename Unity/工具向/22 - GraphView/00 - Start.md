<font color=#4db8ff>Link：</font>https://qiita.com/mu5dvlp/items/4b2286584b08d582474c

<font color=#4db8ff>关于搜索树反射 Link ：</font>https://zhuanlan.zhihu.com/p/650274861

### 一、ポイント

#### 1.1 数据

您可以使用 Node 类中 "<font color=#66ff66>Rect worldBound</font> "和 "<font color=#FFCE70>Rect localBound</font> "周围的坐标。不过，从普通编辑器扩展中存储引用数据可能有点麻烦...

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252Fb41b955f-4a80-457b-ba20-5edb6bdbbcac.pngixlib=rb-4.0.png" alt="node1.png" style="zoom:50%;" />

我们想但如果一个节点还有三四个<font color=#FFCE70>Port </font>端口...如果有两个以上的输入和两个以上的输出，就很难知道该把它们连接到哪里。所以我们给端口分配了 <font color=#bc8df9>uids</font>。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252Fc544cd74-c512-aa0b-d7e4-6c60ca7eef95.pngixlib=rb-4.0.png" alt="新ノードエディタ.png" style="zoom:50%;" />

此外，如果以面向节点的方式考虑数据，即 "节点 A 到 B、C"，"节点 B 从 A 到 C、D"，"换句话说，以列表的方式管理......"。

这样一来，引用就很复杂，恢复也很困难。还可能出现重复。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252F48e7a8d7-88dd-24cc-90a9-567aa0ee4134.pngixlib=rb-4.0.png" alt="node2.png" style="zoom:50%;" />

因此，我们主要从边（连接节点的字符串）的角度来思考问题。观察边时，在任何时间点上总是存在 "<font color="red">起点-终点</font> "的一一对应关系。这样可以避免列表和重复引用。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252F4a4257f3-8ff8-7696-caf8-b85a43355b7e.pngixlib=rb-4.0.png" alt="node3.png" style="zoom:50%;" />

### 二、実装

实施编辑器扩展。本项目参考了以下文章。

-GraphView完全理解（2019年底版）。

<font color=#4db8ff>Link：</font>https://qiita.com/ma_sh/items/7627a6151e849f5a0ede

如果你现在手头没有电脑，千万别看。这篇文章太精彩了，几乎是百分之百的 GraphView 现场版！

<font color=#4db8ff>Link：</font>https://qiita.com/dwl/items/dd14f5f2a187084d317b

如果你有能力，请阅读原文，因为它比本文更详细地介绍了实现方法。尤其是，如果这是你第一次尝试节点编辑器，请先尝试实现其中一个或另一个。

首先，准备好以下类。

#### 2.1 GraphWindow.cs

```C#
using System.Linq;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor;
using UnityEditor.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class GraphWindow : EditorWindow
    {
        MyGraphView graphView;
        ObjectField objectField;
        public ScenarioData scenarioData { get { return (ScenarioData)objectField.value; } }
        //ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー 
        [MenuItem("MU5Editor/Node Editor")]
        public static void Open()
        {
            GetWindow<GraphWindow>("Node Editor");
        }
        //ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        void OnEnable()
        {
            Toolbar toolbar = new Toolbar();
            toolbar.style.flexDirection = FlexDirection.Row;
            objectField = new ObjectField();
            objectField.objectType = typeof(ScenarioData);
            Button loadBtn = new Button(LoadData) { text = "ロード" };
            Button saveBtn = new Button(SaveData) { text = "保存" };
            toolbar.Add(objectField);
            toolbar.Add(loadBtn);
            toolbar.Add(saveBtn);
            rootVisualElement.Add(toolbar);

            graphView = new MyGraphView(this)
            {
                style = { flexGrow = 1 }
            };
            rootVisualElement.Add(graphView);
        }

        void LoadData()
        {
            if (scenarioData == null) return;

            graphView.DeleteAllElements();

            foreach (var nodeData in scenarioData.nodeData_list)
            {
                graphView.LoadNodeData(nodeData);
            }
            foreach (var edgeData in scenarioData.edgeData_list)
            {
                graphView.LoadEdgeData(edgeData);
            }

            Debug.Log($"ロード完了");
        }

        void SaveData()
        {
            if (scenarioData == null) return;

            scenarioData.nodeData_list.Clear();
            scenarioData.edgeData_list.Clear();

            foreach (var graphElement in graphView.graphElements)
            {
                if (graphElement is MU5Node) SaveData_Node(graphElement);
                else if (graphElement is Edge) SaveData_Edge(graphElement);
                else Debug.LogWarning($"Find a non-surported graphElement type: {graphElement.GetType()}");
            }

            EditorUtility.SetDirty(objectField.value);
            AssetDatabase.SaveAssets();

            Debug.Log($"保存完了");
        }

        void SaveData_Node(GraphElement _graphElement)
        {
            MU5Node node = _graphElement as MU5Node;
            NodeData nodeData = new NodeData()
            {
                uid = node.uid,
                nodeType_str = node.GetType().ToString(),
                localBound = node.localBound
            };
            scenarioData.nodeData_list.Add(nodeData);
        }

        void SaveData_Edge(GraphElement _graphElement)
        {
            Edge edge = _graphElement as Edge;

            Port inputPort = edge.input;
            Port outputPort = edge.output;
            MU5Node inputNode = edge.input.node as MU5Node;
            MU5Node outputNode = edge.output.node as MU5Node;
            string uid_inputPort_target = inputNode.port_dict.FirstOrDefault(x => x.Value.Equals(inputPort)).Key;
            string uid_outputPort_target = outputNode.port_dict.FirstOrDefault(x => x.Value.Equals(outputPort)).Key;

            EdgeData edgeData = new EdgeData()
            {
                uid_outputNode = outputNode.uid,
                uid_outputPort = uid_outputPort_target,
                uid_inputNode = inputNode.uid,
                uid_inputPort = uid_inputPort_target
            };
            scenarioData.edgeData_list.Add(edgeData);
        }
    }
}

```

#### 2.2 Toolbar

<font color=#bc8df9>Toolbar：</font>https://docs.unity3d.com/ScriptReference/UIElements.Toolbar.html

描述：工具窗口的工具栏

```c#
MyGraphView graphView;
ObjectField objectField;
public SerializeData scenarioData { get { return (SerializeData)objectField.value; } }
//ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー 
[MenuItem("Tool/22-ScriptGraph")]
public static void Open()
{
GetWindow<GraphWindow>("Node Editor");
}
//ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
void OnEnable()
{
//工具栏
Toolbar toolbar = new Toolbar();
toolbar.style.flexDirection = FlexDirection.Row;

//文件路径
objectField = new ObjectField();
objectField.objectType = typeof(SerializeData);
Button loadBtn = new Button(LoadData) { text = "加載" };
Button saveBtn = new Button(SaveData) { text = "保存" };


toolbar.Add(objectField);
toolbar.Add(loadBtn);
toolbar.Add(saveBtn);
rootVisualElement.Add(toolbar);


//Graph View
graphView = new MyGraphView(this)
{
    style = { flexGrow = 1 }
};
rootVisualElement.Add(graphView);
}
```

<font color=#bc8df9>ObjectField：</font>https://docs.unity3d.com/ScriptReference/Search.ObjectField.html

#### 2.3 LoadData

加载数据，首先需要清除<font color=#bc8df9>Graph View</font>中的所有的元素，随后遍历文件中所有的<font color=#4db8ff>Node 和 Edge</font>，将其逐一添加到<font color=#bc8df9>Graph View</font>

```c#
void LoadData()
{
    if (scenarioData == null) return;
    //清除所有元素
    graphView.DeleteAllElements();

    //遍历Node 和 Edge
    foreach (var nodeData in scenarioData.nodeData_list)
    {
        graphView.LoadNodeData(nodeData);
    }
    foreach (var edgeData in scenarioData.edgeData_list)
    {
        graphView.LoadEdgeData(edgeData);
    }

    Debug.Log($"加载好了");
}
```

#### 2.4 SaveData

保存数据，则需要先将<font color=#FFCE70>SOBJ</font>数据清空，随后逐渐遍历<font color=#bc8df9>Graph View</font>中的所有的元素，随后遍历文件中所有的<font color=#4db8ff>Node 和 Edge</font>，将其逐一添加到<font color=#FFCE70>SOBJ</font>文件中

```c#
void SaveData()
{
    if (scenarioData == null) return;

    scenarioData.nodeData_list.Clear();
    scenarioData.edgeData_list.Clear();



    //遍历graph view 中的所有元素 包括 Node 和 edge
    foreach (var graphElement in graphView.graphElements)
    {
        if (graphElement is MU5Node) 
            SaveData_Node(graphElement);
        else if (graphElement is Edge) 
            SaveData_Edge(graphElement);
        else
            Debug.LogWarning($"Find a non-surported graphElement type: {graphElement.GetType()}");
    }

    EditorUtility.SetDirty(objectField.value);
    AssetDatabase.SaveAssets();

    Debug.Log($"保存完了");
}
```

其中存储<font color=#4db8ff>Node</font>和<font color=#4db8ff>Edge</font>的方法大差不差

```c#
void SaveData_Node(GraphElement _graphElement)
{
    MU5Node node = _graphElement as MU5Node;


    NodeData nodeData = new NodeData()
    {
        uid = node.uid,
        nodeType_str = node.GetType().ToString(),
        localBound = node.localBound
    };

    //存入序列化文件
    scenarioData.nodeData_list.Add(nodeData);
}

void SaveData_Edge(GraphElement _graphElement)
{
    Edge edge = _graphElement as Edge;


    Port inputPort = edge.input;
    Port outputPort = edge.output;
    MU5Node inputNode = edge.input.node as MU5Node;
    MU5Node outputNode = edge.output.node as MU5Node;

    string uid_inputPort_target = inputNode.port_dict.FirstOrDefault(x => x.Value.Equals(inputPort)).Key;
    string uid_outputPort_target = outputNode.port_dict.FirstOrDefault(x => x.Value.Equals(outputPort)).Key;

    EdgeData edgeData = new EdgeData()
    {
        uid_outputNode = outputNode.uid,
        uid_outputPort = uid_outputPort_target,
        uid_inputNode = inputNode.uid,
        uid_inputPort = uid_inputPort_target
    };


    //存入序列化文件
    scenarioData.edgeData_list.Add(edgeData);
}
```

<font color=#4db8ff>GetType Link：</font>https://learn.microsoft.com/en-us/dotnet/api/system.object.gettype?view=net-7.0

<font color=#bc8df9>字典 Dictionary：</font>https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2?view=net-7.0

<font color=#bc8df9>FirstOrDefault Method：</font>https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.firstordefault?view=net-7.0

<font color=#4db8ff>描述：</font>Returns the first element of a sequence, or a default value if no element is found.

利用字典序列化，其中利用<font color=#66ff66>String</font>去获取<font color=#bc8df9>Port</font>



#### 2.5 MyGraphView.cs

```C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class MyGraphView : GraphView
    {
        public EntryNode entryNode;
        public ExitNode exitNode;
        public GraphWindow graphWindow;

        //ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public MyGraphView(GraphWindow graphWindow) : base()
        {
            SetupZoom(ContentZoomer.DefaultMinScale, ContentZoomer.DefaultMaxScale);

            Insert(0, new GridBackground());

            this.AddManipulator(new SelectionDragger());
            this.AddManipulator(new ContentDragger());
            this.AddManipulator(new RectangleSelector());

            SearchWindowProvider searchWindowProvider = ScriptableObject.CreateInstance(typeof(SearchWindowProvider)) as SearchWindowProvider;
            searchWindowProvider.Initialize(this, graphWindow);

            nodeCreationRequest += context =>
            {
                SearchWindow.Open(new SearchWindowContext(context.screenMousePosition), searchWindowProvider);
            };

            CreateBasicNodes();
        }

        //ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public void CreateBasicNodes()
        {
            entryNode = new EntryNode();
            entryNode.SetPosition(new Rect(0, 0, 0, 0));
            AddElement(entryNode);

            exitNode = new ExitNode();
            exitNode.SetPosition(new Rect(580, 0, 0, 0));
            AddElement(exitNode);
        }

        public override List<Port> GetCompatiblePorts(Port startAnchor, NodeAdapter nodeAdapter)
        {
            var compatiblePorts = new List<Port>();
            foreach (var port in ports.ToList())
            {
                if (startAnchor.node == port.node ||
                    startAnchor.direction == port.direction ||
                    startAnchor.portType != port.portType)
                {
                    continue;
                }

                compatiblePorts.Add(port);
            }
            return compatiblePorts;
        }

        public void DeleteAllElements()
        {
            foreach (var element in graphElements)
            {
                RemoveElement(element);
            }
        }

        public void LoadNodeData(NodeData nodeData)
        {
            MU5Node node = (MU5Node)Activator.CreateInstance(nodeData.nodeType);
            node.LoadData(nodeData);
            AddElement(node);
        }

        public void LoadEdgeData(EdgeData edgeData)
        {
            Edge edge = new Edge();
            MU5Node outputNode = GetMU5NodeByUid(edgeData.uid_outputNode);
            MU5Node inputNode = GetMU5NodeByUid(edgeData.uid_inputNode);

            edge.output = outputNode.port_dict[edgeData.uid_outputPort];
            outputNode.port_dict[edgeData.uid_outputPort].Connect(edge);
            edge.input = inputNode.port_dict[edgeData.uid_inputPort];
            inputNode.port_dict[edgeData.uid_inputPort].Connect(edge);

            AddElement(edge);
        }

        public MU5Node GetMU5NodeByUid(string _uid)
        {
            MU5Node node = null;
            foreach (var graphElement in graphElements)
            {
                MU5Node _node = graphElement as MU5Node;
                if (_node == null) continue;
                if (_node.uid != _uid) continue;

                node = _node;
                break;
            }
            return node;
        }
    }
}
```

其中利用搜索树接口去查找文件与节点

```c#
//搜索树结构
SearchWindowProvider searchWindowProvider = ScriptableObject.CreateInstance(typeof(SearchWindowProvider)) as SearchWindowProvider;
searchWindowProvider.Initialize(this, graphWindow);

nodeCreationRequest += context =>
{
SearchWindow.Open(new SearchWindowContext(context.screenMousePosition), searchWindowProvider);
};
```

#### 2.6 Load Node

其中加载数据的方法被写在文件中，作为<font color=#bc8df9>Graph View</font>的函数

```c#
//加载 node 数据
public void LoadNodeData(NodeData nodeData)
{
    //返回符合参数的构造函数所创建的实例
    MU5Node node = (MU5Node)Activator.CreateInstance(nodeData.nodeType);

    //传递数据 到 node 中
    node.LoadData(nodeData);
    AddElement(node);
}
```

首先获得符合<font color=#4db8ff>Node</font>类型的实例，随后传递数据给实例，再将其添加到<font color=#bc8df9>Graph View</font>

<font color=#bc8df9>Activator：</font>https://learn.microsoft.com/en-us/dotnet/api/system.activator?view=net-7.0

<font color=#4db8ff>Activator.CreateInstance Method：</font>https://learn.microsoft.com/en-us/dotnet/api/system.activator.createinstance?view=net-7.0

<font color=#66ff66>describe ：</font> Creates an instance of the specified type using the constructor that best matches the specified parameters.

使用与指定参数最匹配的构造函数创建指定类型的实例。

#### 2.7 Load Edge

```c#
//加载链接数据
public void LoadEdgeData(EdgeData edgeData)
{
    Edge edge = new Edge();

    //UID匹配 Node
    MU5Node outputNode = GetMU5NodeByUid(edgeData.uid_outputNode);
    MU5Node inputNode = GetMU5NodeByUid(edgeData.uid_inputNode);


    //创建链接
    edge.output = outputNode.port_dict[edgeData.uid_outputPort];
    outputNode.port_dict[edgeData.uid_outputPort].Connect(edge);
    edge.input = inputNode.port_dict[edgeData.uid_inputPort];
    inputNode.port_dict[edgeData.uid_inputPort].Connect(edge);


    AddElement(edge);
}


//遍历匹配UID
public MU5Node GetMU5NodeByUid(string _uid)
{
    MU5Node node = null;

    //查找匹配的UID
    foreach (var graphElement in graphElements)
    {
        MU5Node _node = graphElement as MU5Node;
        if (_node == null) continue;
        if (_node.uid != _uid) continue;

        node = _node;
        break;
    }
    return node;
}
```

#### 2.8 SearchWindowProvider.cs

```C#
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class SearchWindowProvider : ScriptableObject, ISearchWindowProvider
    {
        private MyGraphView graphView;
        private GraphWindow graphWindow;

        public void Initialize(MyGraphView graphView, GraphWindow graphWindow)
        {
            this.graphView = graphView;
            this.graphWindow = graphWindow;
        }

        List<SearchTreeEntry> ISearchWindowProvider.CreateSearchTree(SearchWindowContext context)
        {
            var entries = new List<SearchTreeEntry>();
            entries.Add(new SearchTreeGroupEntry(new GUIContent("Create Node")));

            foreach (var assembly in AppDomain.CurrentDomain.GetAssemblies())
            {
                foreach (var type in assembly.GetTypes())
                {
                    if (type.IsClass && !type.IsAbstract && (type.IsSubclassOf(typeof(MU5Node)))
                        && type != typeof(EntryNode) && type != typeof(ExitNode))
                    {
                        entries.Add(new SearchTreeEntry(new GUIContent(type.Name)) { level = 1, userData = type });
                    }
                }
            }

            return entries;
        }

        bool ISearchWindowProvider.OnSelectEntry(SearchTreeEntry searchTreeEntry, SearchWindowContext context)
        {
            var type = searchTreeEntry.userData as System.Type;

            var node = Activator.CreateInstance(type) as MU5Node;
            var worldMousePosition = graphWindow.rootVisualElement.ChangeCoordinatesTo(graphWindow.rootVisualElement.parent, context.screenMousePosition - graphWindow.position.position);
            var localMousePosition = graphView.contentViewContainer.WorldToLocal(worldMousePosition);
            node.SetPosition(new Rect(localMousePosition, new Vector2(100, 100)));
            graphView.AddElement(node);

            return true;
        }
    }
}
```

搜索树结构添加到<font color=#bc8df9>Graph View </font>中需要先获取实例，可以通过初始化函数完成，在创建<font color=#bc8df9>SearchTreeEntry</font>的时候传递过去

```c#
private MyGraphView graphView;
private GraphWindow graphWindow;

public void Initialize(MyGraphView graphView, GraphWindow graphWindow)
{
    this.graphView = graphView;
    this.graphWindow = graphWindow;
}
```

#### 2.9 SearchTreeEntry

<font color=#66ff66>Scripts</font>继承<font color=#bc8df9>ISearchWindowProvider</font>既可以使用<font color=#66ff66>CreateSearchTree</font>去创建搜索树，随后利用<font color=#FFCE70>Domain</font>判断

<font color=#66ff66>CreateSearchTree：</font>用于加入搜索列表容器中

<font color=#66ff66>OnSelectEntry：</font>大部分用于空间转换到<font color=#bc8df9>Graph View</font>

```c#
//Search Tree Entry
//返回搜索窗口中显示的SearchTreeEntry对象列表
List<SearchTreeEntry> ISearchWindowProvider.CreateSearchTree(SearchWindowContext context)
{
    var entries = new List<SearchTreeEntry>();
    entries.Add(new SearchTreeGroupEntry(new GUIContent("Create Node")));

    foreach (var assembly in AppDomain.CurrentDomain.GetAssemblies())
    {
        foreach (var type in assembly.GetTypes())
        {
            if (type.IsClass && !type.IsAbstract && (type.IsSubclassOf(typeof(MU5Node)))
                && type != typeof(EntryNode) && type != typeof(ExitNode))
            {


                entries.Add(new SearchTreeEntry(new GUIContent(type.Name))
                {
                    level = 1, userData = type
                });
            }
        }
    }

    return entries;
}
```

其中利用到了反射的内容

### 三、Reflection

> Type类，用来包含类型的特性。对于程序中的每一个类型，都会有一个包含这个类型的信息的Type类的对象,类型信息包含数据，属性和方法等信息。

根据上述描述我们可以知道，<font color=#bc8df9>Type</font>记录了类的各种信息，那我们接着看下面的foreach循环，首先选择的循环对象是当前程序域<font color="red">AppDomain</font>中所有的程序集<font color="red">Assembly</font>，<font color=#4db8ff>Assembly</font>是什么呢？

#### 3.1 Domain

在前面我们提到<font color=#4db8ff>Assembly</font>类可以获得正在运行的装配件信息，那装配件又是什么呢？似乎这里涉及到的东西一层套一层难以理解，那么我们干脆就从C#程序的构成讲起。

> C#应用程序结构包含 <font color=#FFCE70>应用程序域（AppDomain），程序集（Assembly），模块（Module），类型（Type），成员（EventInfo、FieldInfo、MethodInfo、PropertyInfo</font>） 几个层次
>    他们之间是一种从属关系，也就是说，一个AppDomain能够包括N个Assembly，一个Assembly能够包括N个Module，一个Module能够包括N个Type，一个Type能够包括N个成员。他们都在System.Reflection命名空间下。

我们先从<font color="red">AppDomain</font>中获取一个个<font color=#4db8ff>Assembly</font>，获取到某一个Assembly后我们调用<font color=#bc8df9>GetTypes</font>获取到当前程序集中所有的Type也就是获取到所有的类，此方法的返回值是一个数组

之后我们使用Where查找返回所有符合要求的类，这个属于C# Linq的内容我们之后在讲，<font color="red">总之Where的作用就是构造并返回符合给定要求的可枚举列表</font>，简单来说就是起到了一个筛选的作用

在这里我们给定的要求是，首先要是一个类，其次，这个类不能是抽象类，最后，这个类要是节点基类的子类。这样一来我们就获取到了所有的非抽象类的节点类。最后这个Assembly的节点类获取完后我们就去找下一个<font color=#bc8df9>Assembly</font>的节点类，直到我们找到了所有的节点类。

看，反射就是如此神奇，我们不用手动去找所有的节点类，只需要依靠反射就能够找到项目中所有我们定义的节点。找到所有的节点之后我们就可以进行下一步操作了，即去获取我们在节点声明的属性BTNodeAttribute中的内容了。

### 四、ノード系

吹毛求疵地说，<font color=#FFCE70>Port Class</font>显然不是为继承而设计的。但是，我希望在<font color=#4db8ff>Port</font>发生 "字段名称更改"、"顺序更改 "或 "增加/减少 "时，数据不会被破坏。

因此，我不想创建 MyPort 或类似的东西，而是想在节点上用字典来管理它。换句话说，<font color="red">uid </font>变成了关键字。这次，uid 是根据特定规则分配的，但您也可以根据自己的操作来分配 uid。

#### 4.1 MU5Node.cs

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;


namespace MU5Editor.NodeEditor
{
    public class MU5Node : Node
    {
        public string uid = string.Empty;
        public Dictionary<string, Port> port_dict;

        public Label uidLabel = new Label();

        //<Methods>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public void Init_UID()
        {
            if (uid != string.Empty) return;

            uid = MU5Utility.GenerateUID();
            viewDataKey = uid;
        }

        public void LoadData(NodeData nodeData)
        {
            uid = nodeData.uid;
            SetPosition(nodeData.localBound);
        }
    }
}
```

#### 4.2 EntryNode.cs

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class EntryNode : MU5Node
    {
        //<Variables>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        Port outputPort;
        void Init_Port()
        {
            outputPort = Port.Create<Edge>(Orientation.Horizontal, Direction.Output, Port.Capacity.Multi, typeof(Port));
            outputPort.portName = "output";
            outputContainer.Add(outputPort);

            port_dict = new Dictionary<string, Port>(){
               { "500100",outputPort}
            };
        }

        //<Constructor>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public EntryNode() : base()
        {
            title = "Entry";
            titleContainer.style.backgroundColor = new StyleColor() { value = new Color(0, 0.6f, 0) };
            capabilities -= Capabilities.Deletable;
            Init_UID();
            Init_Port();
        }
    }
}
```

#### 4.3 ExitNode.cs

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class ExitNode : MU5Node
    {
        //<Variables>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        Port inputPort;
        void Init_Port()
        {
            inputPort = Port.Create<Edge>(Orientation.Horizontal, Direction.Input, Port.Capacity.Multi, typeof(Port));
            inputPort.portName = "input";
            inputContainer.Add(inputPort);

            port_dict = new Dictionary<string, Port>(){
                {"000100",inputPort}
            };
        }

        //<Constructor>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public ExitNode() : base()
        {
            title = "Exit";
            titleContainer.style.backgroundColor = new StyleColor() { value = new Color(0.6f, 0, 0) };
            capabilities -= Capabilities.Deletable;
            Init_UID();
            Init_Port();
        }
    }
}
```

#### 4.4 SampleNode.cs

```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEditor.UIElements;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    public class SampleNode : MU5Node
    {
        //<Variables>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        Port inputPort1;
        Port inputPort2;
        Port outputPort1;
        Port outputPort2;
        Port outputPort3;

        void Init_Port()
        {
            inputPort1 = Port.Create<Edge>(Orientation.Horizontal, Direction.Input, Port.Capacity.Multi, typeof(Port));
            inputPort1.portName = "in1";
            inputContainer.Add(inputPort1);

            inputPort2 = Port.Create<Edge>(Orientation.Horizontal, Direction.Input, Port.Capacity.Multi, typeof(Port));
            inputPort2.portName = "in2";
            inputContainer.Add(inputPort2);

            outputPort1 = Port.Create<Edge>(Orientation.Horizontal, Direction.Output, Port.Capacity.Multi, typeof(Port));
            outputPort1.portName = "out1";
            outputContainer.Add(outputPort1);

            outputPort2 = Port.Create<Edge>(Orientation.Horizontal, Direction.Output, Port.Capacity.Multi, typeof(Port));
            outputPort2.portName = "out2";
            outputContainer.Add(outputPort2);

            outputPort3 = Port.Create<Edge>(Orientation.Horizontal, Direction.Output, Port.Capacity.Multi, typeof(Port));
            outputPort3.portName = "out3";
            outputContainer.Add(outputPort3);

            port_dict = new Dictionary<string, Port>(){
                {"000100",inputPort1},
                {"000200",inputPort2},
                {"500100",outputPort1},
                {"500200",outputPort2},
                {"500300",outputPort3}
            };
        }

        //<Constructor>ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
        public SampleNode() : base()
        {
            title = "サンプル";
            Init_UID();
            Init_Port();
        }
    }
}
```

### 五、データ系

类名为 <font color=#66ff66>ScenarioData</font>，因为我想为一款小说游戏开发它。我认为你可以使用 <font color="red">NodeGraphData </font>或类似的东西来做一般用途。(在这种情况下，您也应该更改在其他类中使用的 ScenarioData 名称）。

#### 5.1 ScenarioData.cs

```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor.Experimental.GraphView;

namespace MU5Editor.NodeEditor
{
    [CreateAssetMenu(menuName = "MU5/ScenarioData")]
    public class ScenarioData : ScriptableObject
    {
        public List<NodeData> nodeData_list;
        public List<EdgeData> edgeData_list;
    }

    //ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー
    [Serializable]
    public class NodeData
    {
        public string uid;
        public string nodeType_str;
        public Type nodeType { get { return Type.GetType(nodeType_str); } }
        public Rect localBound;
    }

    //ーーーーーーーーーーーーーーーーーーーーー
    [Serializable]
    public class EdgeData
    {
        public string uid_outputNode;
        public string uid_outputPort;
        public string uid_inputNode;
        public string uid_inputPort;
    }
}
```

### 六、その他

如果只是这一次，我想我就没有必要用不同的方式来制作它，但我还是要重复一下解释：这是一个为小说游戏开发的节点编辑器，所以我想要一个类似实用工具的类...所以我把 <font color="red">uid </font>生成放在了这里。

#### 6.1 MU5Utility

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace MU5Editor
{
    public class MU5Utility
    {
        public static string GenerateUID()
        {
            char[] characters = { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                                  'a','b','c','d','e','f','g','h','i','j','k','l','m',
                                  'n','o','p','q','r','s','t','u','v','w','x','y','z' };
            string uid = string.Empty;
            int digit = 16;
            for (int i = 0; i < digit; i++)
            {
                int num = Random.Range(0, characters.Length);
                uid += characters[num];
            }

            return uid;
        }
    }
}
```

首先，试试这个方法是否有效。从工具栏中选择 "MU5Editor（编辑器）> Node Editor（节点编辑器）"。

现在，您可以右键单击并按下 "创建节点 > 样本节点 "来获得一个节点。拖动端口上的圆圈，将节点连接在一起。

节点数据是通过工具栏中的 "资产">"创建">"MU5">"场景数据 "创建的。

按您认为合适的方式组合节点后，按下保存按钮。(在测试过程中，NullReference 在这里出现过一次，但无法再现......）。寻求信息......)

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252F75c2203d-11bb-3a44-e58f-c7c4d8c2de67.pngixlib=rb-4.0.png" alt="スクリーンショット 2022-10-07 23.33.54.png" style="zoom:50%;" />

用 X 关闭窗口，然后再次打开。然后再次从 "节点（ScenarioData）"中选择 "数据"，并按下 "加载 "按钮。数据就会恢复。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F2607452%252F4346aa17-d244-aeb8-c89a-a14fa6d0db70.pngixlib=rb-4.0.png" alt="スクリーンショット 2022-10-07 23.34.23.png" style="zoom:50%;" />

### 七、まとめ

怎么样？在这篇文章中，我们介绍了 Node Editor 的实现和还原。像这篇文章一样，我们将介绍从高级内容到针对 Unity 初学者的内容等各种内容。
如果你喜欢，请评价并关注我们。