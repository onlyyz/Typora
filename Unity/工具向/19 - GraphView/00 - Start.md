<font color=#4db8ff>Link：</font>https://qiita.com/ma_sh/items/7627a6151e849f5a0ede

### GraphViewとは

Unity 目前正在过渡到新的编辑器 <font color=#bc8df9> UI--UIElements</font>（RMGUI）。UIElements 本身已正式发布，2019 版本中删除了实验版。

<font color="red">GraphView</font> 是允许创建节点编辑器的功能，是 <font color=#66ff66>UIElements </font>的一部分，但尽管目前在 ShaderGraph 和 VFXGraph 等官方 Unity 工具中使用，它仍被视为实验性功能，尚未准备好使用。该功能被视为实验功能，尚未准备就绪，无法使用。

不过，作为一名编辑器扩展爱好者，这是我不能不抑制的一项功能，因此，这是一次尝试，目的是在 2019 年年底之前充分了解 GraphView，并在其正式发布时能够立即使用。

因此，到正式发布时，API 和其他规范可能会有很大变化。希望您能将此作为阅读材料。

### 一、GraphViewの要素

GraphViewは大きく4つの要素から成っています。

- GraphView
- Node
- Port
- Edge

还有其他可选元素，例如<font color=#66ff66>Blackboard</font>、由于我们将采用最简单的 GraphView 实现，因此暂时跳过这些元素。有关上述元素的详细解释将在我们实际实现这些元素的后续章节中给出。

#### 1.1 実際にGraphViewを実装する

首先，创建编辑器窗口（<font color=#66ff66>EditorWindow</font>），这是我们熟悉的编辑器扩展。脚本<font color=#FFCE70>SampleGraphEditorWindow</font>

```C#
using UnityEditor;

public class SampleGraphEditorWindow : EditorWindow
{
    [MenuItem("Window/Open SampleGraphView")]
    public static void Open()
    {
        GetWindow<SampleGraphEditorWindow>("SampleGraphView");
    }
}
```

现在从菜单窗口中选择 "Open SampleGraphView"（打开 SampleGraphView）来显示窗口。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fac73be87-720c-d357-56e0-f41735596ea1.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.43.14.png" style="zoom: 33%;" />

#### 1.2 GraphViewを作成

接下来，我们将快速创建一个 <font color=#bc8df9>GraphView</font>。它是上图中节点和边缘的父节点。一旦最小化，我们将创建一个不显示任何内容的 <font color=#bc8df9>GraphView</font>

```C#
//SampleGraphView
using UnityEditor.Experimental.GraphView;
public class SampleGraphView : GraphView
{
}
```

同时，在这里为<font color=#bc8df9>EditorWindow</font>添加一个<font color=#66ff66>GraphView</font>。

```C#
//SampleGraphEditorWindow.cs    
void OnEnable()
{
    rootVisualElement.Add(new SampleGraphView());
}
```

尚未显示任何内容，但一旦显示就可以了。

#### 1.3 Nodeを作成

<font color=#66ff66>GraphView</font> 通常被称为节点编辑器、<font color=#bc8df9>Node</font>，顾名思义，是节点编辑器中最重要的部分。
一旦创建了基本节点，在实际使用时，很可能会创建具有各种<font color="red">继承进程的节点</font>。

我们还将创建一个至少没有实现的 Node。

```C#
//SampleNode.cs
using UnityEditor.Experimental.GraphView;

public class SampleNode : Node
{
}
```

将此添加到 <font color=#bc8df9>GraphView </font>中。只需在构造函数中添加一次即可。

```C#
//SampleGraphView.cs
public SampleGraphView() : base()
{
    AddElement(new SampleNode());
}
```

那么，让我们检查一下编辑器窗口。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fac73be87-720c-d357-56e0-f41735596ea1.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.43.14.png" style="zoom:33%;" />

然后就什么都不显示了...在这种情况下<font color=#bc8df9>UIElements </font>调试器就派上用场了。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fba8c1dca-0edf-a878-b469-7c0f86e01b7c.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.48.06.png" style="zoom:33%;" />

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Feaa54d5c-52b9-97d5-62ab-215f797c2ef3.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.46.50.png" style="zoom:33%;" />

どうやらNode自体は作成されているのですが、GraphViewのHeightが0になっているようです。
きちんと表示されるようにHeightを設定してあげます。

节点本身似乎已经创建，但 <font color=#bc8df9>GraphView </font>的高度却设置为 0。请设置高度，以便正确显示。

