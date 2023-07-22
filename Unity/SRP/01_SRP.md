# SRP

创建一个渲染管线资产和实例。
渲染一个相机的视图。
进行剔除、过滤和排序。
分离不透明、透明和无效的通道。
使用一个以上的摄像机。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/tutorial-image.jpg)

<center>Rendering with a custom render pipeline.</center>

### 1、A new Render Pipeline

为了渲染任何东西，Unity必须确定哪些形状必须被绘制，在哪里，什么时候，以及用什么设置。这可能会变得非常复杂，取决于涉及到多少种效果。灯光、阴影、透明度、图像效果、体积效果等等都必须按照正确的顺序进行处理，才能得到最终的图像。这就是渲染管线的作用。

在过去，Unity只支持一些内置的方法来渲染东西。Unity 2018引入了可编写脚本的渲染管道--简称RP，这使得我们可以做任何我们想做的事情，同时仍然能够依靠Unity来完成基本步骤，如剔除。Unity 2018还增加了两个用这种新方法制作的实验性RP：轻量级RP和高清晰度RP。在Unity 2019年，轻量级RP不再是实验性的，在Unity 2019.3中被重新命名为通用RP。

通用RP注定要取代目前的传统RP，成为默认的。我们的想法是，它是一个最适合的RP，也将是相当容易定制的。与其定制该RP，这个系列将从头开始创建一个完整的RP。

本教程通过一个最小的RP奠定了基础，该RP使用正向渲染来绘制无光形状。一旦成功，我们就可以在后面的教程中扩展我们的管道，增加照明、阴影、不同的渲染方法和更多的高级功能。

#### 1.1、Project Setup

在Unity 2019.2.6或更高版本中创建一个新的3D项目。我们将创建自己的管道，所以不要选择RP项目模板中的一个。项目打开后，你可以到包管理器中删除所有你不需要的包。我们在本教程中只使用Unity UI包来试验绘制用户界面，所以你可以保留那个包。

我们将完全在线性色彩空间中工作，但Unity 2019.2仍然使用伽马空间作为默认。通过 "编辑/项目设置 "进入播放器设置，然后进入播放器，然后将 "其他设置 "部分下的色彩空间切换为线性。

#### 1.2、Pipeline Asset

C#需要使用[RenderPipelineAsset](https://docs.unity3d.com/ScriptReference/Rendering.RenderPipelineAsset.html) form [ScriptableObject](https://docs.unity3d.com/ScriptReference/ScriptableObject.html)

目前，Unity使用默认的渲染管线。要用自定义的渲染管线来替代它，我们首先要为它创建一个资产类型。我们将使用与<font color="green">Unity为Universal RP</font>使用的大致相同的文件夹结构。创建一个自定义RP资产文件夹和一个Runtime子文件夹。在那里为CustomRenderPipelineAsset类型放一个新的C#脚本。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/a-new-render-pipeline/folder-structure.png" alt="img" style="zoom:50%;" />

<center>Folder structure.</center>

