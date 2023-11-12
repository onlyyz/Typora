# Custom SRP 2.0.0

切换到Render Graph API。使用通行证分区工作。善用剖析

<font color=#4db8ff>Link：</font>https://catlikecoding.com/unity/custom-srp/2-0-0/

![img](https://catlikecoding.com/unity/custom-srp/2-0-0/tutorial-image.jpg)

<center>*The Particles scene demonstrates reading color and depth.*</center>

### 1、Building a Render Graph

在 Unity 2020 开发周期的某个阶段，Unit 在 <font color=#66ff66> RP Core Library </font>中添加了<font color="red">render graph system</font>。虽然我们现在使用的是 Unity 2022，但该系统仍处于实验阶段。这意味着其 API 在未来可能会发生变化，但我们将继续迁移我们的渲染管道以使用它，就像 HDRP 和 URP 正在做的那样。这是一个破坏性的变化，不可能轻易转换回 Unity 2019，因此我们将版本提高到 2.0.0。

请注意，对于我们当前的<font color=#66ff66> render pipeline </font>来说，<font color="red"> render graph system</font>可谓矫枉过正。在 2.0.0 中，我们不会从中获得任何好处，只会增加额外的开销。我们只需进行最少量的转换工作，就能得到一个功能完善的 RP <font color=#66ff66> render graph version</font>，并且仍具有与之前相同的功能。这大致也是 URP 目前的状态，不过对我们来说更简单，因为我们不支持禁用渲染图。

#### 1.1 Profiling

为了演示我们的基线功能，我们使用 Particles 场景作为参考点，并查看其剖析数据。根据我们<font color="red">register</font>的<font color=#66ff66>profiling</font>和<font color="red"> command buffer sample</font>

<font color=#4db8ff>profiler </font>显示了我们的 RP 在 PlayerLoop / RenderPipelineManager.DoRenderLoop_Internal() [Invoke] 下的 CPU 工作情况。

请注意，<font color="red">Editor Only </font>会为编辑器中的每一帧分配内存，因为我们会检索摄像机的名称。

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/building-a-render-graph/profiler.png" alt="img" style="zoom:50%;" />

<center>*Profiler showing the CPU work of our RP.*</center>

The <font color=#66ff66>frame debugger</font> shows only the command buffers samples, in execution order.

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/building-a-render-graph/frame-debugger.png" alt="img" style="zoom:50%;" />

<center>*Frame debugger.*</center>

#### 1.2 One Render Graph

要访问<font color="red">Render Graph API</font>，我们需要使用其<font color=#4db8ff>experimental namespace</font>。将其添加到 <font color=#FFCE70>CustomRenderPipeline </font>中，之后再根据需要添加到所有 C# 脚本中。

```C#
using UnityEngine.Experimental.Rendering.RenderGraphModule;
```

<font color=#66ff66>render pipeline</font>通过<font color="red">single render graph</font>完成所有工作。我们在 <font color=#FFCE70>CustomRenderPipeline </font>中创建并跟踪它，将其命名为<font color=#4db8ff>*Custom SRP Render Graph*.</font>。我们必须在<font color="red">RP disposed</font>时调用 <font color=#66ff66>Cleanup</font>，这样它就会清理它所管理的所有数据。

```C#
	readonly RenderGraph renderGraph = new("Custom SRP Render Graph");

	…

	protected override void Dispose(bool disposing)
	{
		…
		renderGraph.Cleanup();
	}
```

所有摄像机都使用同一个<font color="red">render graph</font>进行渲染。在对每个摄像机调用渲染时，我们会将其作为新的第一个参数传递给它。渲染完成后，我们会在<font color="red">render graph</font>上调用 <font color="red">EndFrame</font>

<font color=#4db8ff>这样它就能清理并释放不再需要的内容</font>

```C#
	protected override void Render(ScriptableRenderContext context, List<Camera> cameras)
	{
		for (int i = 0; i < cameras.Count; i++)
		{
			renderer.Render(
				renderGraph, context, cameras[i], cameraBufferSettings,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings, postFXSettings, colorLUTResolution);
		}
		renderGraph.EndFrame();
	}
```

这就是 <font color="#66ff66">CustomRenderPipeline </font>对<font color=#4db8ff> render graph</font>所需要做的一切。

<font color=#FFCE70>CameraRenderer.Render.</font>

```C#
	public void Render(
		RenderGraph renderGraph,
		ScriptableRenderContext context, Camera camera,
		CameraBufferSettings bufferSettings,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings, PostFXSettings postFXSettings,
		int colorLUTResolution)
	{ … }
```

#### 1.3 Recording and Executing

此时，我们有了一个活动的<font color=#66ff66>render graph</font>，但它还没有做任何事情。为了让<font color=#66ff66>render graph</font>有所作为，我们必须调用 <font color="red">RecordAndExecute</font>，并将新的 <font color=#66ff66>RenderGraphParameters </font>结构作为参数。它返回的睡衣我们立即调用 <font color=#FFCE70>Dispose </font>的内容。

在 <font color=#4db8ff>CameraRenderer.Render </font>结束后，清理并提交之前执行此操作。

```C#
	public void Render(…)
	{
		…
		DrawGizmosAfterFX();
		
		var renderGraphParameters = new RenderGraphParameters { };
		renderGraph.RecordAndExecute(renderGraphParameters).Dispose();

		Cleanup();
		Submit();
	}
```

这将导致<font color=#66ff66>render graph</font>产生<font color="red">Bug</font>。无论它在抱怨什么，原因都是我们没有通过参数结构给它一个<font color=#FFCE70>command buffer</font>。在这里，我们复制 Unity 自己的方法，通过 <font color=#4db8ff>CommandBufferPool</font> 获取一个<font color=#66ff66>pooled command buffer object</font>。在渲染结束时释放它。

虽然可以为命令缓冲区命名，但我们不应这样做。从池中获取<font color="red">command buffer</font>而不为其命名，会将缓冲区的名称设置为空字符串，只有这样才能与渲染图生成剖析器数据的方式保持一致。

```C#
	var renderGraphParameters = new RenderGraphParameters
		{
			commandBuffer = CommandBufferPool.Get()
		};
		renderGraph.RecordAndExecute(renderGraphParameters).Dispose();
		
		Cleanup();
		Submit();
		CommandBufferPool.Release(renderGraphParameters.commandBuffer);
```

现在，<font color=#66ff66>profiler </font>将显示一些新内容：<font color=#FFCE70>ExecuteRenderGraph</font>（执行渲染图）和 <font color=#FFCE70>CompileRenderGraph</font>（编译渲染图），两者都以 Inl_ 作为前缀。前缀代表内联，指的是渲染图的记录和编译阶段。

内联执行部分包括调用 <font color="red">RecordAndExecute 和 Dispose </font>之间发生的一切，目前没有任何内容。编译部分在这之后，目前也什么都没做，因为图是空的。出于同样的原因，也没有任何实际执行。

![img](https://catlikecoding.com/unity/custom-srp/2-0-0/building-a-render-graph/profiler-render-graph.png)

<center>*Profiler showing inline execution and compilation.*</center>

为了使我们对<font color=#66ff66>render graph</font>的使用完全正确，我们还必须向它提供<font color="red"> frame index</font>（当前帧索引）、<font color=#FFCE70>execution name</font>（执行名称）（我们将使用 Render Camera）以及 <font color=#4db8ff>ScriptableRenderContext </font>实例的引用。

```C#
var renderGraphParameters = new RenderGraphParameters
{
    commandBuffer = CommandBufferPool.Get(),
    currentFrameIndex = Time.frameCount,
    executionName = "Render Camera",
    scriptableRenderContext = context
};
```

#### 1.4 Adding a Pass

在调用 <font color=#FFCE70>RecordAndExecute 和 Dispose </font>时，我们会依次向<font color=#66ff66>A render graph</font>中添加<font color=#4db8ff>Pass</font>。

为了方便保证这一阶段的正确结束，<font color="red">Render Graph AP</font>使用了一次性模式。因此，我们可以使用 <font color=#FFCE70>using</font> 关键字在后面的代码块中声明我们正在使用 <font color="DarkOrchid ">RecordAndExecute </font>返回的内容。

当代码执行离开该范围时，即使产生了异常，所使用的内容也会被隐式处理掉。

```C#
using (renderGraph.RecordAndExecute(renderGraphParameters)) {
// Add passes here.
}
```

>   <center>How does ` <font color="DarkOrchid ">using</font>` work?</center>
>
>   这是一种<font color="red">syntactic sugar</font>语法糖，相当于编写以下内容：
>
>   ```c++
>   var disposable = renderGraph.RecordAndExecute(renderGraphParameters);
>   try
>   {
>   	// Add passes here.
>   }
>   finally
>   {
>   	disposable.Dispose();
>   }
>   ```
>
>   

在<font color=#66ff66>render graph</font>上调用 <font color="red">AddRenderPass </font>可以添加<font color=#4db8ff>Pass</font>，它至少需要两个参数。第一个参数是<font color=#4db8ff>Pass</font>的名称。第二个参数是一个输出参数，其类型是一个具有默认构造函数的类。

这样做的目的是让<font color=#66ff66>render graph</font>创建并汇集这些类型的实例，用于存储传递所需的数据。举个简单的例子，我们将添加一个名为 Test Pass 的<font color=#4db8ff>Pass</font>，并使用 <font color="red">CameraSettings </font>作为其数据类型，因为它已经存在，而且没有任何特殊功能。

```C#
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    renderGraph.AddRenderPass("Test Pass", out CameraSettings data);
}
```

添加<font color=#4db8ff>render pass</font>也使用了<font color=#66ff66>disposable pattern</font>，在这种情况下，我们需要返回一个 <font color="red">RenderGraphBuilder </font>来构建<font color=#4db8ff>pass </font>，并在完成后将其<font color=#FFCE70>dispose </font>。

```C#
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    using RenderGraphBuilder builder =
        renderGraph.AddRenderPass("Test Pass", out CameraSettings data);
    //builder.Dispose();
}
```

每个<font color=#66ff66>Pass</font>都需要一个<font color=#FFCE70>render function</font>。这是一个带有两个参数的<font color="red">delegate </font>（委托）

第一个参数用于传递数据，第二个参数用于<font color=#66ff66> render graph</font>提供的 <font color="DarkOrchid ">RenderGraphContext </font>实例。

可以通过调用<font color=#4db8ff>builder</font>上的 <font color=#66ff66>SetRenderFunc </font>来设置。由于这只是一个示例，我们使用 lambda 表达式为其赋予一个空的匿名函数。

```C#
using RenderGraphBuilder builder =
    renderGraph.AddRenderPass("Test Pass", out CameraSettings data);
builder.SetRenderFunc(
    (CameraSettings data, RenderGraphContext context) => { });
```

我们的测试通过现在显示在<font color="red">profiler</font>中，作为其内联执行的一部分，也就是其记录阶段。

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/building-a-render-graph/profiler-test-pass.png" alt="img" style="zoom:50%;" />

<center>*Profiler showing test pass.*</center>



------



### 2、Converting to Passes

既然我们已经知道了如何创建<font color=#4db8ff>Passes</font>，那么我们就来转换现有代码，将其拆分成多个<font color=#4db8ff>Pass</font>。

在这个版本中，我们只做了最少量的修改，因此大部分代码都保留在现有位置。我们不会将代码移出 <font color=#FFCE70>CameraRenderer</font>，而是在需要时将其公开。在 <font color=#FFCE70>CameraRenderer.<font color="red">Render</font> </font>中的任何本地代码，只要是<font color=#4db8ff>Pass</font>需要的，我们都会将其沿着<font color=#4db8ff>Pass</font>存储在它们的<font color=#66ff66>fields</font>中。

这是一种糟糕的编码实践，但我们会在未来的版本中适当隔离这些代码。

当前的<font color=#66ff66> render graph </font>设计基于使用<font color=#FFCE70>data-only classes</font>和单独的匿名函数或静态方法来处理呈现代码。

这是一种低级设计，无法使用面向对象的 C# 功能。HDRP 和 URP 通过传递大量引用来解决这个问题。因此，这些 RP 的代码相当混乱，尤其是因为它们仍在向完全使用<font color=#66ff66>render graph</font>过渡，就像我们一样。

为了让我们的代码简单明了，如果我们能有一个带有

```c++
void Render(RenderGraphContext context) 
```

方法的<font color=#66ff66>pass </font>接口，那就更方便了。因此，当我们创建自己的<font color=#66ff66>pass </font>时，让我们假设情况就是这样。

我们将从头到尾创建<font color=#66ff66>pass </font>，反复将现有代码的一部分引入 <font color="red">RecordAndExecute </font>块，同时在整个过程中保持 RP 的功能。

#### 2.1 Gizmos Pass

我们的 RP 在清理之前要做的最后一件事就是绘制<font color=#66ff66>gizmos</font>，分为特效前和特效后两种，但没有找到<font color=#4db8ff>pre-FX gizmos </font>的例子，也无法创建任何特效后<font color=#66ff66>gizmos</font>，因此认为它们实际上并不存在。因此，我们只需在最后将两者画在一起即可。

创建一个<font color="red"> GizmosPass </font>类，放在 <font color=#66ff66>Runtime </font>下的新 <font color=#66ff66>Passes </font>文件夹中。

让它实现我们假想的接口，复制 <font color=#FFCE70>CameraRenderer.DrawGizmosBeforeFX </font>中的代码，并让它绘制特效后的<font color=#66ff66>gizmos</font>。它需要访问摄像机、是否使用了中间缓冲区以及调用绘制和执行方法，因此需要通过一个字段引用 <font color="DarkOrchid ">CameraRenderer</font>，并将这些成员设为公共成员。

最后，让所有内容只在编辑器中编译，并排除是否应绘制<font color=#66ff66>gizmos</font>的检查，因为如果不需要，我们将跳过该过程。

```C#
using System.Diagnostics;
using UnityEditor;
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class GizmosPass
{
    #if UNITY_EDITOR
        CameraRenderer renderer;

    void Render(RenderGraphContext context)
    {
        if (renderer.useIntermediateBuffer)
        {
            renderer.Draw(
                CameraRenderer.depthAttachmentId, BuiltinRenderTextureType.CameraTarget,
                true);
            renderer.ExecuteBuffer();
        }
        context.renderContext.DrawGizmos(renderer.camera, GizmoSubset.PreImageEffects);
        context.renderContext.DrawGizmos(renderer.camera, GizmoSubset.PostImageEffects);
    }
    #endif
}
```

>   ### Why isn't the new code marked?
>
>   我不再标记新资产的初始代码。否则，它将被完全标记。

要记录<font color=#4db8ff>pass</font>，需要引入一个公共静态 <font color=#66ff66>Record </font>方法，并为<font color="red">render graph</font>和我们的相机渲染器提供参数。该方法不能是局部的，因此我们使用<font color="red">conditional compilation</font>，只在编辑器中包含该方法。只有在为编辑器编译时才会包含其代码。在这里，我们要检查是否要渲染<font color="DarkOrchid ">gizmos</font>。

```C#
#if UNITY_EDITOR
    …
    #endif

    [Conditional("UNITY_EDITOR")]
    public static void Record(RenderGraph renderGraph, CameraRenderer renderer)
{
    #if UNITY_EDITOR
        if (Handles.ShouldRenderGizmos()) { }
    #endif
}
```

如果要对<font color=#66ff66>gizmos </font>进行渲染，我们可以使用 <font color="red">GizmosPass </font>类型添加一个<font color="DarkOrchid ">*Gizmos* render pass</font>。设置它的<font color=#4db8ff>renderer </font>，并赋予它一个调用其虚构接口方法的函数。

```C#
if (Handles.ShouldRenderGizmos())
{
    using RenderGraphBuilder builder =
        renderGraph.AddRenderPass("Gizmos", out GizmosPass pass);
    pass.renderer = renderer;
    builder.SetRenderFunc<GizmosPass>((pass, context) => pass.Render(context));
}
```

>   <details><summary><center>Can't we use closures?</center></summary><pre>
>   是的，但<font color="DarkOrchid ">closures</font>只是语法上的 "糖"，我们要做的是显式操作：用一个类封装整个过程，并创建一个实例。显式操作使我们有可能使用<font color="DarkOrchid "> the pooling of the render graph.</font>。
>   如果依赖<font color="DarkOrchid ">closures</font>，则每次在<font color=#4db8ff>render graph</font>中添加<font color="red">Pass</font>时都会分配一个新对象，每个相机每帧都可能分配一次。
>   当匿名函数从其外层作用域访问一些无法被拉入的内容（如方法参数）时，<font color="DarkOrchid ">Closures </font>就会隐式生成。
>   因此，必须只依赖函数本身的参数，而不要意外地访问 <font color="red">Record</font> 方法中的内容。在使用<font color="red">render graph</font>时，这是<font color=#66ff66>mysterious allocations </font>的潜在来源。
>   </pre></details>

从 <font color=#FFCE70>CameraRenderer</font>.<font color=#66ff66>Render </font>中移除<font color="DarkOrchid ">draw gizmos </font>的调用，并将测试通过替换为<font color="red">gizmos pass</font>的记录。

```C#
//DrawGizmosBeforeFX();
if (postFXStack.IsActive)
{
    postFXStack.Render(colorAttachmentId);
}
else if (useIntermediateBuffer)
{
    DrawFinal(cameraSettings.finalBlendMode);
    ExecuteBuffer();
}
//DrawGizmosAfterFX();

var renderGraphParameters = new RenderGraphParameters
{ … };
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    //using RenderGraphBuilder builder =
    //	renderGraph.AddRenderPass("Test Pass", out CameraSettings data);
    //builder.SetRenderFunc(
    //	(CameraSettings data, RenderGraphContext context) => { });
    GizmosPass.Record(renderGraph, this);
}
```

此时一切仍可正常工作，但如果需要的话，现在小玩意是通过<font color=#66ff66>render graph</font>绘制的。我们还可以删除 <font color="red">CameraRenderer </font>中的所有<font color="DarkOrchid "> gizmo-drawing</font>代码。

```C#
//partial void DrawGizmosBeforeFX();

//partial void DrawGizmosAfterFX();

…
    #if UNITY_EDITOR
    …

    //partial void DrawGizmosBeforeFX() { … }

    //partial void DrawGizmosAfterFX() { … }

    …
    #endif
```

#### 2.2 Final Pass

我们引入的第二个<font color=#4db8ff>Pass</font>是<font color="red">Final Pass</font>，用于在不使用后期特效的情况下渲染到中间缓冲区。它需要<font color=#FFCE70>camera renderer</font>和最终混合模式来完成工作。

```C#
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class FinalPass
{
    CameraRenderer renderer;

    CameraSettings.FinalBlendMode finalBlendMode;

    void Render(RenderGraphContext context)
    {
        renderer.DrawFinal(finalBlendMode);
        renderer.ExecuteBuffer();
    }

    public static void Record(
        RenderGraph renderGraph,
        CameraRenderer renderer,
        CameraSettings.FinalBlendMode finalBlendMode)
    {
        using RenderGraphBuilder builder =
            renderGraph.AddRenderPass("Final", out FinalPass pass);
        pass.renderer = renderer;
        pass.finalBlendMode = finalBlendMode;
        builder.SetRenderFunc<FinalPass>((pass, context) => pass.Render(context));
    }
}
```

Pull it into the <font color=#FFCE70>render graph</font>, leaving the <font color="DarkOrchid ">post FX</font> for the next step.

```C#
if (postFXStack.IsActive)
{
    postFXStack.Render(colorAttachmentId);
}
//else if (useIntermediateBuffer)
//{
//	DrawFinal(cameraSettings.finalBlendMode);
//	ExecuteBuffer();
//}

var renderGraphParameters = new RenderGraphParameters
{ … };
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    if (postFXStack.IsActive)
    { }
    else if (useIntermediateBuffer)
    {
        FinalPass.Record(renderGraph, this, cameraSettings.finalBlendMode);
    }
    GizmosPass.Record(renderGraph, this);
}
```

在我们的自定义<font color=#66ff66>camera component</font>上启用 "<font color="DarkOrchid "> *Override Post FX*</font>"，并将其后期特效设置设为 "无"，从而禁用 "<font color=#FFCE70>*Particles* scene</font> "场景的后期特效，就可以看到<font color=#66ff66>final pass</font>。如果后置特效被禁用，它也将用于场景窗口。

#### 2.3 Post FX Pass

接下来是<font color="red">Post FX pass</font>。它只需使用现有的<font color=#66ff66> post FX stack</font>，还需要知道<font color=#66ff66>color attachment ID</font>。在这里，我们只在<font color=#FFCE70>stack </font>上调用<font color="red">Render</font>。

其 <font color="red">Setup </font>方法仍将在<font color="DarkOrchid ">render graph</font>之外调用，因为它只是复制一些值并确定<font color=#FFCE70>stack </font>是否处于活动状态，而这是我们在记录<font color=#66ff66> recording the graph</font>之前需要知道的。

```C#
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class PostFXPass
{
    PostFXStack postFXStack;

    void Render(RenderGraphContext context) =>
        postFXStack.Render(CameraRenderer.colorAttachmentId);

    public static void Record(RenderGraph renderGraph, PostFXStack postFXStack)
    {
        using RenderGraphBuilder builder =
            renderGraph.AddRenderPass("Post FX", out PostFXPass pass);
        pass.postFXStack = postFXStack;
        builder.SetRenderFunc<PostFXPass>((pass, context) => pass.Render(context));
    }
}
```

Pull it into the render graph as well.

```C#
//if (postFXStack.IsActive)
//{
//	postFXStack.Render(colorAttachmentId);
//}

var renderGraphParameters = new RenderGraphParameters
{ … };
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    if (postFXStack.IsActive)
    {
        PostFXPass.Record(renderGraph, postFXStack);
    }
    …
}
```

#### 2.4 Unsupported Shaders Pass

下一Pass是绘制 RP 不支持的普通着色器。它封装了对我们的<font color="red"> DrawUnsupportedShaders</font> 方法的调用，但仅限于在编辑器中，使用的方法与 <font color=#66ff66>gizmos </font>过程相同。

```C#
using System.Diagnostics;
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class UnsupportedShadersPass
{
#if UNITY_EDITOR
	CameraRenderer renderer;

	void Render(RenderGraphContext context) => renderer.DrawUnsupportedShaders();
#endif

	[Conditional("UNITY_EDITOR")]
	public static void Record(RenderGraph renderGraph, CameraRenderer renderer)
	{
#if UNITY_EDITOR
		using RenderGraphBuilder builder = renderGraph.AddRenderPass(
			"Unsupported Shaders", out UnsupportedShadersPass pass);
		pass.renderer = renderer;
		builder.SetRenderFunc<UnsupportedShadersPass>(
			(pass, context) => pass.Render(context));
#endif
	}
}
```

Pull it into the render graph.

```C#
//DrawUnsupportedShaders();

var renderGraphParameters = new RenderGraphParameters
{ … };
using (renderGraph.RecordAndExecute(renderGraphParameters))
{
    UnsupportedShadersPass.Record(renderGraph, this);
    …
}
```

#### 2.5 Visible Geometry Pass

至此，我们完成了所有可见几何体的绘制。我们可以将这一过程分成多个步骤，但为了尽量简化，我们只需一个步骤，即转发到我们的 <font color="red">DrawVisibleGeometry </font>方法。它只需要复制一堆参数。

```C#
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class VisibleGeometryPass
{
	CameraRenderer renderer;

	bool useDynamicBatching, useGPUInstancing, useLightsPerObject;

	int renderingLayerMask;

	void Render(RenderGraphContext context) => renderer.DrawVisibleGeometry(
		useDynamicBatching, useGPUInstancing, useLightsPerObject, renderingLayerMask);

	public static void Record(
		RenderGraph renderGraph, CameraRenderer renderer,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		int renderingLayerMask)
	{
		using RenderGraphBuilder builder =
			renderGraph.AddRenderPass("Visible Geometry", out VisibleGeometryPass pass);
		pass.renderer = renderer;
		pass.useDynamicBatching = useDynamicBatching;
		pass.useGPUInstancing = useGPUInstancing;
		pass.useLightsPerObject = useLightsPerObject;
		pass.renderingLayerMask = renderingLayerMask;
		builder.SetRenderFunc<VisibleGeometryPass>(
			(pass, context) => pass.Render(context));
	}
}
```

#### 2.6 Setup Pass

接下来是设置过程，调用 <font color="red">Setup</font>，根据需要创建<font color=#66ff66>intermediate buffers</font>并清除渲染目标。

```C#
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class SetupPass
{
	CameraRenderer renderer;

	void Render(RenderGraphContext context) => renderer.Setup();

	public static void Record(
		RenderGraph renderGraph, CameraRenderer renderer)
	{
		using RenderGraphBuilder builder =
			renderGraph.AddRenderPass("Setup", out SetupPass pass);
		pass.renderer = renderer;
		builder.SetRenderFunc<SetupPass>((pass, context) => pass.Render(context));
	}
}
```

Pull it into the render graph.

```C#
	//Setup();

		var renderGraphParameters = new RenderGraphParameters
		{ … };
		using (renderGraph.RecordAndExecute(renderGraphParameters))
		{
			SetupPass.Record(renderGraph, this);
			…
		}
```

但为了保证一切运行正常，我们仍需确定是否在记录<font color="red">render graph.</font>之前使用<font color=#66ff66>intermediate buffer </font>。

```C#
	public void Render(…)
	{
		…

		useIntermediateBuffer = useScaledRendering ||
			useColorTexture || useDepthTexture || postFXStack.IsActive;

		var renderGraphParameters = new RenderGraphParameters
		{ … };
		…
	}

	…

	public void Setup()
	{
		context.SetupCameraProperties(camera);
		CameraClearFlags flags = camera.clearFlags;

		//useIntermediateBuffer = useScaledRendering ||
		//	useColorTexture || useDepthTexture || postFXStack.IsActive;
		…
	}
```

此外，我们还可以延迟设置<font color="red">buffer size vector</font>，直到<font color=#66ff66>Setup</font>结束。这样，除了采样之外，<font color=#66ff66>Render</font>本身就不再使用<font color=#66ff66> command buffer</font>的功能了。<font color="DarkOrchid ">取样除外。</font>

```C#
	public void Render(…)
	{
		…

		buffer.BeginSample(SampleName);
		//buffer.SetGlobalVector(bufferSizeId, new Vector4(
		//	1f / bufferSize.x, 1f / bufferSize.y, bufferSize.x, bufferSize.y));
		ExecuteBuffer();

		…
	}

	…

	public void Setup()
	{
		…
		buffer.SetGlobalVector(bufferSizeId, new Vector4(
			1f / bufferSize.x, 1f / bufferSize.y, bufferSize.x, bufferSize.y));
		ExecuteBuffer();
	}
```

#### 2.7 Lighting Pass

创建的最后一个<font color=#66ff66>pass </font>是第一个要执行的<font color=#66ff66>pass </font>，也就是光照。它会调用灯光对象上的设置，如果需要，还会渲染阴影。

```
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;

public class LightingPass
{
	Lighting lighting;

	CullingResults cullingResults;

	ShadowSettings shadowSettings;

	bool useLightsPerObject;

	int renderingLayerMask;

	void Render(RenderGraphContext context) => lighting.Setup(
		context.renderContext, cullingResults, shadowSettings,
		useLightsPerObject, renderingLayerMask);

	public static void Record(
		RenderGraph renderGraph, Lighting lighting,
		CullingResults cullingResults, ShadowSettings shadowSettings,
		bool useLightsPerObject, int renderingLayerMask)
	{
		using RenderGraphBuilder builder =
			renderGraph.AddRenderPass("Lighting", out LightingPass pass);
		pass.lighting = lighting;
		pass.cullingResults = cullingResults;
		pass.shadowSettings = shadowSettings;
		pass.useLightsPerObject = useLightsPerObject;
		pass.renderingLayerMask = renderingLayerMask;
		builder.SetRenderFunc<LightingPass>((pass, context) => pass.Render(context));
	}
}
```

将其拉入<font color=#FFCE70>graph</font>中，这样就会在设置后期特效堆栈并验证 FXAA 后出现，这样就没问题了。

```C#
		//lighting.Setup(
		//	context, cullingResults, shadowSettings, useLightsPerObject,
		//	cameraSettings.maskLights ? cameraSettings.renderingLayerMask : -1);

		bufferSettings.fxaa.enabled &= cameraSettings.allowFXAA;
		postFXStack.Setup(…);
		buffer.EndSample(SampleName);

		var renderGraphParameters = new RenderGraphParameters
		{ … };
		using (renderGraph.RecordAndExecute(renderGraphParameters))
		{
			LightingPass.Record(
				renderGraph, lighting,
				cullingResults, shadowSettings, useLightsPerObject,
				cameraSettings.maskLights ? cameraSettings.renderingLayerMask : -1);
			…
		}
```

这些是我们在 2.0.0 版本中包含的<font color="red">PAss</font>。现在，所有<font color="red">command buffer</font>的执行和渲染都在<font color=#66ff66>render graph</font>内进行。

### 3 Profiling

现在，我们已经有了一个具有多个<font color="red">passes </font>的<font color=#66ff66>render graph</font>，可以通过 Window（窗口）/ Analysis（分析）/ Render Graph Viewer（渲染图查看器）对其进行检查。

当前图形应设置为自定义 SRP 渲染图形，当前执行的唯一选项是 "渲染摄像机"。如果没有显示任何内容，可以通过<font color=#66ff66>Capture Graph</font>触发可视化。显示哪些<font color="red">passes</font>取决于<font color=#66ff66>graph</font>中包含哪些<font color="red">passes</font>。<font color="DarkOrchid ">Gizmos, post FX, and the final pass</font>都可能丢失。

![img](https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/render-graph-viewer.png)

<center>*Render graph viewer.*</center>

这将为我们提供<font color=#66ff66>render graph</font>的概览，以及哪些<font color="red">passes</font>使用了哪些资源，但我们在此版本中尚未使用该功能，因此它只能显示通道本身。请注意，当<font color="red"> render graph</font>查看器窗口打开时，即使不可见，<font color="red"> render graph</font>也会生成<font color="DarkOrchid ">debug information</font>，从而导致分配。

除了<font color="red"> render graph</font>查看器，<font color="DarkOrchid ">profiler </font>和<font color="DarkOrchid ">frame debugger </font>也非常有用，但我们目前尚未正确使用<font color=#66ff66> render graph </font>，因此这些数据并不完整。

#### 3.1 Using the Pooled Command Buffer

我们的想法是，<font color="red"> render graph</font>使用我们给它的<font color=#4db8ff>pooled command buffer</font>来执行。它会为自己和所有<font color="red">passes </font>添加样本到<font color=#4db8ff>command buffer </font>，但由于我们从未执行过它，所以还看不到这些<font color="DarkOrchid ">samples</font>。为了使其正常工作，我们的所有传递都应使用通过<font color=#FFCE70>render graph context.</font>>提供的<font color=#4db8ff>command buffer </font>。

我们从 <font color="red">CameraRenderer </font>本身开始，移除其<font color="DarkOrchid "> buffer name</font>并保持其缓冲区未初始化。

```C#
	//const string bufferName = "Render Camera";

	…

	//readonly CommandBuffer buffer = new()
	//{
	//	name = bufferName
	//};
	CommandBuffer buffer;
```

然后在<font color="red">Render</font>中将buffer 设置为与<font color="red"> render graph</font>相同的buffer 。这在技术上是不正确的，因为<font color="red"> render graph</font>内部可以使用多个buffer ，但只有在使用<font color="DarkOrchid ">asynchronous </font>异步传递时才会出现这种情况，而我们并没有使用异步传递。

```C#
	var renderGraphParameters = new RenderGraphParameters
		{
			commandBuffer = CommandBufferPool.Get(),
			currentFrameIndex = Time.frameCount,
			executionName = "Render Camera",
			scriptableRenderContext = context
		};
		buffer = renderGraphParameters.commandBuffer;
```

Next, delete <font color=#66ff66>PrepareBuffer</font> and <font color=#4db8ff>SampleName</font>.

```C#
//partial void PrepareBuffer();

#if UNITY_EDITOR

    …

    //string SampleName
    //{ get; set; }

    …

    //partial void PrepareBuffer() { … }

    //#else

    //const string SampleName = bufferName;

    #endif
```

然后删除所有依赖于它们的代码。这意味着我们也可以删除 <font color=#66ff66>Render </font>中对 <font color="red">ExecuteBuffer </font>的初始调用。为了保持框架调试器布局的整洁，我们在这里使用了分割示例作用域。

```C#
public void Render(…)
{
    …
        //PrepareBuffer();
        …

        //buffer.BeginSample(SampleName);
        //ExecuteBuffer();

        bufferSettings.fxaa.enabled &= cameraSettings.allowFXAA;
    postFXStack.Setup(…);
    //buffer.EndSample(SampleName);

    …
}

…

    public void Setup()
{
    …
        buffer.ClearRenderTarget(…);
    //buffer.BeginSample(SampleName);
    …
}

…

    void Submit()
{
    //buffer.EndSample(SampleName);
    ExecuteBuffer();
    context.Submit();
}
```

更改后，我们不再获取<font color="DarkOrchid ">editor-only memory allocations</font>，因为我们不再检索摄像机名称。如果你确实看到了分配，那是因为<font color="red">render graph viewer window</font>打开了。此外，<font color="DarkOrchid ">profiler </font>也不再显示摄像机，阴影也会显示在<font color=#66ff66> render graph</font>之外。

![img](https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/profiler-execute-render-graph.png)

<center>*Profiler showing execution of render graph.*</center>

The frame debugger hierarchy has also become messy.（杂乱无章）

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/frame-debugger-execute-render-graph.png" alt="img" style="zoom:50%;" />

<center>*Frame debugger showing execution of render graph.*</center>

#### 3.2 Lighting and Shadows

主要问题在于，光照和阴影各有自己的<font color=#4db8ff>command buffer</font>，因此无法对采样范围进行适当嵌套。为了解决这个问题，我们将让 <font color="red">LightingPass </font>把<font color=#66ff66>render graph context </font>转发到<font color=#4db8ff> Lighting.Setup</font>，而不是 scriptable render context.。

```C#
	void Render(RenderGraphContext context) => lighting.Setup(
		context, cullingResults, shadowSettings,
		useLightsPerObject, renderingLayerMask);
```

<font color=#66ff66>**Lighting**</font>必须使用该context中的 command buffer，而不是自己的缓冲区，并且不再自行采样。

```C#
//const string bufferName = "Lighting";

	…

	//readonly CommandBuffer buffer = new()
	//{
	//	name = bufferName
	//};
	CommandBuffer buffer;

	…

	public void Setup(
		RenderGraphContext context, CullingResults cullingResults,
		ShadowSettings shadowSettings, bool useLightsPerObject, int renderingLayerMask)
	{
		buffer = context.cmd;
		this.cullingResults = cullingResults;
		//buffer.BeginSample(bufferName);
		shadows.Setup(context, cullingResults, shadowSettings);
		SetupLights(useLightsPerObject, renderingLayerMask);
		shadows.Render();
		//buffer.EndSample(bufferName);
		context.renderContext.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

对阴影进行同样处理。在这种情况下，我们可以为定向阴影和其他阴影保留特定的样本，并为它们赋予适当的名称。

#### 3.3 Post FX

我们的 <font color="red">PostFXStack </font>也有自己的command buffer，我们必须用<font color=#FFCE70>render graph</font>中的command buffer来替换它。在这种情况下，我们可以将<font color=#FFCE70>render graph context</font>作为第一个参数添加到<font color="red">Render</font>方法中，并从 <font color=#66ff66>Setup </font>中移除可脚本化的<font color=#66ff66> render context</font>参数。

```C#
	//const string bufferName = "Post FX";

	…

	//readonly CommandBuffer buffer = new()
	//{
	//	name = bufferName
	//};
	CommandBuffer buffer;

	//ScriptableRenderContext context;

	…

	public void Setup(
		//ScriptableRenderContext context,
		…)
	{
		…
		//this.context = context;
		…
	}

	public void Render(RenderGraphContext context, int sourceId)
	{
		buffer = context.cmd;
		…
		context.renderContext.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

Provide the new argument for Render in <font color="red">PostFXPass</font>.

```C#
	void Render(RenderGraphContext context) =>
		postFXStack.Render(context, CameraRenderer.colorAttachmentId);
```

删除 <font color="red">CameraRenderer </font>中不再需要的 Setup 参数。

```C#
		postFXStack.Setup(
			//context,
			…);
```

#### 3.4 Visible Geometry

现在一切都使用相同的<font color=#FFCE70>command buffer</font>，<font color="DarkOrchid ">frame debugger</font>的层次结构看起来好多了，但还没有完全修复。不透明几何体和天空盒的渲染仍在设置范围内，这是不正确的。

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/frame-debugger-better.png" alt="img" style="zoom:50%;" />

<center>Frame debugger, better.</center>

出现这种情况是因为<font color=#66ff66> render graph</font>在command buffer上调用了 <font color=#FFCE70>BeginSample 和 EndSample</font>，但并没有 execute  buffer。

因此，<font color=#FFCE70>Pass</font>不会从<font color="DarkOrchid ">empty buffer</font>开始，它包含了采样指令和前一次<font color="red">pass </font>未清除的内容。这就是为什么在graph 完成后，我们仍要自己<font color=#4db8ff>execute  buffer</font>的原因。

我们通过在 <font color="red">DrawVisibleGeometry </font>开始时执行并清除<font color=#4db8ff>command buffer</font>来修复<font color=#66ff66>sample hierarchy</font>。

```C#
	public void DrawVisibleGeometry(…)
	{
		ExecuteBuffer();
		
		…
	}
```

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/frame-debugger-good.png" alt="img" style="zoom:50%;" />

<center>Frame debugger, good.</center>

#### 3.5 Custom Samplers

<font color=#FFCE70>render graph</font>使用 UnityEngine.Rendering.<font color=#66ff66>ProfilingSampler </font>实例进行<font color="DarkOrchid ">profiling</font>。每个实例内部都有两个采样器，一个用于通道执行，另一个--内联采样器--用于记录<font color="red">pass</font>。

如果我们不提供一个显式采样器作为 <font color="red">AddRenderPass </font>的第三个参数，那么<font color=#FFCE70>render graph</font>将根据<font color=#4db8ff>pass </font>名称创建一个采样器，并将其汇集到一个字典中。虽然这种方法可行，但效率很低

因为每次添加<font color=#4db8ff>pass </font>时都需要计算名称的<font color="DarkOrchid ">hash code</font>。因此，让我们为每个<font color=#4db8ff>pass </font>创建自己的显式采样器。首先是 <font color=#66ff66>SetupPass</font>。

```C#
static readonly ProfilingSampler sampler = new("Setup");

	CameraRenderer renderer;

	void Render(RenderGraphContext context) => renderer.Setup();

	public static void Record(
		RenderGraph renderGraph, CameraRenderer renderer)
	{
		using RenderGraphBuilder builder =
			renderGraph.AddRenderPass(sampler.name, out SetupPass pass, sampler);
		pass.renderer = renderer;
		builder.SetRenderFunc<SetupPass>((pass, context) => pass.Render(context));
	}
}
```

然后对所有其他<font color="red">Pass</font>进行同样的操作。

#### 3.6 Multiple Cameras

只要我们只有一台摄像机，一切看起来都运行正常。但让我们打开多摄像头场景。该场景渲染了六个相邻和相叠的摄像头，其中一个摄像头渲染的纹理通过<font color="red"> UI canvas</font>显示。

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/multiple-cameras-scene.png" alt="img" style="zoom:50%;" />

<center>Multiple Cameras scene.</center>

事实证明，在<font color="DarkOrchid "> profiler and frame</font>中，所有内容都被归类到 <font color="red">ExecuteRenderGraph </font>下。此外，在渲染图查看器中只能选择一个渲染摄像机。

为了能看到每个摄像机的渲染图，我们必须给它们取一个不同的<font color=#66ff66>execution </font>名称。为此，让我们使用摄像机的名称。

```C#
	var renderGraphParameters = new RenderGraphParameters
		{
			commandBuffer = CommandBufferPool.Get(),
			currentFrameIndex = Time.frameCount,
			executionName = camera.name,
			scriptableRenderContext = context
		};
```

这样做是可行的，但我们要再次为每一帧分配内存，而且这只会对<font color=#66ff66>viewer  </font>产生影响，而<font color="DarkOrchid ">profiler and frame</font>则不受影响。

要解决后者的问题，我们可以在<font color=#FFCE70> render graph</font>的顶层添加一个<font color="DarkOrchid "> extra profiling scope</font>

方法是在第一个记录步骤中创建一个新的 <font color="red">RenderGraphProfilingScope</font>，将<font color=#4db8ff> graph and a profiling sampler</font>（我们使用摄像机名称）传递给它。这是一个可处置程序，可添加开始和结束采样的额外<font color=#FFCE70>passes </font>。

```c#
	var cameraSampler = new ProfilingSampler(camera.name);
		var renderGraphParameters = new RenderGraphParameters
		{
			commandBuffer = CommandBufferPool.Get(),
			currentFrameIndex = Time.frameCount,
			executionName = cameraSampler.name,
			scriptableRenderContext = context
		};
		buffer = renderGraphParameters.commandBuffer;
		using (renderGraph.RecordAndExecute(renderGraphParameters))
		{
			using var _ = new RenderGraphProfilingScope(renderGraph, cameraSampler);
			…
		}
```

>   <center>What does `_` do?</center>
>
>   没有显式作用域的 <font color="red">using</font> 方法需要声明一个变量。由于我们不会在任何地方使用该变量，我们可以给它起一个特殊的下划线名称，即<font color="red">丢弃变量</font>，表示我们自己不需要访问它。

<font color="red">profiler </font>现在可以正确地对每台摄像机的作品进行分组。

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/profiler-multiple-cameras.png" alt="img" style="zoom:50%;" />

<center>Profiler showing multiple cameras.</center>

And the frame debugger does so as well.

<img src="https://catlikecoding.com/unity/custom-srp/2-0-0/profiling/frame-debugger-multiple-cameras.png" alt="img" style="zoom:50%;" />

<center>Frame debugger showing multiple cameras.</center>

最后，为了再次消除分配，我们必须缓存<font color=#FFCE70>camera name and the profiling sampler</font>。由于这是针对每台摄像机的，我们唯一能有效做到这一点的地方就是 <font color=#66ff66>CustomRenderPipelineCamera</font>（自定义渲染管道摄像机）。通过采样器获取属性来获取采样器，就像其设置一样。

```C#
using UnityEngine;
using UnityEngine.Rendering;

[DisallowMultipleComponent, RequireComponent(typeof(Camera))]
public class CustomRenderPipelineCamera : MonoBehaviour
{
	[SerializeField]
	CameraSettings settings = default;

	ProfilingSampler sampler;

	public ProfilingSampler Sampler => sampler ??= new(GetComponent<Camera>().name);

	public CameraSettings Settings => settings ??= new();
}
```

现在我们可以在 CameraRenderer.<font color=#66ff66>Render </font> 中检索采样器（如果相机有我们的组件）。

如果没有，我们将通过 <font color="red">ProfilingSampler.Get </font>使用基于相机类型的通用采样器。这是一种通用方法，可根据枚举类型将采样器汇集到字典中。这并不像使用字符串那么糟糕，因为枚举的哈希值要简单得多。

```C#
	//var crpCamera = camera.GetComponent<CustomRenderPipelineCamera>();
		//CameraSettings cameraSettings =
		//	crpCamera ? crpCamera.Settings : defaultCameraSettings;
		ProfilingSampler cameraSampler;
		CameraSettings cameraSettings;
		if (camera.TryGetComponent(out CustomRenderPipelineCamera crpCamera))
		{
			cameraSampler = crpCamera.Sampler;
			cameraSettings = crpCamera.Settings;
		}
		else
		{
			cameraSampler = ProfilingSampler.Get(camera.cameraType);
			cameraSettings = defaultCameraSettings;
		}

		…

		//var cameraSampler = new ProfilingSampler(camera.name);
```

这种方法有两个缺点

首先，没有使用 <font color=#66ff66>CustomRenderPipelineCamera </font>的摄像头会有一个通用名称，通常是游戏摄像头的 Game，但你也会看到<font color="DarkOrchid ">view, reflection, and preview camera </font>显示出来。

这些都不会显示在<font color="DarkOrchid ">frame debugger</font>中，但你可以在<font color="red">render graph viewer</font>中找到它们，如果处于编辑模式，还可以在<font color=#FFCE70>profiler</font>中找到它们。

第二个缺点是，由于我们对采样器进行了缓存，摄像机名称的更改不会被捕获。只有加载一个场景才会捕捉到变化，要么是明确打开一个场景，要么是重新加载域引起的变化，因为采样器会丢失并重新创建。

URP 也有同样的问题。我们可以在启用组件时取消采样器，从而使情况稍有好转。这样，在进入播放模式时，即使禁用了域和场景重载，如果在播放模式下启用了摄像机，名称也会更新。只有在编辑器或开发构建中才需要这样做。

```C#
#if UNITY_EDITOR || DEVELOPMENT_BUILD
	void OnEnable() => sampler = null;
#endif
```

为实现 2.0.0 版本而进行的修改到此为止。今后的工作将包括清理代码和使用更多渲染图功能。

想知道下一个教程的发布时间吗？请关注我的 Patreon 页面！
