

1. Dialogue
2. Node
3. Graph View  
4. AppDoMain
5. Visual Element
6. Save
7. Editor
8. <font color="red">USS</font>
9. Serialise



Plugins

1、Game Creator2

2、Odin

3、

### 一、Graph View

#### 1.1 NodeBasedDialogueSystem

<font color=#4db8ff>LInk：</font>https://www.youtube.com/channel/UCDK3uzTu962H75UFVWlkrJw

<font color=#bc8df9>NodeBasedDialogueSystem</font>

<font color=#4db8ff>Git Link：https://github.com/merpheus-dev/NodeBasedDialogueSystem</font>





### 二、RMGUI

<font color=#4db8ff>Link：</font>https://www.zhihu.com/question/24462113/answer/40216675

使用<font color="red">Retained Mode GUI </font>(RMGUI)以及按需更新的模式，当UI元素没有发生变化的时候几乎0消耗。

Unity编辑器拓展主流仍然是<font color=#66ff66>Immediate Mode GUI</font> (IMGUI)的形式

<font color=#66ff66>NodeGraphProcessor</font>是基于<font color="red">Unity GraphView</font>的，它享受所有GraphView的特性和基建，性能和轻便性皆为上乘

<font color=#4db8ff>git Link：</font>https://github.com/alelievr/NodeGraphProcessor



<font color=#4db8ff>文档：</font>https://alelievr.github.io/NodeGraphProcessor/api/GraphProcessor.BaseNode.html

#### 2.1 NodeGraphProcessor



### 三、序列化



### 四、Odin 接入

<font color=#4db8ff>Link：</font>https://www.lfzxb.top/nodegraphprocesssor-and-odin/

#### 4.1 Odin NodeGraphProcessor

由于使用了Unity自带的Json方案进行序列化反序列化，所以有非常多类型不支持（Dic，HashSet等），这点几乎是致命的，因为在游戏业务中这些泛型集合是非常常用的。Odin就可以很好的解决这个问题。

需要特别注意的有几点

我们要把一些原本继承自<font color=#4db8ff>ScriptableObject</font>的对象改为继承Odin的<font color=#66ff66>SerializedScriptableObject</font>，否则序列化反序列化支持不完善

弃用<font color=#bc8df9>SerializeField，Serializable，SerializeReference，ISerializationCallbackReceiver</font>等Unity原生序列化相关<font color=#4db8ff>Attribute</font>和接口，他会与<font color="red">Odin</font>的序列化冲突导致数据损坏，丢失等问题

由于原仓库使用了<font color=#FFCE70>SerializeReference</font>序列化<font color=#4db8ff>List<BaseNode></font>引用，这个特性会递归序列化所有引用字段，所以<font color="red">BaseNode</font>中的一些集合字段

<font color=#FFCE70>例如：</font>

```C#
[NonSerialized]
public NodeInputPortContainer	inputPorts;
[NonSerialized]
public NodeOutputPortContainer	outputPorts;
[NonSerialized]
internal Dictionary< string, NodeFieldInformation >	nodeFields = new Dictionary< string, NodeFieldInformation >();
[NonSerialized]
internal Dictionary< Type, CustomPortTypeBehaviorDelegate> customPortTypeBehaviorMap = new Dictionary<Type, CustomPortTypeBehaviorDelegate>();
Stack<PortUpdate> fieldsToUpdate = new Stack<PortUpdate>();
HashSet<PortUpdate> updatedFields = new HashSet<PortUpdate>();
//////////////////////////////////////////////////////////////////////
```

依旧会被作为引用序列化，但如果使用Odin序列化方案，这些内容并不会被序列化并且在进入<font color=#bc8df9>PlayMode</font>后会进行GC置空，但好在这些字段都是<font color="red">实时计算</font>的，所以我们只需要在<font color=#66ff66>BaseNode</font>初始化的时候对这些字段进行初始化即可。

#### 4.2 Commit

<font color=#4db8ff>Link：</font>https://github.com/wqaetly/NodeGraphProcessor/commit/2a68396239d92773e827a63c0c44efb5383cccfe#diff-616c2c0a85fd16b0fdf0904c6a488c7b572a68ca22df7361b647e52f0474c911

#### 4.3步骤

修改<font color="red">BaseGraph</font>继承<font color=#66ff66>SerializedScriptableObject</font>，并去除其中主要字段的<font color=#66ff66>Attribute</font>标记，达到全盘<font color="red">Odin</font>托管的目的

```C#
public void Initialize(BaseGraph graph)
{
    this.graph = graph;
    ExceptionToLog.Call(() => Enable());
    inputPorts = new NodeInputPortContainer(this);
    outputPorts = new NodeOutputPortContainer(this);
    nodeFields = new Dictionary<string, NodeFieldInformation>();
    customPortTypeBehaviorMap = new Dictionary<Type, CustomPortTypeBehaviorDelegate>();
    InitializeInOutDatas();
    InitializePorts();
}
```

去掉为<font color=#66ff66>BaseGraph</font>编写的<font color=#bc8df9>CustomEditor</font>（GraphAssetInspector），使其使用Odin的Inspector面板

```C#
using Sirenix.OdinInspector;
public class BaseGraph : SerializedScriptableObject, ISerializationCallbackReceiver
{}
```

去掉为<font color=#4db8ff>NodeInspectorObject</font>的<font color=#bc8df9>CustomEditor</font>（NodeInspectorObjectEditor）

因为<font color=#4db8ff>NodeGraphProcessor</font>本身使用了一种奇淫方法来绘制Node到<font color=#66ff66>Inspector</font>上：当选中Node时，会查找其中是否有被ShowInInspectorAttribute标记的字段，如果有的话，就将这个节点加入NodeInspectorObject的selectedNodes中，并且将Selection.activeObject设置为NodeInspectorObject达成绘制的目的。

当然如果想要所有Node都可以被Odin绘制在Inspector的话也非常简单



还有一些零碎的地方需要修改，主要是访问权限和序列化相关的修改（哪里报错改哪里，多数是由于空引用），大家要多注意测试从Editor进入PlayMode之后Graph的变化，很有可能序列化方式不对导致数据被GC产生报错。