该资产类型必须从<font color="green">UnityEngine.Rendering</font>命名空间中扩展出<font color="red">RenderPipelineAsset</font>。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderPipelineAsset : RenderPipelineAsset {}
```

Unity 提供一种获取<font color="green">管道对象实例</font>的方法。这是通过覆盖抽象<font color="red">CreatePipeline</font>方法来完成的，它应该返回一个<font color="RoyalBlue">RenderPipeline</font>实例

该<font color="green">CreatePipeline</font>方法是用 <font color="red">protected</font>访问修饰符定义的，这意味着只有定义该方法的类——即<font color="DarkOrchid ">RenderPipelineAsset</font>——以及扩展它的类才能访问它

放在一个*Rendering*子菜单

```c#
[CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline")]
public class CustomRenderPipelineAsset : RenderPipelineAsset {
    protected override RenderPipeline CreatePipeline () {
        return null;
    }
}
```



#### [1.3、渲染管线实例](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#1.3)

<font color="green">RenderPipeline</font>定义了一个受保护的抽象<font color="red">Render</font>方法，我们必须<font color="RoyalBlue">重写它才能创建具体的管道</font>。它有两个参数：一个<font color="green">ScriptableRenderContext</font>和一个<font color="red">Camera</font>数组。现在将该方法留空。

```c#
public class CustomRenderPipeline : RenderPipeline
{
    protected override void Render (
        ScriptableRenderContext context, Camera[] cameras
    ) {}
   
}
```

使**<font color="green">CustomRenderPipelineAsset</font>**.<font color="red">CreatePipeline</font>返回一个新的实例<font color="red">**CustomRenderPipeline**</font>。这将为我们提供一个有效且功能齐全的管道，尽管它还没有渲染任何东西。

```c#
[CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline")]
public class CustomRenderPipelineAsset :  RenderPipelineAsset 
{
    protected override RenderPipeline CreatePipeline () {
        return new CustomRenderPipeline();
    }
}
```

### 2、Rendering

每一帧Unity都会在RP实例上调用Render。它传递一个上下文结构，提供一个与本地引擎的连接，我们可以用它来进行渲染。它还会传递一个摄像机数组，因为场景中可能有多个活跃的摄像机。RP有责任按照所提供的顺序渲染所有这些摄像机。

#### [2.1、相机渲染](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.1)

每个摄像机都独立渲染。因此，**CustomRenderPipeline**我们不会渲染所有相机，而是将这个责任交给一个专门渲染一个<font color="red">相机的新类</font>

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CameraRenderer {
	ScriptableRenderContext context;
	Camera camera;

	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;
	}
}
```

<font color="red" >**CustomRenderPipeline**</font>中创建一个渲染器实例，然后使用它来渲染循环中的<font color="RoyalBlue">所有摄像机</font>

```c#
//渲染器实例
CameraRenderer renderer = new CameraRenderer();

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(context, camera);
		}
	}
```

**CameraRenderer**.<font color="green">Render</font>是绘制它的相机可以看到的所有几何体。为了清晰起见，将特定任务隔离在单独的<font color="red">DrawVisibleGeometry</font>方法中

还没有使天空盒出现。那是因为我们向上下文发出的命令被缓冲了。我们必须通过调用<font color="red">Submit context</font>来提交排队的工作以供执行

[ScriptableRenderContext](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html).<font color="red" >Submit 将所有预定的命令提交给渲染循环执行。</font>

```c#
    public void Render (ScriptableRenderContext context, Camera camera) {
        this.context = context;
        this.camera = camera;
        //渲染天空盒
        DrawVisibleGeometry();
        Submit();
    }

    void Submit () {
        context.Submit();
    }

    void DrawVisibleGeometry () {
        context.DrawSkybox(camera);
    }
}
```

#### 2.2、Drawing the Skybox

相机的方向目前不会影响天空盒的渲染方式。我们将相机传递给`DrawSkybox`，但这仅用于确定是否应该绘制天空盒，这是通过相机的清除标志控制的

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/skybox.png" alt="场景" style="zoom:50%;" />

请注意，摄像机的方向目前并不影响天盒的渲染方式。我们将摄像机传递给<font color="green">DrawSkybox</font>，但这只是用来确定是否应该绘制天幕，这是由摄像机的<font color="red">clear flags</font>来控制的。

为了正确地渲染天盒以及整个场景，我们必须设置<font color="red">**view-projection matrix**</font>。这个变换矩阵结合了摄像机的位置和方向--视图矩阵--以及摄像机的透视或正投影--投影矩阵。它在着色器中被称为<font color="RoyalBlue">unity_MatrixVP</font>，是几何体被绘制时使用的着色器属性之一。你可以在框架调试器的<font color="green">ShaderProperties</font>部分检查这个矩阵，当一个绘制调用被选中时。

目前，<font color="green">unity_MatrixVP</font>矩阵总是相同的。我们必须通过<font color="RoyalBlue">SetupCameraProperties</font>方法将相机的属性应用于上下文。这将设置矩阵以及其他一些属性。在调用<font color="green">DrawVisibleGeometry</font>之前，在一个单独的<font color="green">Setup</font>方法中做这个。

```c#
public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;
    	Setup();
		DrawVisibleGeometry();
		Submit();
	}

	void Setup () {
        //设置相机属性
		context.SetupCameraProperties(camera);
	}
```

#### [2.3、Command Buffers](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.3)

创建一个缓冲区<font color="red">CameraRenderer</font>并将对它的引用存储在一个<font color="green">buffer</font>中。还要为缓冲区命名，以便我们可以在帧调试器中识别它。<font color="red">Render Camera</font>会做。

要执行缓冲区，在上下文中调用<font color="RoyalBlue">ExecuteCommandBuffer</font>，并把缓冲区作为参数。这是从缓冲区中复制命令，但并不清除它，如果我们想重新使用它，我们必须在事后明确地这样做。

因为执行和清除总是一起进行的，所以添加一个方法来做这两件事是很方便的。

```c#
public class CameraRenderer {

    const string bufferName = "Render Camera";

    CommandBuffer buffer = new CommandBuffer {
        name = bufferName
    };
    ...
        
  void Setup () {
		buffer.BeginSample(bufferName);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}

	void Submit () {
		buffer.EndSample(bufferName);
		ExecuteBuffer();
		context.Submit();
	}

	void ExecuteBuffer () {
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
}
    
```

>   [BeginSample](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.BeginSample.html)  添加一个命令以开始配置文件采样
>
>   [EndSample](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.EndSample.html)  添加一个命令以结束配置文件采样
>
>   [ExecuteCommandBuffer](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.ExecuteCommandBuffer.html) 指定要执行的命令缓冲区
>
>   可以使用<font color="red">**command buffers**</font>来注入<font color="green">profiler samples</font>，这些样本将同时出现在profiler 盒 frame debugger中，这是通过在适当的点调用`**BeginSample、EndSample**`来完成的，在我们的例子中是在开始处必须为这两种方法提供相同的样本名称，作为我们将使用缓冲区的名称。
>
>   其中要执行<font color="RoyalBlue">**buffer**</font>，`ExecuteCommandBuffer`请使用缓冲区作为参数调用上下文，它从缓冲区复制命令但不清除它，因为执行和清除总是一起完成的，所以添加一个同时执行这两项操作的方法会很方便

Camera.<font color="green">RenderSkyBox</font>示例现在被嵌套在<font color="red">Render </font>Camera.<font color="RoyalBlue">RenderSkyBox</font>中。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/render-camera-sample.png)

<center>Render camera sample.</center>

#### [2.4、Clearing the Render Target](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.4)

我们画的东西最终会被渲染到摄像机的渲染目标上，默认情况下是帧缓冲区，但也可以是渲染纹理。

之前绘制到该目标上的东西仍然存在，这可能会干扰我们现在正在渲染的图像。为了保证正常的渲染，我们必须清除渲染目标，以摆脱它的旧内容。这可以通过在命令缓冲区上调用<font color="red">ClearRenderTarget</font>来实现，它属于<font color="green">Setup</font>方法。

<font color="red">CommandBuffer.ClearRenderTarget</font>至少需要三个参数。前两个表明<font color="RoyalBlue">depth and color data</font>是否应该被清除，对两者来说都是真的。第三个参数是用于清除的颜色，对此我们将使用<font color="green">Color.clear</font>。

```c#
	void Setup () {
		buffer.BeginSample(bufferName);
		buffer.ClearRenderTarget(true, true, Color.clear);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-nested-sample.png" alt="img" style="zoom:50%;" />

​																	Clearing, with nested sample.

帧调试器现在显示了清除动作的<font color="green">Draw GL</font>条目，它显示嵌套在<font color="RoyalBlue">Render Camera</font>的一个额外层次中。

这是因为<font color="red">ClearRenderTarget</font>用命令缓冲区的名字将清除动作包裹在一个样本中。我们可以通过在开始我们自己的样本之前进行清除来摆脱多余的嵌套。这导致了两个相邻的Render Camera样本范围，它们被合并了。

```c#
	void Setup () {
		buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(bufferName);
		//buffer.ClearRenderTarget(true, true, Color.clear);
		ExecuteBuffer();
		context.SetupCameraProperties(camera);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-one-sample.png" alt="img" style="zoom:50%;" />

​																		Clearing, without nesting.

Draw GL条目表示用<font color="green">Hidden/InternalClear</font>着色器绘制全屏四边形<font color="red">，该着色器会写入渲染目标</font>，这并不是最有效的清除方式。

使用这种方法是因为我们是在设置摄像机属性之前进行清除。

如果我们把这两个步骤的顺序对调一下，就可以得到快速的清除方式。

<font color="red" >**先设置好相机再清除，不会有冗余的数据**</font>

```c#
	void Setup () {
		context.SetupCameraProperties(camera);
		buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(bufferName);
		ExecuteBuffer();
		//context.SetupCameraProperties(camera);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/clearing-correct.png" alt="img" style="zoom:50%;" />

​																				Correct clearing.

#### [2.5、Culling](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.5)

我们目前看到的是天幕，但没有看到我们放在场景中的任何物体。与其绘制每一个物体，我们不如只渲染那些对摄像机可见的物体。我们通过从场景中所有带有渲染器组件的对象开始，然后剔除那些在摄像机视域之外的对象。

为了弄清楚哪些物体可以被剔除，我们需要跟踪多个相机设置和矩阵，为此我们可以使用<font color="red" >ScriptableCullingParameters</font>结构。我们可以在摄像机上调用**TryGetCullingParameters**，而不是自己填充它。

它会返回参数是否被成功获取，因为对于退化的相机设置，它可能会失败。为了获得参数数据，我们必须把它作为一个输出参数，在它前面写上输出。在一个单独的Cull方法中做到这一点，该方法返回成功或失败。

我们不会绘制每个对象，而是只渲染相机可见的对象。为此，我们从场景中所有具有渲染器组件的对象开始，然后剔除那些落在相机视锥之外的对象。

>   [Camera](https://docs.unity3d.com/ScriptReference/Camera.html).[TryGetCullingParameters](https://docs.unity3d.com/ScriptReference/Camera.TryGetCullingParameters.html) 获取相机的剔除参数P
>
>   [ScriptableCullingParameters](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableCullingParameters.html) 在可编写脚本的渲染管道中配置剔除操作的参数。

```c#
bool Cull () {
    if (camera.TryGetCullingParameters(out ScriptableCullingParameters p)) {
        return true;
    }
    return false;
}
```

> 为什么要写out？
> 当struct参数被定义为输出参数时，它的作用就像一个对象引用，指向参数所在的内存堆栈上的位置。
>
> Out关键字告诉我们，该方法负责正确设置参数，替换以前的值。
> Try-get方法是表示成功或失败并产生结果的常见方法。

当作为输出参数使用时，可以在参数列表中内联变量声明，所以我们来做这个。

```c#
	bool Cull () {
		//ScriptableCullingParameters p
		if (camera.TryGetCullingParameters(out ScriptableCullingParameters p)) {
			return true;
		}
		return false;
	}
```

在<font color="green">Render</font>中的<font color="red">Setup</font>之前调用<font color="green">Cull</font>，如果失败则中止。

```c#
	public void Render (ScriptableRenderContext context, Camera camera) {
		this.context = context;
		this.camera = camera;

		if (!Cull()) {
			return;
		}

		Setup();
		DrawVisibleGeometry();
		Submit();
	}
```

实际的剔除是通过在<font color="RoyalBlue">context</font>中调用Cull来完成的，它会产生一个<font color="red">CullingResults</font>结构。如果成功的话就在Cull中进行，并将结果存储在一个字段中。

在这种情况下，我们必须把剔除参数<font color="green">（ culling parameters）</font>作为一个引用参数来传递，在它前面写上ref。

```c#
CullingResults cullingResults;

	…
	
	bool Cull () {
		if (camera.TryGetCullingParameters(out ScriptableCullingParameters p)) {
			cullingResults = context.Cull(ref p);
			return true;
		}
		return false;
	}
```

>   [ScriptableRenderContext](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html) .Cull <font color="red" >基于正在渲染的相机中获取的 ScriptableCullingParameters 执行剔除</font>
>
>   <font color="red" >cullingResults会被送到DrawVisibleGeometry 进行渲染</font>
>
>   为什么我们必须使用**ref**？
>
>   **ref**关键字的工作方式与 类似**out**，只是方法不需要为其赋值。调用该方法的人负责首先正确初始化该值。所以它可以用于输入，也可以选择用于输出。
>
>   在这种情况下**ref**用作优化，以防止传递ScriptableCullingParameters相当大的结构副本。它是一个结构而不是一个对象是另一种优化，以防止内存分配。
>



#### [2.6、Drawing Geometry](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.6)

<font color="red" >一旦我们知道什么是可见的，我们就可以继续渲染这些东西了。</font>

这可以通过在上下文中调用<font color="green" >DrawRenderers</font>来完成，并将剔除结果作为一个参数，告诉它要使用哪些渲染器（renderers）。

除此之外，我们还必须提供绘制设置和过滤设置。两者都是结构体--绘图设置<font color="green" >（DrawingSettings）</font>和过滤设置<font color="green" >（FilteringSettings）</font>，我们最初会使用它们的默认构造函数。两者都必须通过引用来传递。在绘制天空盒之前，在DrawVisibleGeometry中这样做。

```c#
void DrawVisibleGeometry () {
		var drawingSettings = new DrawingSettings();
		var filteringSettings = new FilteringSettings();

		context.DrawRenderers(
			cullingResults, ref drawingSettings, ref filteringSettings
		);
		//绘制天空盒
		context.DrawSkybox(camera);
	}
```

>   <font color="red" >[DrawRenderers ](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.DrawRenderers.html) 安排一组可见对象的绘制，并可选择覆盖 GPU 的渲染状态 </font>

我们还没有看到任何东西，因为我们还必须指出哪种<font color="green">Shader Pass</font>是允许的。由于我们在本教程中只支持Unli着色器，我们必须为<font color="green" >SRPDefaultUnlit</font>通道获取着色器标签ID，我们可以做一次并将其缓存在一个静态字段中。

```c#
	static ShaderTagId unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit");
```

将它作为<font color="red">DrawingSettings</font>构造函数的第一个参数，第二个参数是<font color="green">SortingSettings</font>结构值。将相机传递给<font color="RoyalBlue">sorting settings</font>的构造函数，因为它用于确定是否应用<font color="red">正交排序或基于距离</font>的排序。

```c#
void DrawVisibleGeometry () {
		var sortingSettings = new SortingSettings(camera);
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		);
		…
	}
```

>   [SortingSettings](https://docs.unity3d.com/ScriptReference/Rendering.SortingSettings.html) 该结构描述了在渲染过程中对对象进行排序的方法。
>
>   [DrawingSettings](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings.html) 描述了如何对可见对象进行排序 ( [sortingSettings](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings-sortingSettings.html) ) 以及使用哪个Shader传递 ( [shaderPassName](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings-shaderPassName.html) )。
>
>   [FilteringSettings ](https://docs.unity3d.com/ScriptReference/Rendering.FilteringSettings.html)结构描述了`FilteringSettings`如何过滤[ScriptableRenderContext.DrawRenderers](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.DrawRenderers.html)接收到的对象集，以便 Unity 绘制其中的一个子集。



```c#
	void DrawVisibleGeometry () {
		var sortingSettings = new SortingSettings(camera);
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		);
		…
	}
