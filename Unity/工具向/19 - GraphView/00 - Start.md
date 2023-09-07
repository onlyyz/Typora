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

脚本：<font color=#FFCE70>SampleGraphView</font>

```C#
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

节点本身似乎已经创建，但 GraphView 的高度却设置为 0。请设置高度，以便正确显示。