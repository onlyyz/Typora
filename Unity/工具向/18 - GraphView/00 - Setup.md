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