```C#
//SampleGraphEditorWindow.CS
void OnEnable()
{
    var graphView = new SampleGraphView()
    {
        style = { flexGrow = 1 }
    };
    rootVisualElement.Add(graphView);
}
```

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fcf1b758c-052d-9a58-5acf-172b93d024af.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.49.31.png" style="zoom:33%;" />

现在，<font color=#bc8df9>GraphView </font>可以拉伸和收缩，以适应编辑器窗口的高度，并完整显示，隐藏的节点现在可见。

#### 1.4 NodeにPortをつける

<font color=#66ff66>Node</font>与其他<font color=#66ff66>Node</font>相连。在本例中，边缘是从<font color=#bc8df9>OutputPort</font>连接到<font color=#bc8df9>InputPort</font>的。
首先，为节点附加输入端口和输出端口。此外，我们还将简要准备节点的外观。

```C#
//SampleNode.cs
public SampleNode()
{
    title = "Sample";

    var inputPort = Port.Create<Edge>(Orientation.Horizontal, Direction.Input, Port.Capacity.Single, typeof(Port));
    inputContainer.Add(inputPort);

    var outputPort = Port.Create<Edge>(Orientation.Horizontal, Direction.Output, Port.Capacity.Single, typeof(Port));
    outputContainer.Add(outputPort);
}
```

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fe7cbe842-c934-c944-337d-06eab11ad140.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.50.02.png" style="zoom:33%;" />

目前，<font color=#66ff66>Port </font>和 <font color=#66ff66>Edge </font>使用的是基础材料，但它突然变成了类似节点的东西。

#### 1.5 Nodeを動かせるようにする

在这种情况下，我觉得节点不动有点奇怪，所以在这里让它动一下。在 <font color=#bc8df9>GraphView </font>中，您可以为节点添加<font color="red">Manipulator</font>操纵器，使其移动。操纵器有多种类型，但现在我们只讨论可以拖动节点移动的操纵器。

<font color=#66ff66>UnityEngine.UIElements </font>可以利用<font color=#FFCE70>using</font>来使用

```C#
//SampleGraphView.cs
using UnityEngine.UIElements;
```

SelectionDraggerをAddManipulatorすると，当 <font color=#66ff66>SelectionDragger </font>为 <font color="red">AddManipulator </font>参数时。

```C#
public SampleGraphView() : base()
{
    AddElement(new SampleNode());
    this.AddManipulator(new SelectionDragger());
}
```

拖动即可移动。

#### 1.6 Nodeを複数作れるようにする

目前，<font color=#bc8df9>GraphView </font>构造函数应该会创建一个、这可以通过右键菜单添加。

```C#
//SampleGraphView.cs
public SampleGraphView() : base()
{
    this.AddManipulator(new SelectionDragger());

    nodeCreationRequest += context =>
    {
        AddElement(new SampleNode());
    };
}
```

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252Fda804162-3a53-c531-80dd-2cb4628125dc.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.50.52.png" style="zoom:33%;" />

现在，您可以创建多个 <font color=#bc8df9>SampleNodes</font>。现在，您想连接这些节点，但它们似乎无法正常连接。

#### 1.7 Node同士を繋げる

<font color=#FFCE70>Node</font>不能连接到任何<font color=#FFCE70>Node</font>或任何端口。它需要从<font color=#66ff66>InputPort</font>连接到正确的<font color=#66ff66>OutputPort</font>。因此，覆盖 <font color=#bc8df9> GetCompatiblePorts </font>可返回正确的端口。

```C#
//SampleGraphView.cs
public override List<Port> GetCompatiblePorts(Port startAnchor, NodeAdapter nodeAdapter)
{
    return ports.ToList();
}
```

在这种情况下，我暂时尝试连接所有<font color=#66ff66>Port</font>、事实上，有必要从 <font color="red">startAnchor </font>中的端口做出正确的决定。

<img src="assets/https%253A%252F%252Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%252F0%252F46278%252F5d774e9d-ba21-4c89-450c-a3267c62d569.pngixlib=rb-4.0.png" alt="スクリーンショット 2019-11-23 23.50.58.png" style="zoom:33%;" />

要看清<font color="red">Edge</font>颜色和背景颜色非常困难，但现在节点是相互连接的。至此，我想我已经将 <font color=#66ff66>GraphView </font>用户界面本身的设计达到了一个合理的水平。我想过就此结束，但我觉得要真正创建一个工具，元素还是不够多、我将继续学习创建工具的章节。

### 二、GraphViewを用いてツールを作成する