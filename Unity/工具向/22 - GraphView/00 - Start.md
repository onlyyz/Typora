<font color=#4db8ff>Link：</font>https://qiita.com/mu5dvlp/items/4b2286584b08d582474c

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