```

除此之外，我们还必须指出允许哪些渲染队列。传递<font color="green">RenderQueueRange.all</font>给<font color="red">FilteringSettings</font>构造函数，以便我们包含所有内容。

```c#
void DrawVisibleGeometry () {
    // 摄像机到排序设置的构造器中，因为它被用来确定是否适用正交或基于距离的排序。
    var sortingSettings = new SortingSettings(camera);
    //shader 、rendering way
    var drawingSettings = new DrawingSettings(
        unlitShaderTagId, sortingSettings
    );
    //允许渲染的队列
    var filteringSettings = new FilteringSettings(RenderQueueRange.all);
    
    //  we have to supply drawing settings and filtering settings
    context.DrawRenderers(
        cullingResults, ref drawingSettings, ref filteringSettings
    );

    context.DrawSkybox(camera);
}
```

![scene](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit.png)



<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/drawing-unlit-debugger.png" alt="debugger" style="zoom:50%;" />

<center>Drawing unlit geometry.</center>

绘图顺序是随意的。criteria我们可以通过设置排序设置的属性来强制执行特定的绘制顺序。让我们使用<font color="red">SortingCriteria.CommonOpaque。 </font>

>   [SortingCriteria](https://docs.unity3d.com/ScriptReference/Rendering.SortingCriteria.html) 如何在渲染过程中对对象进行排序。
>
>   [CommonOpaque](https://docs.unity3d.com/ScriptReference/Rendering.SortingCriteria.html) 不透明物体的典型分类渲染顺序。

```c#
	var sortingSettings = new SortingSettings(camera) {
			criteria = SortingCriteria.CommonOpaque
		};
