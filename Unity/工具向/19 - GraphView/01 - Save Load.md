## 保存和加载使用 GraphView 创建的数据

1.   准备要保存的数据（ScriptableObject）
2.   编辑反映在 ScriptableObject 中

3.   准备节点序列化器和反序列化器
4.   在 Project 窗口中选择 ScriptableObject 打开 EditorWindow

5.   打开时的节点
6.   以便保存连接节点的边
7.   这样就生成了连接节点的边

### 一、Edits are reflected in  ScriptableObject

要保存的数据如下所示，创建保存脚本<font color=#bc8df9>ScriptGraphAsset</font>

```c++
using System.Collections.Generic;
using UnityEngine;

[CreateAssetMenu(fileName = "scriptgraph.asset", menuName ="ScriptGraph Asset")]
public class ScriptGraphAsset : ScriptableObject
{
    public List<ScriptNodeData> list = new List<ScriptNodeData>();
}
```

创建数据脚本<font color=#bc8df9>ScriptNodeData</font>

```c++
using System;
using TreeEditor;
using UnityEngine;

[Serializable]
public class ScriptNodeData
{
    public int id;
    public TreeEditorHelper.NodeType type;
    public Rect rect;
    public int[] outIds;
    public byte[] serialData;
}
```

### 二、Save

如果您在创建节点时将数据添加到可编写脚本的对象中，您可以暂时将其保存为陷阱。



由于某些原因，这并不能保存正确的位置

当我尝试在<font color=#66ff66>_scriptGtaphView.AddElement()</font>的同一帧中获取节点的位置时
返回的是<font color=#bc8df9> Rect(0,0,float.Nan,float.Nan)</font>

作为对策，请调整包管理器中的 <font color="red">EditorCoroutine </font>或类似程序，以便在创建节点后的下一帧保存节点。

### 三、最后

这只是一个非常粗略的描述，但却是现在使用 graphView 所需的最基本信息。

如果你真的想用这个系统制作一款新颖的游戏，你需要对它进行扩展。
我们还提供了一个扩展示例（BranchNode），因此我认为任何程序员都可以对其进行扩展。

故事、图形、声音和各种扩展都可以处理，最终就能完成一款新颖的游戏。

我相信这样的未来！