```

<center><iframe src="https://gfycat.com/ifr/rawimpishalpinegoat" style="border: none; overflow: hidden; width: 380px; height: 380px"></iframe><center></center>

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/sorting.png" alt="debugger" style="zoom:50%;" />

<center>Common opaque sorting.</center>

物体现在或多或少都是从前向后绘制的，这对不透明的物体来说是很理想的。如果某个物体最终被画在其他物体的后面，它的隐藏片段就会被跳过，这就加快了渲染的速度。常见的不透明排序选项还考虑了一些其他标准，包括渲染队列和材料<font color="green" >（render queue and materials.）</font>。

#### [2.7、Drawing Opaque and Transparent Geometry Separately](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#2.7)

帧调试器（ frame debugger ）向我们展示了透明物体的绘制，<font color="green">但天空盒被绘制在所有不在不透明物体前面的物体上。</font>

天空盒被绘制在不透明几何体之后，所以它所有的隐藏片段都可以被跳过，但它却覆盖了透明几何体。发生这种情况是因为透明着色器不向深度缓冲区<font color="green" >depth buffe</font>写入。它们不会隐藏它们后面的东西，因为我们可以看穿它们。

解决方案是分别绘制不透明和透明几何体，<font color="red">先绘制不透明物体，然后绘制天空盒，然后才绘制透明物体</font>。

我们可以通过切换到<font color="red">RenderQueueRange.opaque</font>.来消除初始<font color="green">DrawRenderers</font>调用中的透明对象。

```c#
		var filteringSettings = new FilteringSettings(RenderQueueRange.opaque);
```

然后在绘制天空盒后再次调用<font color="green">DrawRenderers</font>。但在这之前，将渲染队列范围改为<font color="green">RenderQueueRange</font>.<font color="red">transparent</font>。同时将排序标准改为<font color="RoyalBlue">SortingCriteria.CommonTransparent</font>，并再次设置绘图设置的排序。这样就颠倒了透明物体的绘制顺序。

```C#
	context.DrawSkybox(camera);

		sortingSettings.criteria = SortingCriteria.CommonTransparent;
		drawingSettings.sortingSettings = sortingSettings;
		filteringSettings.renderQueueRange = RenderQueueRange.transparent;

		context.DrawRenderers(
			cullingResults, ref drawingSettings, ref filteringSettings
		);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/rendering/opaque-skybox-transparent.png" alt="debugger" style="zoom:50%;" />

​															Opaque, then skybox, then transparent.

### [3、Editor Rendering](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#3)

我们的RP能正确地绘制出Unlit的物体，但我们可以做一些事情来改善在Unity编辑器中使用它的体验。

#### 3.1、Drawing Legacy Shaders

因为我们的管线只支持<font color="red" bordercolor="red" >unlit shaders passes</font>，使用不同通道的物体不会被渲染，从而使它们不可见。虽然这是正确的，但它掩盖了一个事实，即场景中的一些物体使用了错误的着色器。所以让我们还是对它们进行渲染，但要分开进行。

如果有人从一个默认的Unity项目开始，然后切换到我们的RP，那么他们的场景中可能会有使用<font color="green">wrong shader</font>的对象。为了涵盖Unity的所有默认着色器，我们必须使用着色器标签ID来表示<font color="DarkOrchid ">Always、ForwardBase、PrepassBase、Vertex、VertexLMRGBM和VertexLM</font>通道。在一个静态数组中保持对这些的跟踪。

```c#
static ShaderTagId[] legacyShaderTagIds = {
		new ShaderTagId("Always"),
		new ShaderTagId("ForwardBase"),
		new ShaderTagId("PrepassBase"),
		new ShaderTagId("Vertex"),
		new ShaderTagId("VertexLMRGBM"),
		new ShaderTagId("VertexLM")
	};
```

在可见几何体之后的单独方法中绘制所有不支持的明暗器，只从第一pass开始。由于这些是无效的传递，结果无论如何都是错误的，所以我们并不关心其他的设置。我们可以通过<font color="red" >FilteringSettings.defaultValue</font>属性获得默认的过滤设置。

```c#
public void Render (ScriptableRenderContext context, Camera camera) {
		…

		Setup();
		DrawVisibleGeometry();
		DrawUnsupportedShaders();
		Submit();
	}

	…
	  // Editor Rendering
    void DrawUnsupportedShaders () {
        //shader 、rendering way
        var drawingSettings = new DrawingSettings(
            legacyShaderTagIds[0], new SortingSettings(camera)
        );
        // filtering settings for rendering queue
        var filteringSettings = FilteringSettings.defaultValue;
        context.DrawRenderers(
            cullingResults, ref drawingSettings, ref filteringSettings
        );
    }
}
```

我们可以通过在<font color="green">DrawSetting</font>上调用<font color="DarkOrchid ">SetShaderPassName</font>，并将绘图顺序索引和标签（index and tag ）作为参数来绘制多个<font color="green">Pass</font>。对数组中的所有<font color="green">passes</font>都这样做，从第二个开始，因为我们在构建绘图设置时已经设置了第一个 <font color="green">pass</font>。

```c#
var drawingSettings = new DrawingSettings(
			legacyShaderTagIds[0], new SortingSettings(camera)
		);
		for (int i = 1; i < legacyShaderTagIds.Length; i++) {
			drawingSettings.SetShaderPassName(i, legacyShaderTagIds[i]);
		}
```

[SetShaderPassName](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings.SetShaderPassName.html) 着色器通道的名称。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/standard-black.png" alt="img" style="zoom:50%;" />

​															<center>Standard shader renders black.</center>

#### [3.2、Error Material](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#3.2)

为了清楚地表明哪些对象使用了不受支持的Shader，我们将使用 Unity 的<font color="magenta" >error Shader</font>绘制它们。使用该着色器作为参数构建一个新材质，我们可以通过`Shader.Find`调用<font color="magenta" >*Hidden/InternalErrorShader*</font>字符串作为参数。通过静态字段缓存材质，这样我们就不会在每一帧都创建一个新的材质。

然后将其分配给<font color="green">overrideMaterial</font>绘图设置的属性。

[overrideMaterial](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings-overrideMaterial.html) Sets the Material to use for all drawers that would render in this group. 设置 Material 以用于将在该组中呈现的所有抽屉。

```c++
	static Material errorMaterial;

	…

	void DrawUnsupportedShaders () {
		if (errorMaterial == null) {
			errorMaterial =
				new Material(Shader.Find("Hidden/InternalErrorShader"));
		}
		var drawingSettings = new DrawingSettings(
			legacyShaderTagIds[0], new SortingSettings(camera)
		) {
			overrideMaterial = errorMaterial
		};
		…
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/standard-magenta.png" alt="img" style="zoom:50%;" />

<center>现在所有无效的对象都可见并且显然是错误的</center>

#### [3.3、Partial Class](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#3.3)

绘制无效对象对开发很有用，但不适用于已发布的应用程序。因此，让我们将所有仅用于编辑器的代码<font color="green">CameraRenderer</font>放在一个单独的部分类文件中。从复制原件开始<font color="red">CameraRenderer</font>脚本资产并将其重命名为<font color="DarkOrchid ">CameraRenderer.Editor</font>



<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/two-assets.png" alt="img" style="zoom:50%;" />

​	<center>One class, two script assets.</center>

把原来的<font color="green">CameraRenderer</font>变成<font color="red">partial class</font>，去掉和<font color="DarkOrchid ">error material </font>有关的方法

```c++
public partial class CameraRenderer { … }
```

>   它是一种将一个类或结构定义分割成多个部分的方法，存储在不同的文件中。唯一的目的是为了组织代码。典型的使用情况是将自动生成的代码与手工编写的代码分开。就编译器而言，这都是同一个类定义的一部分。它们在《对象管理，更复杂的层次》教程中被介绍过。

清理另一个部分类文件，使其只包含我们从另一个文件中删除的内容。

```c#
using UnityEngine;
using UnityEngine.Rendering;

partial class CameraRenderer {

	static ShaderTagId[] legacyShaderTagIds = {	… };

	static Material errorMaterial;

	void DrawUnsupportedShaders () { … }
}
```

编辑器部分的内容只需要存在于编辑器中，所以让它以<font color="red" >UNITY_EDITOR</font>为条件。

```c#
partial class CameraRenderer {

#if UNITY_EDITOR

	static ShaderTagId[] legacyShaderTagIds = { … }
	};

	static Material errorMaterial;

	void DrawUnsupportedShaders () { … }

#endif
}
```

但是，此时构建将失败，因为另一部分始终包含 的调用<font color="green">DrawUnsupportedShaders</font>现在仅在编辑器中存在。为了解决这个问题，我们也使该方法成为部分方法。我们通过始终<font color="red">partial</font>在其前面声明方法签名来做到这一点，类似于抽象方法声明。

我们可以在类定义的任何部分这样做，所以让我们把它放在编辑器部分。完整的方法声明也必须用 标记**partial**。

```c#
	partial void DrawUnsupportedShaders ();

#if UNITY_EDITOR

	…

	partial void DrawUnsupportedShaders () { … }

#endif
```

构建的编译工作现在成功了。编译器将剥离所有最终没有完整声明的部分方法的调用。

>   我们可以让无效的对象出现在开发构建中吗？
>   是的，你可以基于UNITY_EDITOR || DEVELOPMENT_BUILD来进行条件编译。然后DrawUnsupportedShaders也会存在于开发构建中，而在发布构建中仍然不会出现。但在这个系列中，我将始终把所有与开发有关的东西只限制在编辑器中。

#### [3.4、Drawing Gizmos](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#3.4)

<font color="green">Currently our RP</font> 不绘制<font color="red">gizmos</font>，无论是在场景窗口还是在游戏窗口，如果它们在那里被启用的话。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/without-gizmos.png" alt="img" style="zoom:50%;" />

​																		Scene without gizmos.

我们可以通过调用<font color="DarkOrchid ">UnityEditor.Handles.ShouldRenderGizmos </font>来检查是否应该绘制<font color="green">gizmos</font>。如果是的话，我们必须在<font color="red">context </font>中调用<font color="green">DrawGizmos</font>，将相机作为一个参数，再加上第二个参数来表示应该绘制哪个gizmo子集。

有两个子集，用于图像效果(image effects)之前和之后。由于我们目前不支持图像效果，我们将同时调用这两个子集。在一个新的编辑器专用的DrawGizmos方法中这样做。

```C#
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;

partial class CameraRenderer {
	
	partial void DrawGizmos ();

	partial void DrawUnsupportedShaders ();

#if UNITY_EDITOR

	…

	partial void DrawGizmos () {
		if (Handles.ShouldRenderGizmos()) {
			context.DrawGizmos(camera, GizmoSubset.PreImageEffects);
			context.DrawGizmos(camera, GizmoSubset.PostImageEffects);
		}
	}

	partial void DrawUnsupportedShaders () { … }

#endif
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/with-gizmos.png" alt="img" style="zoom:50%;" />

<center>Scene with gizmos.</center>

#### 3.5、Drawing Unity UI

另一个需要我们注意的是Unity的游戏内用户界面。例如，通过<font color="green">GameObject / UI / Button</font>添加一个按钮来创建一个简单的用户界面。它将显示在游戏窗口中，但不会显示在场景窗口中。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-button.png" alt="img" style="zoom:50%;" />

<font color="green">UI button in game window.</font>

>   当*Render Mode*画布组件的设置为*Screen Space - Overlay*，这是默认值。将其更改为*Screen Space - Camera*并使用主摄像头作为其*Render Camera*将使它成为透明几何体的一部分
>

帧调试器向我们展示了 UI 是单独渲染的，而不是由我们的 RP 渲染的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-debugger.png" alt="img" style="zoom:50%;" />

<center>*UI in frame debugger.*</center>

至少，当画布组件的渲染模式被设置为<font color="DarkOrchid ">*Screen Space - Overlay*</font>时，情况就是这样，这是默认的。把它改为 "<font color="DarkOrchid ">*Screen Space - Camera* </font>"，并使用主摄像机作为它的渲染摄像机，将使它成为透明几何的一部分。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/ui-camera-debugger.png" alt="img" style="zoom:50%;" />

<center>Screen-space-camera UI in frame debugger.</center>

用户界面始终使用<font color="green">World Space</font>它在场景窗口中渲染时的模式，这就是它通常最终变得非常大的原因。但是，虽然我们可以通过场景窗口编辑 UI，但它不会被绘制出来。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/editor-rendering/invisible-ui-scene.png" alt="img" style="zoom:50%;" />

在渲染场景窗口时，我们必须明确地将 UI 添加到世界几何体中，方法是<font color="green">ScriptableRenderContext.EmitWorldGeometryForSceneView</font>将相机作为参数进行调用。

在一个新的仅限编辑器的<font color="red">PrepareForSceneWindow</font>方法中执行此操作。<font color="green">cameraType</font>当它的属性等于时，我们正在使用场景相机进行渲染。<font color="RoyalBlue">CameraType.SceneView</font>

>   [EmitWorldGeometryForSceneView](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.EmitWorldGeometryForSceneView.html) 将 UI 几何体发射到场景视图中进行渲染。
>
>   [CameraType](https://docs.unity3d.com/ScriptReference/30_search.html?q=CameraType) .SceneView 用于指示相机用于在编辑器中渲染场景视图。

```c#
	partial void PrepareForSceneWindow ();

#if UNITY_EDITOR

	…

	partial void PrepareForSceneWindow () {
		if (camera.cameraType == CameraType.SceneView) {
			ScriptableRenderContext.EmitWorldGeometryForSceneView(camera);
		}
	}
```

因为这可能会给场景增加几何图形，所以必须在剔除之前完成。

```c#
PrepareForSceneWindow();
		if (!Cull()) {
			return;
		}
```



### [4、Multiple Cameras](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#4)

场景中可能有不止一个活动的摄像机。如果是这样，我们必须确保它们一起工作。

#### [4.1、Two Cameras](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#4.1)

每个摄像头都有一个<font color="green">Depth</font>值，默认主相机为-1。它们按深度递增的顺序呈现。要查看此内容，请复制<font color="red">Main Camera</font>, 重命名为<font color="green">Secondary Camera</font>, 并设置其Depth到 0。给它另一个标签也是个好主意，因为<font color="green">MainCamera</font>应该只由一个相机使用。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/two-cameras-sample-sample.png" alt="img" style="zoom:50%;" />

<center>两台相机都归入了一个样本范围。</center>

场景现在被渲染两次。生成的图像仍然相同，因为渲染目标在两者之间被清除。帧调试器显示了这一点，但是由于合并了具有相同名称的相邻示例范围，我们最终得到了一个<font color="green">Render Camera</font>范围。

如果每个相机都有自己的范围，那就更清楚了。为了实现这一点，添加一个编辑器专用<font color="green">PrepareBuffer</font>方法，使缓冲区的名称等于相机的名称。如果每个相机都有自己的范围，那就更清楚了。为了实现这一点，添加一个编辑器专用<font color="green">PrepareBuffer</font>方法，<font color="red">使缓冲区的名称等于相机的名称</font>。

```c#
	partial void PrepareBuffer ();

#if UNITY_EDITOR

	…
	
	partial void PrepareBuffer () {
		buffer.name = camera.name;
	}

#endif
```

在我们准备场景窗口之前调用它。

```c#
		PrepareBuffer();
		PrepareForSceneWindow();
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/separate-samples.png" alt="img" style="zoom:50%;" />

<center>Separate samples per camera.</center>

#### [4.2、Dealing with Changing Buffer Names](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#4.2)

虽然帧调试器现在为每个摄像机显示一个单独的示例层次结构，但当我们进入播放模式时，Unity 的控制台将充满警告我们的消息<font color="red" >*BeginSample*和*EndSample*计数必须匹配</font>。

它变得混乱，因为我们对<font color="green">sample</font>及其<font color="green">buffer</font>使用了不同的名称。除此之外，我们每次访问相机的<font color="red">name</font>属性时也会最终分配内存，所以我们不想在构建中这样做。

为了解决这两个问题，我们将添加一个<font color="green">SampleName</font>字符串属性。如果我们在编辑器中，我们将它<font color="red">PrepareBuffer</font>与缓冲区的名称一起设置，否则它只是一个常量别名<font color="green">Render Camera string</font>

```c#
#if UNITY_EDITOR

	…

	string SampleName { get; set; }
	
	…
	
	partial void PrepareBuffer () {
		buffer.name = SampleName = camera.name;
	}

#else

	const string SampleName = bufferName;

#endif
```

用于和<font color="green">SampleName</font>中的示例。<font color="green">`Setup` and `Submit`.</font>

```c#
	void Setup () {
		context.SetupCameraProperties(camera);
		buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(SampleName);
		ExecuteBuffer();
	}

	void Submit () {
		buffer.EndSample(SampleName);
		ExecuteBuffer();
		context.Submit();
	}
```

我们可以通过检查探查器来看到差异——通过以下方式打开*<font color="green">Window / Analysis / Profiler</font>*- 首先在编辑器中播放。

切换到<font color="green">Hierarchy</font>模式并按GC Alloc柱子。您会看到两次调用的条目GC.Alloc, 共分配100字节，这是由于相机名称的检索导致的。再往下你会看到这些名字显示为样本：<font color="green">Main Camera和Secondary Camera.</font>

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/profiler-cg-alloc.png" alt="img" style="zoom:50%;" />

<center>具有单独样本和 100B 分s配的分析器。</center>

接下来，用开发构建和自动连接剖析器的功能进行构建。播放该构建，并确保剖析器被连接并记录。在这种情况下，我们不会得到100字节的分配，而是得到单一的<font color="green">Render Camera</font>样本。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/profiler-build.png" alt="img" style="zoom:50%;" />

<center>Profiling build.</center>

>   其他48个字节是用来做什么的？
>   这是为相机阵列准备的，我们无法控制它。它的大小取决于有多少个摄像机被渲染出来。

我们可以通过将相机名称检索包装在名为<font color="red">Editor Only</font>. 在这种情况下，我们需要从命名空间调用<font color="green">Profiler.BeginSample</font>和。只需要传递名称<font color="DarkOrchid ">Profiler.EndSampleUnityEngine.ProfilingBeginSample</font>。

```c#
using UnityEditor;
using UnityEngine;
using UnityEngine.Profiling;
using UnityEngine.Rendering;
	
#if UNITY_EDITOR
    ...

	partial void PrepareBuffer () {
		Profiler.BeginSample("Editor Only");
		buffer.name = SampleName = camera.name;
		Profiler.EndSample();
	}

#else

	string SampleName => bufferName;

#endif
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/editor-only-allocations.png" alt="img" style="zoom:50%;" />

#### 4.3、Layers

相机也可以配置为只看到某些层上的东西。这是通过调整他们的*Culling Mask*. 为了实际看到这一点，让我们将所有使用标准着色器的对象移动到*Ignore Raycast*层。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/ignore-raycast-layer.png" alt="img" style="zoom:50%;" />

<center>图层切换到*Ignore Raycast*.</center>

从剔除蒙版中排除该层<font color="green">Main Camera.</font>

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/culling-ignore-raycast.png" alt="img" style="zoom:50%;" />

<center>剔除*Ignore Raycast*层。</center>

并使其成为唯一可见的层<font color="green">*Secondary Camera*.</font>

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/only-ignore-raycast.png" alt="img" style="zoom:50%;" />

<center>剔除一切，但*Ignore Raycast*层</center>
因为*<font color="green">Secondary Camera</font>*最后渲染我们最终只看到无效的对象。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/only-ignore-raycast-game.png" alt="img" style="zoom:50%;" />

<center>Only *Ignore Raycast* layer visible in game window.</center>

#### [4.4、Clear Flags](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/#4.4)

我们可以通过调整[clear flags](https://docs.unity3d.com/ScriptReference/CameraClearFlags.html) 来组合两个相机的结果

它们由一个枚举定义，我们可以通过相机的属性<font color="green">CameraClearFlags</font>检索该枚举。在清理之前<font color="green">clearFlags</font>执行此操作

```c#
	void Setup () {
		context.SetupCameraProperties(camera);
		
        CameraClearFlags flags = camera.clearFlags;
		
        buffer.ClearRenderTarget(true, true, Color.clear);
		buffer.BeginSample(SampleName);
		ExecuteBuffer();
	}
```

<font color="red">CameraClearFlags</font>枚举定义了四个值。从 1 到 4 它们是<font color="green">Skybox, Color, Depth, 和Nothing</font>。这些实际上不是独立的标志值，而是代表减少的清算量。除最后一种情况外，所有情况下都必须清除深度缓冲区，因此当标志值最多为Depth.

当标志设置为Color时，我们<font color="red" >只真正需要清除颜色缓冲区</font>，因为在这种情况下，Skybox我们最终会替换所有以前的颜色数据。

```C#
	buffer.ClearRenderTarget(
			flags <= CameraClearFlags.Depth,
			flags == CameraClearFlags.Color,
			Color.clear
		);
```

如果我们清除纯色，我们必须使用相机的背景色。但是因为我们在线性颜色空间中渲染，我们必须将该颜色转换为线性空间，所以我们最终需要<font color="green">camera.backgroundColor.linear.</font> 在所有其他情况下，颜色无关紧要，因此我们可以使用<font color="red">Color.clear</font>.

```c#
		buffer.ClearRenderTarget(
			flags <= CameraClearFlags.Depth,
			flags == CameraClearFlags.Color,
			flags == CameraClearFlags.Color ?
				camera.backgroundColor.linear : Color.clear
		);
```

因为<font color="green">Main Camera</font>是第一个渲染的，它的<font color="green">Clear Flags</font>应设置为<font color="DarkOrchid ">Skybox或Color</font>。当启用<font color="RoyalBlue">frame debugge</font>时，我们总是从一个<font color="red">clear buffer</font>开始，但通常不能保证这一点。

The clear flags of *Secondary Camera*确定两个相机的渲染如何组合。在skybox或color的情况下，之前的结果将被完全替换。

only depth is cleared,Secondary Camera*除了不绘制skybox外，渲染正常，因此之前的结果显示为背景。

nothing gets cleared ，深度缓冲区将被保留，因此未点亮的对象最终会遮挡无效对象，就好像它们是由同一台相机绘制的一样。然而，之前的相机绘制的透明物体没有深度信息，所以被绘制了过来，就像之前的天空盒所做的那样。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-color.png" alt="color" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-depth.png" alt="depth" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/clear-nothing.png" alt="nothing" style="zoom:50%;" />

<center>Clear color, depth-only, and nothing.</center>

通过调整相机的视口矩形，也可以将渲染区域缩小到整个渲染目标的一小部分。渲染目标的其余部分仍然不受影响。在这种情况下，通过<font color="green">Hidden/InternalClear</font>着色器进行清除。模板缓冲区被用来限制渲染在视口区域内。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/multiple-cameras/reduced-viewport.png" alt="img" style="zoom:50%;" />

<center>Reduced viewport of secondary camera, clearing color.</center>

请注意，每帧渲染一个以上的摄像机意味着删减、设置、排序等工作也要进行多次。每个独特的视点使用一个摄像机通常是最有效的方法。