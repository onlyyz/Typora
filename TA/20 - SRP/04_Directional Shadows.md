

# 04_ [Directional Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/)

-   Render and sample shadow maps.
-   Support multiple shadowed directional lights.
-   Use cascaded shadow maps.
-   Blend, fade, and filter shadows.

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/tutorial-image.jpg)

<center>防止光线到达不应到达的地方。</center>

## 1、Rendering Shadows

绘制某些东西时， <font color="blue">surface and light information is enough to calculating lighting</font>.。但在这两者之间可能有一些东西阻挡了光线，在我们所画的表面投下了阴影。

为了使阴影成为可能，我们必须以某种方式让着色器知道阴影投射对象<font color="RoyalBlue">**shadow-casting objects**</font>。有多种技术可以做到这一点。最常见的方法是生成一个阴影贴图，存储光线在到达表面之前可以远离光源的距离。同一个方向上任何更远的地方都不能被同一盏灯照亮。<font color="green">Unity's RPs</font>使用这种方法，我们也将如此。

#### [1.1、Shadow Settings](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.1)

在我们开始渲染阴影之前，我们首先要做一些关于质量的决定，具体到我们要渲染多远的阴影以及我们的阴影贴图要有多大。

虽然我们可以在摄像机能看到的范围内渲染阴影，但这需要大量的drawing 和一个非常大的map来充分覆盖该区域，这几乎是不现实的。

所以我们将为阴影引入一个最大距离，最小值为0，默认设置为100单位。创建一个新的可序列化的<font color="green">**ShadowSettings**</font>类来包含这个选项。这个类纯粹是一个配置选项的容器，所以我们将给它一个公共的<font color="blue">**maxDistance field**</font>.

```c#
using UnityEngine;

[System.Serializable]
public class ShadowSettings {

	[Min(0f)]
	public float maxDistance = 100f;
}
```

对于贴图大小，我们将引入一个嵌套在<font color="green">ShadowSettings</font>中的<font color="blue">TextureSize（enum type）</font>枚举类型。用它来定义允许的纹理大小，所有的纹理都是256-8192范围内的2次方。

```c#
	public enum TextureSize {
		_256 = 256, _512 = 512, _1024 = 1024,
		_2048 = 2048, _4096 = 4096, _8192 = 8192
	}
```

然后为<font color="red"> shadow map</font>添加一个 size field，将1024作为默认值。

我们将使用一个纹理来包含多个阴影贴图，所以将其命名为<font color="red">**atlasSize**</font>。由于我们现在只支持方向性的灯光，所以在这一点上我们也只使用方向性的阴影贴图。但是我们在未来会支持其他的灯光类型，这些灯光会有自己的阴影设置。

把<font color="red">**atlasSize**</font>放在一个内部的<font color="red">**Directional**</font>结构中。这样我们就能在检查器中自动获得一个分层的配置。

```
[System.Serializable]
	public struct Directional {

		public TextureSize atlasSize;
	}

	public Directional directional = new Directional {
		atlasSize = TextureSize._1024
	};
```

为<font color="red">CustomRenderPipelineAsset</font>添加一个阴影设置的字段。

```c#
	[SerializeField]
	ShadowSettings shadows = default;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/shadow-settings.png" alt="img" style="zoom:50%;" />

<center>Shadow settings.</center>

当<font color="blue">CustomRenderPipeline</font>实例被构建时，将这些设置传递给它。

```c#
	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			useDynamicBatching, useGPUInstancing, useSRPBatcher, shadows
		);
	}
```

<font color="red">**并使其保持对它们的跟踪**</font>。

```c#
	ShadowSettings shadowSettings;

	public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		ShadowSettings shadowSettings
	) {
		this.shadowSettings = shadowSettings;
		…
	}
```

#### [1.2、Passing Along Settings](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.2)

从现在开始，我们将在调用<font color="red">**camera renderer** </font>的<font color="red">**Render**</font>方法时将这些设置传递给它。这样一来，就很容易在运行时添加对改变阴影设置的支持，但在本教程中我们不会处理这个问题。

<font color="RoyalBlue">C# at CustomRenderPipeline</font>

```c#
protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera, useDynamicBatching, useGPUInstancing,
				shadowSettings
			);
		}
	}
```

<font color="red">CameraRenderer.Render</font>然后将它传递<font color="RoyalBlue">**Lighting.Setup**</font>给它自己<font color="red">**Cull**</font>方法。

```c#
	public void Render (
		ScriptableRenderContext context, Camera camera,
		bool useDynamicBatching, bool useGPUInstancing,
		ShadowSettings shadowSettings
	) {
		…
		if (!Cull(shadowSettings.maxDistance)) {
			return;
		}

		Setup();
		lighting.Setup(context, cullingResults, shadowSettings);
		…
	}
```

我们需要设置<font color="red">Cull</font>，因为<font color="red">**shadow distance**</font>是通过<font color="RoyalBlue">Culling parameters</font>设置的。

```c#
bool Cull (float maxShadowDistance) {
		if (camera.TryGetCullingParameters(out ScriptableCullingParameters p)) {
			p.shadowDistance = maxShadowDistance;
			cullingResults = context.Cull(ref p);
			return true;
		}
		return false;
	}
```

<font color="green">渲染比摄像机所能看到的更远的阴影是没有意义的</font>，所以要取 <font color="red">**minimum of max shadow distance and the camera's far clip plane**</font>

```
	p.shadowDistance = Mathf.Min(maxShadowDistance, camera.farClipPlane);
```

为了使代码能够编译，我们还必须为<font color="green">**Lighting.Setup**</font>添加一个阴影设置的参数，但我们现在还不会对它们做任何处理。

```c#
	public void Setup (
		ScriptableRenderContext context, CullingResults cullingResults,
		ShadowSettings shadowSettings
	) { … }
```

#### [1.3、Shadows Class](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.3)

虽然阴影在逻辑上是光照的一部分，但它们相当复杂，所以让我们创建一个新的 <font color="green">Shadows class</font>来专门处理它们。它开始时是<font color="green"> a stripped-down stub copy of Lighting说</font>，有自己 buffer, fields for the context, culling results, and settings,  一个初始化字段的<font color="blue">Setup</font>方法和一个<font color="blue">ExecuteBuffer</font>方法。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class Shadows {

	const string bufferName = "Shadows";

	CommandBuffer buffer = new CommandBuffer {
		name = bufferName
	};

	ScriptableRenderContext context;

	CullingResults cullingResults;

	ShadowSettings settings;

	public void Setup (
		ScriptableRenderContext context, CullingResults cullingResults,
		ShadowSettings settings
	) {
		this.context = context;
		this.cullingResults = cullingResults;
		this.settings = settings;
	}

	void ExecuteBuffer () {
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
}
```

<font color="red">那么<font color="green">Lighting</font>需要做的就是跟踪<font color="green">Shadows实例</font></font>，并在自己的**Setup**方法中的<font color="green">SetupLights</font>之前调用其<font color="green">Setup</font>方法。

```c#
	Shadows shadows = new Shadows();

	public void Setup (…) {
		this.cullingResults = cullingResults;
		buffer.BeginSample(bufferName);
		shadows.Setup(context, cullingResults, shadowSettings);
		SetupLights();
		…
	}
```

#### [1.4、Lights with Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.4)

由于渲染阴影需要额外的工作，它可能会降低帧率，所以我们将限制有多少阴影的定向灯，与支持多少个定向灯无关。在<font color="red">**Shadows**</font>中添加一个常量，最初设置为只有一个。

```c
const int maxShadowedDirectionalLightCount = 1;
```

我们不知道哪个可见光会得到阴影，所以我们必须跟踪这个。除此以外，我们还将在以后跟踪每个阴影灯的一些数据，所以让我们定义一个内部的<font color="red">**ShadowedDirectionalLight**</font>结构，它暂时只包含索引，并跟踪这些数组。

```c#
	struct ShadowedDirectionalLight {
		public int visibleLightIndex;
	}

	ShadowedDirectionalLight[] ShadowedDirectionalLights =
		new ShadowedDirectionalLight[maxShadowedDirectionalLightCount];
```

<font color="red">**为了弄清楚哪个光能得到阴影**</font>，我们将添加一个公共的<font color="green">**ReserveDirectionalShadows**</font>方法，它有一个光和可见光的<font color="blue">index parameters</font>。

它的工作是在阴影图，集中为灯光的<font color="RoyalBlue">shadow atlas</font>保留空间，并存储渲染它们所需的信息。

```c#
public void ReserveDirectionalShadows (Light light, int visibleLightIndex) {}
```

<font color="red">**由于阴影灯的数量是有限的，我们必须跟踪有多少已经被保留**</font>。

在<font color="green">Setup</font>中把计数重置为零。然后在<font color="green">ReserveDirectionalShadows</font>中检查我们是否还没有达到最大值。如果还有空间，那么就存储灯光的可见索引并增加<font color="green">count</font>。

```c#
	int ShadowedDirectionalLightCount;

	…
	
	public void Setup (…) {
		…
		ShadowedDirectionalLightCount = 0;
	}
	
	public void ReserveDirectionalShadows (Light light, int visibleLightIndex) {
		if (ShadowedDirectionalLightCount < maxShadowedDirectionalLightCount) {
			ShadowedDirectionalLights[ShadowedDirectionalLightCount++] =
				new ShadowedDirectionalLight {
					visibleLightIndex = visibleLightIndex
				};
		}
	}
```

但是阴影应该只保留给有阴影的灯。如果<font color="green">光的阴影模式设置为无或其阴影强度为零，则它没有阴影，应该被忽略</font>。

```c#
if (
	ShadowedDirectionalLightCount < maxShadowedDirectionalLightCount &&
	light.shadows != LightShadows.None && light.shadowStrength > 0f
		) { … }
```

除此之外，有可能可见光最终没有影响任何投射阴影的物体，要么是因为它们被配置为<font color="red"> not affecting cast shadows</font>，要么是因为该光只影响最大阴影距离以外的物体。

我们可以通过调用<font color="red">**GetShadowCasterBounds**</font>对一个可见光索引的剔除结果进行检查。它有第二个<font color="blue">output parameter </font>，是关于边界的<font color="blue">--which we don't need—and returns </font>是否有效。

<font color="red">**如果不是，那么这个灯就没有阴影需要渲染，它应该被忽略**</font>。

```c#
		if (
			ShadowedDirectionalLightCount < maxShadowedDirectionalLightCount &&
			light.shadows != LightShadows.None && light.shadowStrength > 0f &&
			cullingResults.GetShadowCasterBounds(visibleLightIndex, out Bounds b)
		) { … }
```

>   [CullingResults](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.html).[GetShadowCasterBounds](https://docs.unity3d.com/ScriptReference/30_search.html?q=GetShadowCasterBounds) 返回封装可见阴影投射器的边界框。例如，可用于动态调整级联...
>
>    **bool** 如果光线影响场景中的至少一个阴影投射对象，则为真。
>

现在我们可以在<font color="RoyalBlue">Lighting.SetupDirectionalLight</font>中保留阴影。

```
	void SetupDirectionalLight (int index, ref VisibleLight visibleLight) {
		dirLightColors[index] = visibleLight.finalColor;
		dirLightDirections[index] = -visibleLight.localToWorldMatrix.GetColumn(2);
		shadows.ReserveDirectionalShadows(visibleLight.light, index);
	}
```

#### [1.5、Creating the Shadow Atlas](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.5)

<font color="red">**保存阴影之后我们需要渲染他们**</font>，我们在<font color="red">**Lighting.Render.SetupLights**</font>完成后，通过调用一个新的<font color="blue">Shadows.Render</font>方法来完成。

```c#
	shadows.Setup(context, cullingResults, shadowSettings);
		SetupLights();
		shadows.Render();
```

该<font color="red">Shadows.Render</font>方法会将<font color="green">rendering of directional shadows</font>委托给另一个<font color="blue">**RenderDirectionalShadows方法**</font>，<font color="red">**但前提是存在任何阴影光**</font>

```c#
	public void Render () {
		if (ShadowedDirectionalLightCount > 0) {
			RenderDirectionalShadows();
		}
	}

	void RenderDirectionalShadows () {}
```

创建<font color="green">**shadow map**</font> 是通过在纹理上绘制<font color="blue">shadow-casting 对象</font>来完成的。

我们将使用<font color="red">**_DirectionalShadowAtlas**</font>来指代 <font color="green">directional shadow **atlas**</font>。从设置中获取图集大小的整数，然后在 <font color="RoyalBlue">command buffer</font>上调用<font color="RoyalBlue">GetTemporaryRT</font>，将<font color="green"> texture identifier</font> 作为参数，加上其宽度和高度的大小（以像素为单位）。

```c#
static int dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas");
	
	…
	
	void RenderDirectionalShadows () {
		int atlasSize = (int)settings.directional.atlasSize;
		buffer.GetTemporaryRT(dirShadowAtlasId, atlasSize, atlasSize);
	}
```

>   [GetTemporaryRT](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.GetTemporaryRT.html) 添加获取临时渲染纹理命令。

这要求一个正方形的<font color="green"> render texture</font>，但它默认是一个正常的ARGB纹理。我们需要的是<font color="green">shadow map</font>，我们通过在调用中添加另外三个参数来指定它。

首先是<font color="RoyalBlue">depth buffer</font>的位数。我们希望这个位数越高越好，所以我们使用32。

第二是<font color="RoyalBlue">filter mode</font>,我们使用默认的<font color="blue">bilinear filtering</font>。

第三是<font color="RoyalBlue"> render texture type</font>，它必须是<font color="blue">RenderTextureFormat.Shadowmap。</font>这给了我们一个适合渲染阴影贴图的纹理，尽管具体格式取决于目标平台。

```c#
buffer.GetTemporaryRT(
			dirShadowAtlasId, atlasSize, atlasSize,
			32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap
		);
```

>   我们得到了什么样的纹理格式?
>   它通常是一个24或32位的整数或浮点纹理。你也可以选择16位，这也是Unity的rp所做的。

当我们得到一个<font color="red">**temporary render texture**</font> 时，我们也应该在用完后<font color="green">release  it </font>。

我们必须保持对它的控制，直到我们完成了相机的渲染，<font color="red">after which we can release it by invoking <font color="green">ReleaseTemporaryRTwith</font> the texture identifier of the buffer and then execute it</font> ，然后执行它。

我们将在一个新的公共Cleanup方法中做到这一点。我们将在一个新的公共<font color="green">Cleanup</font>方法中执行此操作。

```c#
public void Cleanup () {
		buffer.ReleaseTemporaryRT(dirShadowAtlasId);
		ExecuteBuffer();
	}
```

>   [ExecuteCommandBuffer](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.ExecuteCommandBuffer.html) 安排自定义图形命令缓冲区的执行。
>
>   [ReleaseTemporaryRT  ](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.ReleaseTemporaryRT.html)添加释放临时渲染纹理命令。

也给<font color="red">**Lighting**</font>一个公共的<font color="blue">Cleanup</font>方法，它将调用转给<font color="blue">Shadows。</font>

```c#
	public void Cleanup () {
		shadows.Cleanup();
	}
```

<font color="red">然后<font color="green">**CameraRenderer**</font>可以在提交前直接请求cleanup。</font>

```c#
public void Render (…) {
		…
		lighting.Cleanup();
		Submit();
	}
```

我们只有在先<font color="red"> claimed texture（声明纹理）</font> 的情况下才能释放它，目前我们只在<font color="blue">directional shadows to render</font>时才这样做。显而易见的解决方案是，只有在有阴影时才释放纹理。

然而，<font color="red">**不声称一个纹理将导致WebGL 2.0的问题**</font>，因为它将 <font color="green">textures</font>和 <font color="green">samplers 采样器</font>绑定在一起。

当一个带有我们的<font color="green">shader</font>的材质在纹理缺失的情况下被加载时，它将会失败，<font color="red">**因为它将会得到默认的纹理，而这个纹理将不会与 shadow sampler兼容。**</font>

我们可以通过引入<font color="RoyalBlue">Shader keyword</font>来避免这种情况，生成省略阴影采样代码的<font color="blue"> shader variants</font>。

另一种方法是在不需要阴影的时候得到一个1×1的<font color="green">假纹理（ dummy texture ）</font>，避免额外的<font color="blue"> shader variants</font>。让我们这样做吧。

```c#
	public void Render () {
		if (shadowedDirLightCount > 0) {
			RenderDirectionalShadows();
		}
		else {
			buffer.GetTemporaryRT(
				dirShadowAtlasId, 1, 1,
				32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap
			);
		}
	}
```

在请求<font color="green">render texture</font>后，<font color="red"><font color="green">Shadows.RenderDirectionalShadows</font>  渲染到此纹理而不是相机的目标</font>。这是通过在缓冲区上调用<font color="blue">SetRenderTarget</font>来完成的，确定一个渲染纹理以及它的数据应该如何加载和存储。

>   [SetRenderTarget](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.SetRenderTarget.html) 增加一个“set active render target”命令。

<font color="red">**我们并不关心它的初始状态，因为我们会立即清除它**</font> ，所以我们会使用RenderBufferLoadAction.<font color="green">DontCare</font>。而纹理的目的是为了包含阴影数据，所以我们需要使用RenderBufferStoreAction.<font color="green">Store</font>作为第三个参数。

```c#
buffer.GetTemporaryRT(…);
		buffer.SetRenderTarget(
			dirShadowAtlasId,
			RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
```

一旦完成，我们就可以像清除摄像机目标一样使用<font color="red">**ClearRenderTarget**</font> ，在这种情况下只关心 <font color="RoyalBlue">depth buffer</font>。

最后执行缓冲区。<font color="red">如果你至少有一个有阴影的定向光处于活动状态，那么你会看到阴影图集的清除动作在帧调试器中显示出来</font>。

```c#
	buffer.SetRenderTarget(
			dirShadowAtlasId,
			RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
		buffer.ClearRenderTarget(true, false, Color.clear);
		ExecuteBuffer();
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/clearing-two-render-targets.png" alt="img" style="zoom:50%;" />

<center>清除两个渲染目标。</center>

>   为什么我得到一个关于尺寸不匹配的错误?
>   这一点可以在Unity 2020中实现。请继续，它将在下一节中得到解决。

#### [1.6、Shadows First](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.6)

由于我们在<font color="red">阴影图集</font> 之前设置普通相机，我们<font color="green">最终会在渲染常规几何体之前切换到阴影图集</font>，这不是我们想要的。我们应该在<font color="RoyalBlue">**CameraRenderer**.Render</font>调用<font color="RoyalBlue">**CameraRenderer**.Setup</font>之前<font color="green">渲染阴影</font>，这样常规渲染就不会受到影响。

```c#
	//Setup();
		lighting.Setup(context, cullingResults, shadowSettings);
		Setup();
		DrawVisibleGeometry(useDynamicBatching, useGPUInstancing);
```

我们可以在<font color="blue"> frame debugger帧调试器</font>中保持阴影条目嵌套在摄像机的条目中，方法是在<font color="red">设置灯光之前开始采样，在设置灯光之后立即结束采样，然后再清除摄像机的目标。</font>

```glsl
		buffer.BeginSample(SampleName);
		ExecuteBuffer();
		lighting.Setup(context, cullingResults, shadowSettings);
		buffer.EndSample(SampleName);
		Setup();
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/nested-shadows.png" alt="img" style="zoom:50%;" />

<center>Nested shadows.</center>

#### [1.7、Rendering](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.7)

为了渲染单个灯光的阴影，我们将为<font color="red">**Shadow**</font>添加一个变体<font color="blue">**RenderDirectionalShadows**</font>方法，有两个参数：第一是<font color="green">shadowed light index</font>，第二是 <font color="green">size of its tile in the **atlas**</font>。

然后在另一个<font color="RoyalBlue">**RenderDirectionalShadows**</font>方法中为<font color="red">所有被阴影的灯光调用这个方法</font>，由<font color="blue">BeginSample</font>和<font color="blue">EndSample</font>调用来包装。

因为我们目前只支持一个阴影灯，它的<font color="green">tile size</font>等于 <font color="green">atlas size</font>。

```c#
void RenderDirectionalShadows () {
		…
		buffer.ClearRenderTarget(true, false, Color.clear);
		buffer.BeginSample(bufferName);
		ExecuteBuffer();

		for (int i = 0; i < ShadowedDirectionalLightCount; i++) {
			RenderDirectionalShadows(i, atlasSize);
		}
		
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}	

	void RenderDirectionalShadows (int index, int tileSize) {}
```

为了渲染阴影，我们需要一个<font color="green">**ShadowDrawingSettings**</font>结构值。我们可以通过调用它的构造方法来创建一个<font color="blue">正确配置的结构值</font>，其中包括我们要是用的灯之前存储的<font color="red">**culling results** (剔除结果)</font>	和适当的<font color="red">**visible light index**（可见光索引）</font>。

```c#
void RenderDirectionalShadows (int index, int tileSize) {
		ShadowedDirectionalLight light = ShadowedDirectionalLights[index];
		var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
	}
```

>    [`ShadowDrawingSettings` ](https://docs.unity3d.com/ScriptReference/Rendering.ShadowDrawingSettings.html)   该结构描述了使用何种分割设置 ( [splitData](https://docs.unity3d.com/ScriptReference/Rendering.ShadowDrawingSettings-splitData.html) )渲染哪个阴影光 ( [lightIndex](https://docs.unity3d.com/ScriptReference/Rendering.ShadowDrawingSettings-lightIndex.html) )。另请参阅：[ShadowSplitData](https://docs.unity3d.com/ScriptReference/Rendering.ShadowSplitData.html)。
>
>   [ShadowDrawingSettings Constructor](https://docs.unity3d.com/ScriptReference/Rendering.ShadowDrawingSettings-ctor.html) <font color="red">构造函数 Create a shadow settings object.</font>(创建阴影设置对象) 



**shadow map**概念是，我们从光线的角度渲染场景，只存储深度信息。其结果是告诉我们光线在击中某物之前走了多远。

然而，定向灯被认为是无限远的，因此没有一个真实的位置。所以我们要做的是找出与灯光的方向相匹配的<font color="green">view and projection matrices（视图和投影矩阵）</font> 并给我们一个 <font color="RoyalBlue">clip space cube（剪辑空间立方体）</font>，该立方体与摄像机可见的区域相重叠，可以包含光的阴影。与其自己想办法，不如使用<font color="red">ComputeDirectionalShadowMatricesAndCullingPrimitives</font> ，我们需要传给它九个参数。![image-20230326013003196](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230326013003196.png)

>   CullingResults.[ComputeDirectionalShadowMatricesAndCullingPrimitives](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives.html)
>

<font color="green">计算一个定向光的视图和投影矩阵以及阴影分割数据。</font>

<font color="RoyalBlue">fenbianl </font>比例、 <font color="RoyalBlue">Resolution</font> 分辨率、                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                

第一个参数是<font color="reen">visible light index可见光索引</font>。接下来的三个参数是两个整数和一个Vector3，它们控制阴影的级联。

我们稍后会处理级联问题，所以现在使用0、1和零矢量。之后是纹理大小，我们需要使用<font color="green">tile size瓦片大小</font>。第六个参数是<font color="green">shadow near plane阴影的近平面</font>，我们将忽略这个参数，暂时设置为零。

这些是输入参数

剩下的三个是<font color="red">输出参数</font>。首先是<font color="green">view matrix _ 视图矩阵</font>，然后是<font color="green">projection matrix_投影矩阵</font>，最后一个参数是<font color="green">ShadowSplitData结构</font>。

>   [ShadowSplitData](https://docs.unity3d.com/ScriptReference/Rendering.ShadowSplitData.html)  描述一个给定的阴影分割的剔除信息（例如，定向级联）。

```c#
	var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
		cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(
			light.visibleLightIndex, 0, 1, Vector3.zero, tileSize, 0f,
			out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix,
			out ShadowSplitData splitData
		);
```

<font color="red"> how shadow-casting objects should be culled（split data 包含了关于如何剔除投射阴影的物体的信息）</font>，我们必须将其复制到<font color="red"> shadow settings</font>中。

我们还必须通过调用缓冲区上的<font color="RoyalBlue">**SetViewProjectionMatrices**</font>来应用 <font color="green">view matrices</font>和<font color="green"> projection matrices。</font>

>   [CommandBuffer](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.html).SetViewProjectionMatrices 添加一个命令来设置视图和投影矩阵。

```c#
	cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(…);
		shadowSettings.splitData = splitData;
		buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
```

我们最后通过执行<font color="RoyalBlue">缓冲区</font>来安排<font color="red">shadow casters的绘制</font> ，然后在上下文中调用<font color="red">**DrawShadows**</font> ，<font color="green"> shadows settings</font>通过引用传递给它。

```c#
	shadowSettings.splitData = splitData;
		buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
		ExecuteBuffer();
		context.DrawShadows(ref shadowSettings);
```

>   [DrawShadows](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.DrawShadows.html) Schedules the drawing of shadow casters for a single Light.
>
>   为单一的灯安排绘制影子投射器。
>



#### [1.8、Shadow Caster Pass](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.8)

At this point shadow casters should get rendered，但<font color="green">atlas</font>仍然是空的。<font color="red">这是因为<font color="blue">**DrawShadows**</font>只渲染有<font color="green">**ShadowCaster Pass**</font>的材质的物体。</font>

因此，在我们的<font color="green">Lit shader</font>上添加第二个<font color="green">Pass</font>块，将其光照模式设置为<font color="green">ShadowCaster</font>。使用相同的目标层，让它支持实例化，加上<font color="blue">**_CLIPPING**</font>着色器功能。

然后让它使用特殊的<font color="green">Shadow-caster</font>函数，我们将在一个新的<font color="RoyalBlue">ShadowCasterPass.HLSL</font>文件中定义。另外，因为我们只需要写深度，所以禁止写颜色数据，在<font color="green">HLSL</font>程序前加入<font color="red">ColorMask 0</font>。

```glsl
	SubShader {
		Pass {
			Tags {
				"LightMode" = "CustomLit"
			}

			…
		}

		Pass {
			Tags {
				"LightMode" = "ShadowCaster"
			}

			ColorMask 0

			HLSLPROGRAM
			#pragma target 3.5
			#pragma shader_feature _CLIPPING
			#pragma multi_compile_instancing
			#pragma vertex ShadowCasterPassVertex
			#pragma fragment ShadowCasterPassFragment
			#include "ShadowCasterPass.hlsl"
			ENDHLSL
		}
	}
```

创建<font color="red">**ShadowCasterPass**</font>文件，复制LitPass并删除所有不需要的<font color="green">**shadow casters.**</font>的代码。

所以<font color="red">我们只需要 <font color="RoyalBlue">clip-space position</font>，加上<font color="RoyalBlue">clipping</font>基础颜色</font>。

<font color="green">**fragment function**</font>没有任何东西可以返回，所以变成了<font color="RoyalBlue">void</font>没有语义。它所做的唯一事情就是潜在地<font color="green">clip fragments</font>。

```glsl
#ifndef CUSTOM_SHADOW_CASTER_PASS_INCLUDED
#define CUSTOM_SHADOW_CASTER_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

struct Attributes {
	float3 positionOS : POSITION;
	float2 baseUV : TEXCOORD0;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct Varyings {
	float4 positionCS : SV_POSITION;
	float2 baseUV : VAR_BASE_UV;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings ShadowCasterPassVertex (Attributes input) {
	Varyings output;
	UNITY_SETUP_INSTANCE_ID(input);
	UNITY_TRANSFER_INSTANCE_ID(input, output);
	float3 positionWS = TransformObjectToWorld(input.positionOS);
	output.positionCS = TransformWorldToHClip(positionWS);

	float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseMap_ST);
	output.baseUV = input.baseUV * baseST.xy + baseST.zw;
	return output;
}

void ShadowCasterPassFragment (Varyings input) {
	UNITY_SETUP_INSTANCE_ID(input);
	float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
	float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
	float4 base = baseMap * baseColor;
	#if defined(_CLIPPING)
		clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
	#endif
}

#endif
```

我们现在可以<font color="red">**render shadow casters**</font>。创建了一个简单的测试场景，其中包含一些不透明的物体在一个平面上，有一个方向性的灯光，在全强度下启用了阴影，以尝试它。无论灯光被设置为使用硬阴影还是软阴影都没有关系。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/test-scene.png" alt="img" style="zoom: 50%;" />

<center>Shadow test scene.</center>

阴影还不影响<font color="red">rendered image </font>，但我们已经可以通过<font color="blue">frame debugger</font>看到被渲染到阴影图集中的东西了。它通常被看作是一个单色纹理，随着距离的增加由白变黑，但在使用OpenGL时，它是红色的，并向相反方向发展。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/distance-100.png" alt="img" style="zoom:50%;" />

<center>512 atlas; max distance 100.</center>

在最大阴影距离设置为100的情况下，我们最终会发现所有东西都只渲染到纹理的一小部分。减少最大距离可以有效地使阴影贴图放大到摄像机前面的部分。

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/distance-20.png" alt="20" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/distance-10.png" alt="10" style="zoom:50%;" /></center>

<center>Max distance 20 and 10.</center>

请注意，投影者是用<font color="red">**orthographic projection**</font> 来渲染的，因为我们是为定向光而渲染的。

#### [1.9、Multiple Lights](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#1.9)

我们最多可以有四个方向灯，所以我们也支持最多四个带阴影的方向灯。

```c#
	const int maxShadowedDirectionalLightCount = 4;
```

作为一项快速测试，我使用了四个等效的定向灯，只是我将它们的 Y 旋转角度调整了 90°。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/four-lights.png" alt="img" style="zoom:50%;" />

<center>Shadow casters for four lights, superimposed.（叠加）</center>

尽管我们最终正确地渲染了所有灯光的<font color="green">shadow casters</font>，<font color="red">但当我们为每个灯光渲染整个<font color="green"> atlas</font>时，它们是叠加的。<font color="blue">我们必须分割我们的图集，</font>这样我们就可以给每个灯提供它自己的tile来渲染。</font>

我们最多支持四个有阴影的灯光，我们将在我们的方形图集中给每个灯光一个<font color="green">square tile_方块地砖</font>。因此，如果我们最终有一个以上的阴影灯，我们就必须把 <font color="green">atlas分成四块，把tile size减半</font>。

在<font color="blue">Shadows.RenderDirectionalShadows</font>中确定<font color="green">分割量和tile size</font>，并将两者都传给每个灯的另一个方法。

```c#
	void RenderDirectionalShadows () {
		…
		
		int split = ShadowedDirectionalLightCount <= 1 ? 1 : 2;
		int tileSize = atlasSize / split;

		for (int i = 0; i < ShadowedDirectionalLightCount; i++) {
			RenderDirectionalShadows(i, split, tileSize);
		}
	}
	
	void RenderDirectionalShadows (int index, int split, int tileSize) { … }
```

我们可以通过调整<font color="green">render viewport.</font>来渲染到<font color="red">单个tile</font>。为此创建一个新的方法，它有<font color="blue">tile index and split as parameters。</font>

它首先计算<font color="red"> tile offset</font>，将 <font color="green">index modulo the split_索引与分割线 </font>相乘作为X偏移量，将<font color="green"> index divided by the split _索引除以分割线 </font>作为Y偏移量。

这些都是整数运算，但我们最终定义了一个<font color="green">Rect _ 矩形</font>，所以将结果存储为Vector2。

```c#
	void SetTileViewport (int index, int split) {
		Vector2 offset = new Vector2(index % split, index / split);
	}
```

然后用Rect在缓冲区调用<font color="RoyalBlue">SetViewPort</font>，<font color="green">偏移量按tile size缩放，这应该成为第三个参数</font>，它可以立即成为一个float.

```c#
	void SetTileViewport (int index, int split, float tileSize) {
		Vector2 offset = new Vector2(index % split, index / split);
		buffer.SetViewport(new Rect(
			offset.x * tileSize, offset.y * tileSize, tileSize, tileSize
		));
	}
```

>   [SetViewport](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.SetViewport.html) 
>
>   增加一条设置渲染视口的命令，在渲染目标改变后，视口会被设置为包含整个渲染目标的范围。这个命令可以用来将进一步的渲染限制在指定的像素矩形范围内
>
>   [rect 构造函数](https://docs.unity3d.com/ScriptReference/Rect-ctor.html) 
>
>   创建一个给定尺寸和位置的矩形。当你已经在使用Vector2值的时候，这种形式的构造函数是很方便的。
>

在设置<font color="green">矩阵时</font>	调用<font color="red">**RenderDirectionalShadows**</font>的<font color="blue">**SetTileViewport**</font>。

```
	SetTileViewport(index, split, tileSize);
	buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/rendering-shadows/four-tiles.png" alt="img" style="zoom:50%;" />

<center>Shadow atlas with four tiles in use.</center>

<font color="red">Light.Setup (…) => shadow.setp()   </font> 							 跟踪Shaodw实例
<font color="red">Shadows.Render</font> =>	RenderDirectionalShadows  		渲染阴影
=》Shadow.RenderDirectionalShadows							渲染单个灯光阴影
{
	SetTileViewport() 																分割视口
}



### [2、Sampling Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2)

现在我们正在渲染 <font color="blue">shadows casters</font>，但这还不影响最终的图像。为了使阴影显示出来，我们必须在<font color="red">CustomLit  pass</font>中对<font color="green">shadow map</font>进行采样，并利用它来确定一个表面片段是否有阴影。

#### [2.1、Shadow Materials](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2.1)

对于每个片段，我们必须从<font color="green">shadow atla</font>中的适当<font color="red">tile</font>中提取<font color="green">深度信息</font>。所以我们必须为给定一个<font color="red">world-space position _ 世界空间位置</font>找到<font color="RoyalBlue">shadow texture coordinates_阴影纹理坐标。</font>

我们将通过为每个阴影方向的光线创建一个<font color="green">shadow transformation matrix </font>并将它们发送到<font color="blue">GPU</font>来实现这一目标。

在<font color="green">Shadows</font>中添加一个<font color="red">_DirectionalShadowMatrices shader property</font> 和<font color="red">static matrix array _ 静态矩阵数组</font>来实现这一目标。

```c#
static int
		dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas"),
		dirShadowMatricesId = Shader.PropertyToID("_DirectionalShadowMatrices");
		
	static Matrix4x4[]
		dirShadowMatrices = new Matrix4x4[maxShadowedDirectionalLightCount];
```

我们可以通过在<font color="red">**RenderDirectionalShadows**</font>中乘以<font color="green">light's shadow projection矩阵和View矩阵</font>来创建一个从<font color="red">世界空间到灯光空间</font> 的转换矩阵。

```c#
void RenderDirectionalShadows (int index, int split, int tileSize) {
		…
		SetTileViewport(index, split, tileSize);
		dirShadowMatrices[index] = projectionMatrix * viewMatrix;
		buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
		…
	}
```

然后，一旦所有的阴影灯被渲染，通过调用缓冲区的<font color="RoyalBlue">SetGlobalMatrixArray</font>将矩阵发送到GPU。

```c#
	void RenderDirectionalShadows () {
		…

		buffer.SetGlobalMatrixArray(dirShadowMatricesId, dirShadowMatrices);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

然而，这忽略了一个事实，即我们正在使用<font color="red">**a shadow atlas**</font>。

让我们创建一个<font color="red">**ConvertToAtlasMatrix**</font>方法，该方法接收<font color="green">light matrix, tile offset, and split</font>，并返回一个从<font color="RoyalBlue">world space to shadow tile space</font>的矩阵

```c#
Matrix4x4 ConvertToAtlasMatrix (Matrix4x4 m, Vector2 offset, int split) {
		return m;
	}
```

我们已经在<font color="red">SetTileViewport</font>中计算了偏移量，所以让它返回这个值。

```c#
Vector2 SetTileViewport (int index, int split, float tileSize) {
		…
		return offset;
	}
```

然后调整<font color="red">RenderDirectionalShadows</font>，使其调用<font color="blue">ConvertToAtlasMatrix</font>。

```c#
//SetTileViewport(index, split, tileSize);
		dirShadowMatrices[index] = ConvertToAtlasMatrix(
			projectionMatrix * viewMatrix,
			SetTileViewport(index, split, tileSize), split
		);
```

在<font color="blue">ConvertToAtlasMatrix</font>中，我们应该做的第一件事是，如果使用了反转的Z缓冲区，就要否定Z维度。我们可以通过<font color="red">**SystemInfo.usesReversedZBuffer**</font>来检查。

```c#
Matrix4x4 ConvertToAtlasMatrix (Matrix4x4 m, Vector2 offset, int split) {
		if (SystemInfo.usesReversedZBuffer) {
			m.m20 = -m.m20;
			m.m21 = -m.m21;
			m.m22 = -m.m22;
			m.m23 = -m.m23;
		}
		return m;
	}
```

第二，<font color="RoyalBlue">clip space</font>被定义在一个立方体内，其坐标从-1到1，中心是0。

但纹理坐标和深度则从0到1。我们可以通过缩放和偏移XYZ尺寸的一半来将这种转换烘托到矩阵中。我们可以用矩阵乘法来做，但这将导致大量的零乘法和无谓的加法。所以让我们直接调整矩阵。

```c#
		m.m00 = 0.5f * (m.m00 + m.m30);
		m.m01 = 0.5f * (m.m01 + m.m31);
		m.m02 = 0.5f * (m.m02 + m.m32);
		m.m03 = 0.5f * (m.m03 + m.m33);
		m.m10 = 0.5f * (m.m10 + m.m30);
		m.m11 = 0.5f * (m.m11 + m.m31);
		m.m12 = 0.5f * (m.m12 + m.m32);
		m.m13 = 0.5f * (m.m13 + m.m33);
		m.m20 = 0.5f * (m.m20 + m.m30);
		m.m21 = 0.5f * (m.m21 + m.m31);
		m.m22 = 0.5f * (m.m22 + m.m32);
		m.m23 = 0.5f * (m.m23 + m.m33);
	return m;
```

最后，我们必须应用<font color="red"> tile offset and scale</font>。我们可以再次直接这样做，以避免大量不必要的计算。

```c#
	float scale = 1f / split;
		m.m00 = (0.5f * (m.m00 + m.m30) + offset.x * m.m30) * scale;
		m.m01 = (0.5f * (m.m01 + m.m31) + offset.x * m.m31) * scale;
		m.m02 = (0.5f * (m.m02 + m.m32) + offset.x * m.m32) * scale;
		m.m03 = (0.5f * (m.m03 + m.m33) + offset.x * m.m33) * scale;
		m.m10 = (0.5f * (m.m10 + m.m30) + offset.y * m.m30) * scale;
		m.m11 = (0.5f * (m.m11 + m.m31) + offset.y * m.m31) * scale;
		m.m12 = (0.5f * (m.m12 + m.m32) + offset.y * m.m32) * scale;
		m.m13 = (0.5f * (m.m13 + m.m33) + offset.y * m.m33) * scale;
```

#### [2.2、Storing Shadow Data Per Light](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2.2)

要对光的阴影进行采样，我们需要知道它在<font color="red"> index of its tile in the shadow atlas _ 阴影图集中的图块索引</font>（如果有的话）。

这是每个灯都必须存储的东西，所以让我们<font color="blue">ReserveDirectionalShadows</font>返回所需的数据。我们将提供两个值：<font color="RoyalBlue">shadow strength _ 阴影强度</font>和<font color="RoyalBlue">阴影tile offset量</font>，打包在一个Vector2. 如果光没有阴影，那么结果就是零向量。

```c#
	public Vector2 ReserveDirectionalShadows (…) {
		if (…) {
			ShadowedDirectionalLights[ShadowedDirectionalLightCount] =
				new ShadowedDirectionalLight {
					visibleLightIndex = visibleLightIndex
				};
			return new Vector2(
				light.shadowStrength, ShadowedDirectionalLightCount++
			);
		}
		return Vector2.zero;
	}
```

让<font color="green">**Lighting**</font>通过一个<font color="red">_DirectionalLightShadowData</font>向量数组将这些数据提供给<font color="RoyalBlue">Shader</font>。

```c#
static int
		dirLightCountId = Shader.PropertyToID("_DirectionalLightCount"),
		dirLightColorsId = Shader.PropertyToID("_DirectionalLightColors"),
		dirLightDirectionsId = Shader.PropertyToID("_DirectionalLightDirections"),
		dirLightShadowDataId =
			Shader.PropertyToID("_DirectionalLightShadowData");

	static Vector4[]
		dirLightColors = new Vector4[maxDirLightCount],
		dirLightDirections = new Vector4[maxDirLightCount],
		dirLightShadowData = new Vector4[maxDirLightCount];

	…

	void SetupLights () {
		…
		buffer.SetGlobalVectorArray(dirLightShadowDataId, dirLightShadowData);
	}

	void SetupDirectionalLight (int index, ref VisibleLight visibleLight) {
		dirLightColors[index] = visibleLight.finalColor;
		dirLightDirections[index] = -visibleLight.localToWorldMatrix.GetColumn(2);
		dirLightShadowData[index] =
			shadows.ReserveDirectionalShadows(visibleLight.light, index);
	}
```

And add it to the <font color="green">_CustomLight buffer  </font> in the <font color="red">Light HLSL </font> file as well.

```glsl
CBUFFER_START(_CustomLight)
	int _DirectionalLightCount;
	float4 _DirectionalLightColors[MAX_DIRECTIONAL_LIGHT_COUNT];
	float4 _DirectionalLightDirections[MAX_DIRECTIONAL_LIGHT_COUNT];
	float4 _DirectionalLightShadowData[MAX_DIRECTIONAL_LIGHT_COUNT];
CBUFFER_END
```

#### [2.3、Shadows HLSL File](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2.3)

我们还将创建一个专门的<font color="red">用于阴影采样的<font color="green">Shadows.HLSL</font>文件</font>。定义相同的最大阴影定向光计数，以及<font color="green">DirectionalShadowAtlas纹理</font>，加上<font color="RoyalBlue">CustomShadows缓冲区</font>中的<font color="green">DirectionalShadowMatrices数组</font>。

```glsl
#ifndef CUSTOM_SHADOWS_INCLUDED
#define CUSTOM_SHADOWS_INCLUDED

#define MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT 4

TEXTURE2D(_DirectionalShadowAtlas);
SAMPLER(sampler_DirectionalShadowAtlas);

CBUFFER_START(_CustomShadows)
	float4x4 _DirectionalShadowMatrices[MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT];
CBUFFER_END

#endif
```

<font color="red">atlas isn't a regular texture let's define_由于图集不是一个普通的纹理</font>，为了清楚起见，我们通过<font color="RoyalBlue">**TEXTURE2D_SHADOW**</font>宏来定义它，尽管这对我们支持的平台来说并没有什么区别。

我们将使用一个特殊的<font color="RoyalBlue">**SAMPLER_CMP**</font>宏来定义<font color="green">sampler</font>的状态，因为这确实定义了一种不同的阴影贴图采样方式，<font color="red">因为常规的<font color="RoyalBlue">**bilinear filtering** </font>对<font color="green">深度数据</font>没有意义。</font>

```glsl
TEXTURE2D_SHADOW(_DirectionalShadowAtlas);
SAMPLER_CMP(sampler_DirectionalShadowAtlas);
```

事实上，只有一种合适的方式来对<font color="RoyalBlue">shadow map</font>进行采样，所以我们可以定义一个明确的<font color="red">sampler state</font>，而不是依赖Unity为我们的<font color="green"> render texture</font>推导出来的状态。

采样器状态可以通过创建一个名称中带有特定单词的采样器来内联定义。

我们可以使用<font color="red">**sampler_linear_clamp_compare**</font>。让我们也为它定义一个简写的<font color="green">SHADOW_SAMPLER</font>宏。

```c#
TEXTURE2D_SHADOW(_DirectionalShadowAtlas);
#define SHADOW_SAMPLER sampler_linear_clamp_compare
SAMPLER_CMP(SHADOW_SAMPLER);
```

Include *Shadows* before *Light* in *LitPass*.

```c#
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Shadows.hlsl"
#include "../ShaderLibrary/Light.hlsl"
```

#### [2.4、Sampling Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2.4)

<font color="red">为了对阴影进行采样，我们需要知道每个灯光的阴影数据</font> ，所以让我们在<font color="red">**Shadows**</font>中定义一个结构，专门用于定向灯光。它包含了<font color="green">强度和tile offset</font>，但<font color="red">Shadows</font> 中的代码并不知道它被存储在哪里。

```c#
struct DirectionalShadowData {
	float strength;
	int tileIndex;
};
```

We also need to know the <font color="green">surface position,</font> so add it to the <font color="red">**Surface** </font> struct.

```c#
struct Surface {
	float3 position;
	…
};
```

And assign it in <font color="red">**LitPassFragment.**</font> 

```c#
	Surface surface;
	surface.position = input.positionWS;
	surface.normal = normalize(input.normalWS);
```

在<font color="red">**Shadows**</font> 中添加一个<font color="blue">**SampleDirectionalShadowAtlas**</font>函数，通过<font color="RoyalBlue">**SAMPLE_TEXTURE2D_SHADOW**</font>宏对 <font color="green">shadow atlas阴影图集</font>进行采样

将<font color="green"> atlas、 shadow sampler</font>和在<font color="green">position in shadow texture space, _ 阴影纹理空间的位置</font>传递给它，这是一个相应的参数。

<font color="red">STS = Shadow Texture Space</font>

```c#
float SampleDirectionalShadowAtlas (float3 positionSTS) {
	return SAMPLE_TEXTURE2D_SHADOW(
		_DirectionalShadowAtlas, SHADOW_SAMPLER, positionSTS
	);
}
```

添加一个<font color="red">**GetDirectionalShadowAttenuation**</font>函数，

给定<font color="reD">directional  shadow data _ 阴影数据</font>和一个<font color="green">表面</font>，返回<font color="red">shadow attenuation _ 阴影衰减</font>，这个表面应该定义在世界空间。

它<font color="red">使用<font color="green">tile offset</font> 来检索正确的矩阵，将<font color="RoyalBlue">表面位置转换为阴影tile空间</font>，然后对<font color="green">atlas</font>进行采样</font>  。

```glsl
float GetDirectionalShadowAttenuation (DirectionalShadowData data, Surface surfaceWS) {
	float3 positionSTS = mul(
		_DirectionalShadowMatrices[data.tileIndex],
		float4(surfaceWS.position, 1.0)
	).xyz;
	float shadow = SampleDirectionalShadowAtlas(positionSTS);
	return shadow;
}
```

对<font color="green">shadow atlas </font>进行采样的结果是一个系数，它决定了有多少光线到达表面，只考虑到阴影。

这是一个在0-1范围内的值，被称为<font color="red">attenuation（ 衰减因子）</font>。

如果片段完全被阴影覆盖，那么我们得到的是0，当它完全没有阴影时，我们得到的是1。介于两者之间的数值表示该片断是部分阴影的。

除此之外，灯光的阴影强度可以被降低，无论是出于艺术原因还是为了表现半透明表面的阴影。当强度降低到零时，衰减就完全不受阴影的影响，否则应该是1。

所以最终的衰减是通过1和采样的衰减之间的线性插值找到的，以强度为基础。

```c#
return lerp(1.0, shadow, data.strength);
```

但是当阴影强度为零时，就根本不需要对阴影进行采样，因为它们没有任何效果，甚至还没有被渲染过。在这种情况下，我们有一个没有阴影的光线，应该总是返回一个。

```c#
float GetDirectionalShadowAttenuation (DirectionalShadowData data, Surface surfaceWS) {
	if (data.strength <= 0.0) {
		return 1.0;
	}
	…
}
```

>   在着色器中进行分支是一个好主意吗？
>   分支曾经是低效的，但现代GPU可以很好地处理它们。你必须记住的是，片段的块是平行着色的。如果哪怕是一个片段以特定的方式进行分支，那么整个块都会这样做，即使所有其他片段都忽略了该代码路径的结果。在这种情况下，我们根据光线的强度进行分支，至少在这一点上，所有片段都是一样的。

#### [2.5、Attenuating Light](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#2.5)

We'll store the light's attenuation in the<font color="RoyalBlue"> Light</font> struct.

```c#
struct Light {
	float3 color;
	float3 direction;
	float attenuation;
};
```

Add a function to ***Light*** that gets the <font color="red">directional shadow data</font>.

```
DirectionalShadowData GetDirectionalShadowData (int lightIndex) {
	DirectionalShadowData data;
	data.strength = _DirectionalLightShadowData[lightIndex].x;
	data.tileIndex = _DirectionalLightShadowData[lightIndex].y;
	return data;
}
```

然后给<font color="red">GetDirectionalLight</font>添加一个<font color="green">world-space surface parameter</font>，让它检索<font color="RoyalBlue">directional shadow data </font>并使用<font color="red">**GetDirectionalShadowAttenuation**</font>来设置灯光的衰减。

```c#
Light GetDirectionalLight (int index, Surface surfaceWS) {
	Light light;
	light.color = _DirectionalLightColors[index].rgb;
	light.direction = _DirectionalLightDirections[index].xyz;
    
    
	DirectionalShadowData shadowData = GetDirectionalShadowData(index);
	light.attenuation = GetDirectionalShadowAttenuation(shadowData, surfaceWS);
	return light;
}
```

现在<font color="red">**Lighting**</font>中的<font color="red">GetLighting</font>也必须把<font color="green">surface</font>传给<font color="RoyalBlue">GetDirectionalLight</font>。而且现在表面应该是在世界空间中定义的，所以要相应地重命名这个参数。

<font color="red">**只有BRDF不关心光和面的空间，只要它们匹配即可**</font>。

```c#
float3 GetLighting (Surface surfaceWS, BRDF brdf) {
	float3 color = 0.0;
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		color += GetLighting(surfaceWS, brdf, GetDirectionalLight(i, surfaceWS));
	}
	return color;
}
```

Lighting.GetLighting (Surface surfaceWS, BRDF brdf) 

=> light.GetDirectionalLight														//get the light Data

=> Shadow.GetDirectionalShadowAttenuation     				 //return light attenuation



 <font color="red">让阴影发挥作用的最后一步是将<font color="red">衰减计入光的强度。</font></font>

```c#
float3 IncomingLight (Surface surface, Light light) {
	return
		saturate(dot(surface.normal, light.direction) * light.attenuation) *
		light.color;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/sampling-shadows/one-shadowed-light.png" alt="img" style="zoom:50%;" />

<center>One light with shadows; max distance 10; atlas size 512.</center>

我们终于有了阴影，但它们看起来很糟糕。不应该有阴影的表面最终被形成像素带的阴影伪影所覆盖。

这些是由阴影贴图的<font color="red">有限分辨率</font>导致的自阴影引起的。使用不同的分辨率可以改变伪影模式，但不能消除它们。

这些表面最终会部分形成阴影，但我们以后会处理这个问题。阴影伪影使我们很容易看到阴影贴图所覆盖的区域，所以我们现在要保留它们。



例如，我们可以看到阴影贴图只覆盖了可见区域的一部分，由最大阴影距离控制。改变最大距离可以增长或缩小该区域。阴影贴图是与光线方向一致的，而不是与摄像机一致。

一些阴影在最大距离之外是可见的，但也有一些是缺失的，当阴影被采样到地图的边缘之外时，就会变得很奇怪。

如果只有一个有阴影的光是活跃的，那么结果就会被钳制，否则采样就会跨越**tile**边界，一个光最终会使用另一个光的阴影。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/sampling-shadows/two-shadowed-lights.png" alt="img" style="zoom:50%;" />

<center>Two lights with shadows, both at half intensity.</center>

我们以后会正确地切断最大距离的阴影，但现在这些无效的阴影仍然可见。

## [3、Cascaded Shadow Maps](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3)

因为定向光会影响到最大阴影距离以内的所有东西，他们的阴影图最终会覆盖很大的区域。由于阴影贴图使用<font color="green">**orthographic projecting**</font>，阴影贴图中的每个纹素有一个固定<font color="blue">world-space size。</font>

如果这个尺寸太大，那么个别的阴影纹素就会清晰可见，导致阴影边缘参差不齐 ，小的阴影也会消失。这种情况可以通过增加<font color="red">**atlas size**</font>来缓解，但只能达到一定程度。

<font color="RoyalBlue">jagged _ 锯齿</font>

当使用<font color="red">perspective camera _ 透视摄影机</font>时，更远的东西看起来更小。

在一定的视觉距离上，一个阴影图纹素将映射到一个显示像素，这意味着阴影的分辨率在理论上是最佳的。在离摄像机较近的地方，我们需要更高的阴影分辨率，而在较远的地方，较低的分辨率就足够了。这表明，在理想情况下，我们会根据<font color="green"> shadow receiver</font> ，使用一个可变的阴影图分辨率。



级联阴影图是解决这个问题的一个办法。这个想法是，<font color="red">shadow casters</font> 被渲染了不止一次，所以每个光在图集中得到了多个atlas，被称为级联。

第一个级联只覆盖靠近摄像机的一个小区域，而连续的级联则以相同数量的 texels 覆盖一个越来越大的区域。然后，The shader对每个片断的最佳级联进行采样。

#### [3.1、Settings](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.1)

<font color="red">Unity的阴影代码支持每个方向的灯光最多有四个级联。</font>

到目前为止，我们只使用了一个单一的级联，它覆盖了所有<font color="RoyalBlue"> max shadow distances</font>。为了支持更多，我们将在方向性阴影设置中添加一个级联计数滑块。虽然我们可以对每个方向灯使用不同的数量，但对所有有阴影的方向灯使用相同的数量是最合理的。

每个级联都覆盖了阴影区域的一部分，直到<font color="RoyalBlue"> max shadow distances</font>。

我们将通过为前三个级联添加比例滑块来使确切的部分可以配置。最后一个级联总是覆盖整个范围，所以不需要一个滑块。默认情况下将级联计数设置为4，级联比率为0.1、0.25和0.5。这些比率应该在每级联中增加，但我们不会在用户界面中强制执行。

代码在Shadow Setting

```c#
	public struct Directional {

		public MapSize atlasSize;

		[Range(1, 4)]
		public int cascadeCount;

		[Range(0f, 1f)]
		public float cascadeRatio1, cascadeRatio2, cascadeRatio3;
	}

	public Directional directional = new Directional {
		atlasSize = MapSize._1024,
		cascadeCount = 4,
		cascadeRatio1 = 0.1f,
		cascadeRatio2 = 0.25f,
		cascadeRatio3 = 0.5f
	};
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-count-ratios.png" alt="img" style="zoom:50%;" />

<center>Cascade counts and ratios.</center>

<font color="red">ComputeDirectionalShadowMatricesAndCullingPrimitives</font>方法要求我们提供打包在**Vector3**中的比率，所以让我们为设置添加一个方便的属性，以这种形式检索它们。

>   CullingResults.[ComputeDirectionalShadowMatricesAndCullingPrimitives](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives.html)
>
>   Calculates the view and projection matrices and shadow split data for a directional light.
>
>   ![image-20230327233224044](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230327233224044.png)

```c#
	public Vector3 CascadeRatios =>
			new Vector3(cascadeRatio1, cascadeRatio2, cascadeRatio3);
```

#### [3.2、Rendering Cascades](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.2)

每个级联都需要自己的变换矩阵，所以<font color="red">Shadows</font>的<font color="green"> shadow matrix array size </font>必须乘以每个灯的最大级联数量，也就是四个。

```c#
	const int maxShadowedDirectionalLightCount = 4, maxCascades = 4;

	…

	static Matrix4x4[]
		dirShadowMatrices = new Matrix4x4[maxShadowedDirectionalLightCount * maxCascades];
```

在*<font color="green">Shadows</font>*中也要增加阵列的大小。

```glsl
#define MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT 4
#define MAX_CASCADE_COUNT 4

…

CBUFFER_START(_CustomShadows)
	float4x4 _DirectionalShadowMatrices
		[MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT * MAX_CASCADE_COUNT];
CBUFFER_END
```

这样做之后，Unity会抱怨<font color="green">shader's array size </font>发生了变化，但它不能使用新的大小。这是因为一旦固定的数组被着色器声称，它们的大小就不能在同一个会话中在GPU上改变。我们必须重新启动Unity来重新初始化它。

完成后，将<font color="red">Shadows.</font><font color="blue">ReserveDirectionalShadows</font>中返回的<font color="green">tile offset</font> 乘以配置的级联量，因为每个方向的光现在会要求<font color="red">多个连续的tiles</font>。

```c#
	return new Vector2(
				light.shadowStrength,
				settings.directional.cascadeCount * ShadowedDirectionalLightCount++
			);
```

同样，使用的<font color="red">tiles </font>数量会被乘以<font color="RoyalBlue">RenderDirectionalShadows</font>中，这意味着我们最终可能会有16块tiles ，需要分成4份。

```c#
	int tiles = ShadowedDirectionalLightCount * settings.directional.cascadeCount;
		int split = tiles <= 1 ? 1 : tiles <= 4 ? 2 : 4;
		int tileSize = atlasSize / split;
```

现在<font color="red">**RenderDirectionalShadows**</font>必须为每个级联绘制阴影。把从<font color="blue">**ComputeDirectionalShadowMatricesAndCullingPrimitives**</font>到<font color="red">DrawShadows</font>的代码放在每个配置的级联的一个循环中。



>   [CullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives.html)
>
>   计算一个定向灯的视图和投影矩阵以及阴影分割数据
>
>   ![image-20230327233823089](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230327233823089.png)
>



<font color="RoyalBlue">ComputeDirectionalShadowMatricesAndCullingPrimitives</font>的第二个参数现在变成了<font color="red">cascade index _ 级联索引</font>，然后是<font color="green">number and ratios of cascades _级联计数和级联比率</font>。

同时调整<font color="blue">tile index _ 瓦片索引</font>，使其成为<font color="blue">灯光的 tile offset加上级联索引。</font>

```c#
	void RenderDirectionalShadows (int index, int split, int tileSize) {
		ShadowedDirectionalLight light = shadowedDirectionalLights[index];
		var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
		int cascadeCount = settings.directional.cascadeCount;
		int tileOffset = index * cascadeCount;
		Vector3 ratios = settings.directional.CascadeRatios;
		
		for (int i = 0; i < cascadeCount; i++) {
			cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(
				light.visibleLightIndex, i, cascadeCount, ratios, tileSize, 0f,
				out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix,
				out ShadowSplitData splitData
			);
			shadowSettings.splitData = splitData;
			int tileIndex = tileOffset + i;
			dirShadowMatrices[tileIndex] = ConvertToAtlasMatrix(
				projectionMatrix * viewMatrix,
				SetTileViewport(tileIndex, split, tileSize), split
			);
			buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
			ExecuteBuffer();
			context.DrawShadows(ref shadowSettings);
		}
	}
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/one-light-four-cascades.png" alt="one" style="zoom: 67%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/four-lights-four-cascades.png" alt="two" style="zoom: 67%;" /></center>

<center>One and four lights with four cascades; max distance 30; ratios 0.3, 0.4, 0.5.</center>

#### 3.3、[Culling Spheres](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.3)

Unity通过为每个级联创建一个<font color="green">culling sphere</font>来确定其覆盖的区域。由于阴影的投影是<font color="red">orthographic and square</font>，它们最终与它们的剔除球体紧密结合，但也覆盖了它们周围的一些空间。这就是为什么有些影子可以在剔除区域之外看到。另外，光的方向对球体来说并不重要，所以所有方向的光最终都使用相同的剔除球体。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/culling-spheres.png" alt="img" style="zoom:50%;" />

​																	用透明球体可视化的剔除球体。

这些球体也需要用来确定从哪个级联中取样，所以我们必须将它们发送到GPU。添加一个<font color="red">indentifier for a count and a cascadeed culling sphere array _ 级联计数的标识符和一个级联剔除球体数组</font>，再加上一个<font color="green">static array for the sphere data _ 球体数据的静态阵列</font>。它们由四个分量的向量定义，包含它们的

**XYZ位置加上W分量的半径。**

```c#
	static int
		dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas"),
		dirShadowMatricesId = Shader.PropertyToID("_DirectionalShadowMatrices"),
		cascadeCountId = Shader.PropertyToID("_CascadeCount"),
		cascadeCullingSpheresId = Shader.PropertyToID("_CascadeCullingSpheres");

	static Vector4[] cascadeCullingSpheres = new Vector4[maxCascades];
```

级联的剔除球体是<font color="RoyalBlue">**ComputeDirectionalShadowMatricesAndCullingPrimitives**</font>输出的分割数据的一部分。

在<font color="red">**RenderDirectionalShadows**</font>的循环中把它分配给球体数组。但是我们<font color="green">只需要对第一个灯光做这个，因为所有灯光的级联都是相等的。</font>	

>   [ShadowSplitData](https://docs.unity3d.com/ScriptReference/Rendering.ShadowSplitData.html)
>
>   剔除球体。矢量的前三个分量描述球体中心，最后一个分量指定半径。
>
>   ![image-20230327235041351](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230327235041351.png)

```c#
	for (int i = 0; i < cascadeCount; i++) {
			cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(…);
			shadowSettings.splitData = splitData;
			if (index == 0) {
				cascadeCullingSpheres[i] = splitData.cullingSphere;
			}
			…
		}
```

<font color="red">我们需要<font color="green">Shader中的球体</font>来检查<font color="green">surface fragment</font>是否在其内部</font>

这可以通过比较球体中心的square距离和square半径来实现。因此，让我们来存储<font color="green">square radius_半径平方</font>，这样我们就不必在Shader中计算它了。

>   [shadowSplitData](https://docs.unity3d.com/ScriptReference/Rendering.ShadowSplitData.html) 描述一个给定的阴影分割的剔除信息（例如，定向级联）。

```c#
Vector4 cullingSphere = splitData.cullingSphere;
				cullingSphere.w *= cullingSphere.w;
				cascadeCullingSpheres[i] = cullingSphere;
```

Send the cascade count and spheres to the GPU after rendering the cascades.

```c#
void RenderDirectionalShadows () {
		…
		
		buffer.SetGlobalInt(cascadeCountId, settings.directional.cascadeCount);
		buffer.SetGlobalVectorArray(
			cascadeCullingSpheresId, cascadeCullingSpheres
		);
		buffer.SetGlobalMatrixArray(dirShadowMatricesId, dirShadowMatrices);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

#### [3.4、Sampling Cascades](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.4)

Add the<font color="RoyalBlue"> cascade count and culling spheres array </font>to<font color="red"> Shadows</font> .

```glsl
CBUFFER_START(_CustomShadows)
	int _CascadeCount;
	float4 _CascadeCullingSpheres[MAX_CASCADE_COUNT];
	float4x4 _DirectionalShadowMatrices
		[MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT * MAX_CASCADE_COUNT];
CBUFFER_END
```

<font color="red">cascade index</font>是由每个 <font color="RoyalBlue">fragment</font>,决定的，而不是由每个光决定的。所以让我们引入一个包含它的全局<font color="red">**ShadowData**</font> 结构。我们稍后会向它添加一些更多的数据。同时添加一个<font color="red">**GetShadowData**</font> 函数来返回<font color="green">data for a world-space surface _ 世界空间表面的阴影数据</font>，最初级联索引总是设置为零。

```c#
struct ShadowData {
	int cascadeIndex;
};

ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.cascadeIndex = 0;
	return data;
}
```

将新的数据作为参数添加到<font color="red">**GetDirectionalShadowData**</font> 中，这样它就可以通过将<font color="RoyalBlue"> cascade index</font>添加到灯光的阴影<font color="green">tile offset</font>来选择正确的<font color="green">tile index</font>。

<font color="blue">Code at light.hlsl</font>

```c#
DirectionalShadowData GetDirectionalShadowData (
	int lightIndex, ShadowData shadowData
) {
	DirectionalShadowData data;
	data.strength = _DirectionalLightShadowData[lightIndex].x;
	data.tileIndex =
		_DirectionalLightShadowData[lightIndex].y + shadowData.cascadeIndex;
	return data;
}
```

同时给<font color="blue">GetDirectionalLight</font>添加同样的参数，这样它就可以把数据转发给<font color="red">**GetDirectionalShadowData**</font>。适当地重命名方向性<font color="RoyalBlue"> shadow data variable</font>。

```c#
Light GetDirectionalLight (int index, Surface surfaceWS, ShadowData shadowData) {
	…
	DirectionalShadowData dirShadowData =
		GetDirectionalShadowData(index, shadowData);
	light.attenuation = GetDirectionalShadowAttenuation(dirShadowData, surfaceWS);
	return light;
}
```

在<font color="red">GetLighting</font>中获取<font color="green">阴影数据</font>，并将其传递出去。

```c#
float3 GetLighting (Surface surfaceWS, BRDF brdf) {
	ShadowData shadowData = GetShadowData(surfaceWS);
	float3 color = 0.0;
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		Light light = GetDirectionalLight(i, surfaceWS, shadowData);
		color += GetLighting(surfaceWS, brdf, light);
	}
	return color;
}
```

Lighting.getLighting => light.GetDirectionalLight() =>Shadows.GetDirectionalShadowAttenuation()

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-first.png" alt="第一的" style="zoom:67%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-last.png" alt="最后的" style="zoom: 67%;" />

<center>始终使用第一个与最后一个级联。</center>

要选择正确的级联，我们需要计算两点之间的平方距离。让我们<font color="red">Common</font>为此添加一个方便的功能。

```c#
float DistanceSquared(float3 pA, float3 pB) {
	return dot(pA - pB, pA - pB);
}
```

<font color="red">循环浏览<font color="green">Shadows.**GetShadowData**</font>中的所有<font color="green">cascade culling spheres _ 级联剔除球体</font>，直到找到一个包含<font color="blue">表面位置</font>的球体。</font>

<font color="red">**get 级联索引值**</font>

一旦找到，就跳出循环，<font color="green">然后使用当前的<font color="red">**循环迭代器**</font>作为<font color="red">**级联索引**</font></font>。这意味着如果<font color="RoyalBlue">the fragment</font>位于所有球体之外，我们最终会得到一个无效的索引，但我们现在会忽略这一点。

```c#
	int i;
	for (i = 0; i < _CascadeCount; i++) {
		float4 sphere = _CascadeCullingSpheres[i];
		float distanceSqr = DistanceSquared(surfaceWS.position, sphere.xyz);
		if (distanceSqr < sphere.w) {
			break;
		}
	}
	data.cascadeIndex = i;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/selecting-cascade.png" alt="img" style="zoom:50%;" />

<center>Selecting the best cascade.</center>

我们现在得到的阴影在<font color="green">texel</font>密度上的分布要好得多。

级联之间的弧形过渡边界也是由于<font color="green"> self-shadowing artifacts</font>而可见的，尽管我们可以通过用<font color="green"> shadow attenuation with the cascade index _ 级联指数代替阴影衰减</font>，再除以4来使它们更容易被发现。

```c#
Light GetDirectionalLight (int index, Surface surfaceWS, ShadowData shadowData) {
	…
	light.attenuation = GetDirectionalShadowAttenuation(dirShadowData, surfaceWS);
	light.attenuation = shadowData.cascadeIndex * 0.25;
	return light;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-indices.png" alt="img" style="zoom:50%;" />

<center>Shadowing with cascade indices.</center>						

#### [3.5、Culling Shadow Sampling](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.5)

如果我们最终<font color="red">超过了最后一个级联，那么很可能就没有有效的阴影数据，我们根本就不应该对阴影进行采样。</font>强制执行的一个简单方法是在<font color="blue">**ShadowData**</font>中添加一个强度字段，默认设置为1，如果我们最终超过了最后的级联，则设置为0。

```c#
struct ShadowData {
	int cascadeIndex;
	float strength;
};

ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.strength = 1.0;
	int i;
	for (i = 0; i < _CascadeCount; i++) {
		…
	}

	if (i == _CascadeCount) {
		data.strength = 0.0;
	}

	data.cascadeIndex = i;
	return data;
}
```

然后在<font color="blue">**GetDirectionalShadowData**</font>中把<font color="green">全局阴影强度</font>计入<font color="green">方向性阴影强度</font>。这样就可以消除最后一个级联以外的所有阴影。

<font color="blue"> code at  Shadow.cs.ReserveDirectionalShadows().x是光的强度         Light.hlsl</font>

```c#
	data.strength =
		_DirectionalLightShadowData[lightIndex].x * shadowData.strength;
```

另外，在<font color="blue">**GetDirectionalLight**</font>中恢复正确的衰减。

```c#
	light.attenuation = GetDirectionalShadowAttenuation(dirShadowData, surfaceWS);
	//light.attenuation = shadowData.cascadeIndex * 0.25;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/culled-shadows.png" alt="img" style="zoom:50%;" />

<center>Culled shadows; max distance 12.</center>

#### [3.6、Max Distance](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.6)

对最大阴影距离的一些实验会发现，一些<font color="red">**shadow casters**</font>在最后一个级联的剔除球内突然消失。这是因为最外层的剔除球体并不完全在配置的最大距离处结束，而是在它之外延伸了一点。这种差异在小的最大距离下最为明显。

我们可以通过停止对最大距离的阴影进行采样来解决阴影弹出的问题。为了实现这一点，我们必须在<font color="red">**Shadows**</font>中向GPU发送最大距离。

```c#
	static int
		…
		cascadeCullingSpheresId = Shader.PropertyToID("_CascadeCullingSpheres"),
		shadowDistanceId = Shader.PropertyToID("_ShadowDistance");

	…

	void RenderDirectionalShadows () {
		…
		buffer.SetGlobalFloat(shadowDistanceId, settings.maxDistance);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

最大距离是基于<font color="red">**view-space depth**</font>的深度，而不是到摄像机位置的距离。所以要进行这种<font color="blue">**culling**</font>，我们需要知道表面的深度。在<font color="green">**Surface**</font>中添加一个字段。

```c++
struct Surface {
	float3 position;
	float3 normal;
	float3 viewDirection;
	float depth;
	…
};
```

在<font color="red">**LitPassFragment**</font>中，深度可以通过<font color="RoyalBlue">**TransformWorldToView**</font>从世界空间转换到视图空间，并采取否定的Z坐标来找到。由于这种转换只是相对于世界空间的旋转和偏移，深度在视图空间和世界空间都是一样的。

```c++
	surface.viewDirection = normalize(_WorldSpaceCameraPos - input.positionWS);
	surface.depth = -TransformWorldToView(input.positionWS).z;
```

现在，在<font color="RoyalBlue">**GetShadowData**</font>中不要总是将强度初始化为1，而是只在表面深度小于最大距离时才这样做，否则将其设置为0。

```c#
CBUFFER_START(_CustomShadows)
	…
	float _ShadowDistance;
CBUFFER_END

…
float FadedShadowStrength (float distance, float scale, float fade) {
	return saturate((1.0 - distance * scale) * fade);
}

ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.strength = surfaceWS.depth < _ShadowDistance ? 1.0 : 0.0;
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/depth-culled-shadows.png" alt="img" style="zoom:50%;" />

<center>Also culled based on depth.</center>

#### [3.7、Fading Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.7)

突然切断最大距离处的阴影会非常明显，所以让我们通过线性淡入淡出来使过渡更平滑。 衰落在最大值之前的一段距离开始，直到我们在最大值处达到强度为零，我们可以使用函数

![CodeCogsEqn](D:\Google Download\CodeCogsEqn.gif)

为此钳制到 0–1，<font color="green">d是表面深度，m 是最大阴影距离，并且f是淡入淡出范围</font>，表示为最大距离的分数。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/distance-fade-graph.png" alt="img" style="zoom:50%;" />

​																			f 0.1, 0.2, and 0.5.

将<font color="RoyalBlue"> distance fade </font>的<font color="green">slider</font>添加到<font color="red">shadow settings</font>。 由于fade 和最大值都用作除数，因此它们不应为零，因此将它们的最小值设置为 0.001。

```c#
	[Min(0.001f)]
	public float maxDistance = 100f;
	
	[Range(0.001f, 1f)]
	public float distanceFade = 0.1f;
```

将 <font color="red">**Shadows** </font>中的阴影距离<font color="green">identifier</font>替换为一个用于<font color="green">距离值和淡入淡出值。</font>

```c#
//shadowDistanceId = Shader.PropertyToID("_ShadowDistance");
		shadowDistanceFadeId = Shader.PropertyToID("_ShadowDistanceFade");
```

将<font color="green">距离值和淡入淡出值</font>作为向量的 XY 分量发送到 GPU 时，使用一个除以值，这样我们就可以避免在<font color="green">Shader</font>中进行除法，因为乘法速度更快。

```c#
buffer.SetGlobalFloat(shadowDistanceId, settings.maxDistance);
		buffer.SetGlobalVector(
			shadowDistanceFadeId,
			new Vector4(1f / settings.maxDistance, 1f / settings.distanceFade)
		);
```

调整<font color="red">Shadows</font>中的<font color="RoyalBlue"> _CustomShadows </font>缓冲区以匹配。

```c#
	//float _ShadowDistance;
	float4 _ShadowDistanceFade;
```

现在我们可以使用计算<font color="red"> faded shadow strength</font>，并且替换公式

![img](E:\Typora file\SRP\assets\clip_image002.png)

替换为

![mylatex20230221_020147](E:\Typora file\SRP\assets\mylatex20230221_020147.svg)

用**1/m**取代**s**、用**1/f**取代**f**

<font color="green">d是表面深度，m 是最大阴影距离，并且f是淡入淡出范围，r是culling sphere 半径</font>

为此创建一个<font color="red"> FadedShadowStrength </font>函数并在<font color="RoyalBlue"> GetShadowData </font>中使用它。

```c#
float FadedShadowStrength (float distance, float scale, float fade) {
	return saturate((1.0 - distance * scale) * fade);
}

ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.strength = FadedShadowStrength(
		surfaceWS.depth, _ShadowDistanceFade.x, _ShadowDistanceFade.y
	);
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/distance-fade-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/distance-fade-scene.png" alt="scene" style="zoom:50%;" />

<center>Distance fade.</center>

#### [3.8、Fading Cascades](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#3.8)

我们也可以使用相同的方法淡化最后一个级联边缘的阴影，而不是将它们切断。 为此添加<font color="green"> cascade fade shadow  _ 级联衰减阴影</font>设置滑块。

<font color="RoyalBlue">Scripts at the shaodwSetting</font>

```c#
	public struct Directional {

		…

		[Range(0.001f, 1f)]
		public float cascadeFade;
	}

	public Directional directional = new Directional {
		…
		cascadeRatio3 = 0.5f,
		cascadeFade = 0.1f
	};
```

唯一的区别是我们使用的是级联的平方距离和半径，而不是线性深度和最大值。 这意味着过渡变得非线性：

![mylatex20230213_005445](E:\Typora file\SRP\assets\mylatex20230221_013943.svg)

<font color="green">d是表面深度，m 是最大阴影距离，并且f是淡入淡出范围，r是culling sphere 半径</font>，区别不是很大，但要保持配置的fade ratio 相同，我们必须替换**f**为

![mylatex20230221_014147](E:\Typora file\SRP\assets\mylatex20230221_014147.svg)

然后我们将它存储在阴影距离fade向量的 Z 分量中，再次倒置。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-fade-graph.png" alt="img" style="zoom:50%;" />

<center>f 0.1, 0.2, and 0.5 with square distance.</center>

```c#
	float f = 1f - settings.directional.cascadeFade;
		buffer.SetGlobalVector(
			shadowDistanceFadeId, new Vector4(
				1f / settings.maxDistance, 1f / settings.distanceFade,
				1f / (1f - f * f)
			)
		);
```

要执行级联渐变，检查我们是否在<font color="red">GetShadowData</font>的循环中处于最后一个级联中。如果是这样，计算渐变阴影强度，并将其纳入最终强度。

<font color="blue">脚本在Shadows.hlsl</font>

```c#
	for (i = 0; i < _CascadeCount; i++) {
		float4 sphere = _CascadeCullingSpheres[i];
		float distanceSqr = DistanceSquared(surfaceWS.position, sphere.xyz);
		if (distanceSqr < sphere.w) {
			if (i == _CascadeCount - 1) {
				data.strength *= FadedShadowStrength(
					distanceSqr, 1.0 / sphere.w, _ShadowDistanceFade.z
				);
			}
			break;
		}
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-fade-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/cascaded-shadow-maps/cascade-fade-scene.png" alt="scene" style="zoom:50%;" />

​																	<font style:text-align:center>Both cascade and distance fade.</font>

## [4、Shadow Quality](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4)

现在我们有了功能性的级联阴影贴图，让我们专注于提高阴影的质量。 我们一直观察到的伪影被称为<font color="red">阴影痤疮**（shadow acne）**</font>，这是由于表面的不正确自阴影未与光方向完全对齐而引起的。 随着表面越来越接近平行于光的方向，**Acne**会变得更严重。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/shadow-acne.png" alt="img" style="zoom:50%;" />

<center>Shadow acne.</center>

增加<font color="green">**atlas size**</font>会减小<font color="green">texels</font>的<font color="green"> world-space </font>大小，因此<font color="red">**acne artifacts**</font> 会变小。 但是，**artifacts** 的数量也随之增加，因此不能通过简单地增加图集大小来解决问题。

#### [4.1、Depth Bias](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.1)

有多种方法可以减轻阴影痤疮（**Shadow Acne**）。 最简单的方法是为**shadow casters**的深度添加一个恒定的偏差，将它们推离光线，这样不正确的自阴影就不会再发生。 

添加此技术的最快方法是在渲染时应用全局深度偏差，在<font color="red"> **DrawShadows** </font>之前在缓冲区上调用 <font color="red">**SetGlobalDepthBias**</font>，然后将其设置回零。

 这是应用于裁剪空间的深度偏差，是一个非常小的值的倍数，具体取决于用于阴影贴图的确切格式。 我们可以通过使用一个较大的值（例如 50000）来了解它是如何工作的。

第二个参数是关于<font color="red">斜率比例偏差**(slope-scale bias)**</font>，但我们现在将其保持为零。

<font color="RoyalBlue">scripts in the Shadows.cs .RenderDirectionalShadows()</font>

```c#
	buffer.SetGlobalDepthBias(50000f, 0f);
			ExecuteBuffer();
			context.DrawShadows(ref shadowSettings);
			buffer.SetGlobalDepthBias(0f, 0f);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/constant-depth-bias.png" alt="img" style="zoom:50%;" />

<center>Constant depth bias.</center>

恒定偏差很简单，但只能消除大部分正面照亮的表面的<font color="green">伪影(artifacts)</font>。 去除所有<font color="green">粉刺(acne)</font>需要更大的偏差，比如大一个数量级。

```c#
		buffer.SetGlobalDepthBias(500000f, 0f);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/large-depth-bias.png" alt="img" style="zoom:50%;" />

<center>Larger depth bias.</center>

<font color="red">**Peter-Panning**</font>

然而，随着<font color="red">depth bias pushes shadow casters away from the light _ 深度偏差将阴影投射器推离光线</font>，采样的阴影也会在相同的方向上移动。 大到足以去除大多数粉刺的偏差总是将阴影移动得如此之远，以至于它们似乎与它们的脚轮分离，从而导致称为彼得-潘宁的视觉伪影。

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/without-peter-panning.png" alt="without" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/with-peter-panning.png" alt="with" style="zoom:50%;" /></center>

<center>Bias causes peter-panning.</center>

另一种方法是应用坡度尺度偏差<font color="red">**（lope-scale bias）**</font>，这是通过对<font color="blue">SetGlobalDepthBias</font>的第二个参数使用非零值来实现的。

此值用于沿X和Y维度缩放<font color="red"> **absolute clip-space depth** _ 绝对裁剪空间深度</font>导数的最大值。

因此，对于正面照射的曲面，它是0，当光至少在两个维度中的一个以45°角照射时，它是1，当曲面法线和光方向的点积达到0时，它接近无穷大。所以当需要更多时，偏差会自动增加，但没有上限。

因此，消除痤疮所需的因子要低得多，例如3而不是500000。

```c#
	buffer.SetGlobalDepthBias(0f, 3f);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/slope-scale-bias.png" alt="img" style="zoom:50%;" />

<center>Slope scale bias.</center>

<font color="red">斜率偏差是有效的但不直观</font>。 需要进行实验才能得出可接受的结果，即用<font color="red"> **trades acne for Peter-Panning** _ 彼得潘宁代替粉刺</font>。 因此，让我们暂时禁用它并寻找一种更直观和可预测的方法。

```c#
//buffer.SetGlobalDepthBias(0f, 3f);
			ExecuteBuffer();
			context.DrawShadows(ref shadowSettings);
			//buffer.SetGlobalDepthBias(0f, 0f);
```

#### [4.2、Cascade Data](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.2)

因为<font color="blue">痤疮（ **acne**）</font>的大小取决于世界空间的<font color="green">texel</font>大小，所以一个在所有情况下都有效的一致方法必须考虑到这一点。

由于每个级联的texel大小不同，这意味着我们必须向GPU发送一些更多的级联数据。为此在<font color="red">**Shadows**</font>中添加一个通用的<font color="red">**cascade data vector array** _ 级联数据向量数组</font>。

```c#
	static int
		…
		cascadeCullingSpheresId = Shader.PropertyToID("_CascadeCullingSpheres"),
		cascadeDataId = Shader.PropertyToID("_CascadeData"),
		shadowDistanceFadeId = Shader.PropertyToID("_ShadowDistanceFade");

	static Vector4[]
		cascadeCullingSpheres = new Vector4[maxCascades],
		cascadeData = new Vector4[maxCascades];
```

将它与其他所有内容一起发送到 GPU。

```c#
	buffer.SetGlobalVectorArray(
			cascadeCullingSpheresId, cascadeCullingSpheres
		);
		buffer.SetGlobalVectorArray(cascadeDataId, cascadeData);
```

我们已经可以做的一件事是将<font color="green">平方级联半径的倒数</font>放在这些向量的 X 分量中, 这样我们就不必在<font color="blue">**Shader**</font>中执行除法。

 在新的 <font color="blue">SetCascadeData</font> 方法中执行此操作，同时存储剔除球体并在 <font color="red">RenderDirectionalShadows</font> 中调用它。 将<font color="green">级联索引、剔除球体和 tile size</font>作为float传递给它。

```c#
void RenderDirectionalShadows (int index, int split, int tileSize) {
		…
		
		for (int i = 0; i < cascadeCount; i++) {
			…
			if (index == 0) {
				SetCascadeData(i, splitData.cullingSphere, tileSize);
			}
			…
		}
	}

	void SetCascadeData (int index, Vector4 cullingSphere, float tileSize) {
		cascadeData[index].x = 1f / cullingSphere.w;
		cullingSphere.w *= cullingSphere.w;
		cascadeCullingSpheres[index] = cullingSphere;
	}
```

Add the cascade data to the <font color="blue">_CustomShadows buffer </font>in <font color="red">Shadows.</font>

```c#
CBUFFER_START(_CustomShadows)
	int _CascadeCount;
	float4 _CascadeCullingSpheres[MAX_CASCADE_COUNT];
	float4 _CascadeData[MAX_CASCADE_COUNT];
	…
CBUFFER_END
```

并在 <font color="RoyalBlue">GetShadowData </font>中使用新的逆预计算。And use the new precomputed inverse in GetShadowData.

```glsl
	data.strength *= FadedShadowStrength(
					distanceSqr, _CascadeData[i].x, _ShadowDistanceFade.z
				);
```

#### [4.3、Normal Bias](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.3)

发生不正确**（Incorrect）**的自阴影是因为<font color="red">阴影投射深度纹素</font>覆盖了多个片段，这导致<font color="green">**caster's volume**</font>从其表面伸出。

 因此，如果我们将**caster**缩小得足够多，这种情况就不会再发生了。

 然而，<font color="red">shrinking shadows caster </font> 会使阴影比它们应该的更小，并且会引入不应该存在的孔洞。

我们也可以做相反的事情：在对阴影进行采样的同时膨胀表面。 然后我们在离表面稍远的地方采样，刚好足以避免不正确的自阴影。 

这将稍微调整阴影的位置，可能会导致沿边缘未对齐并添加虚假阴影，但这些伪影往往远没有**Peter-Panning**那么明显。

为了采样阴影，我们可以通过<font color="red">表面位置将沿其法向量稍微移动</font> 来做到这一点。

如果我们只考虑一个维度，那么一个等于<font color="red">世界空间**texel**大小的偏移量</font> 就足够了。 我们可以通过将<font color="red">剔除球体的直径除以图块大小=>阴影分辨率</font>来在<font color="RoyalBlue"> **SetCascadeData** </font>中找到纹素大小。 将其存储在级联数据向量的 Y 分量中。

```c#
		float texelSize = 2f * cullingSphere.w / tileSize;
		cullingSphere.w *= cullingSphere.w;
		cascadeCullingSpheres[index] = cullingSphere;
		//cascadeData[index].x = 1f / cullingSphere.w;
		cascadeData[index] = new Vector4(
			1f / cullingSphere.w,
			texelSize
		);
```

然而，这并不总是足够的，因为纹素是正方形。 在最坏的情况下，我们最终不得不沿着正方形的对角线进行偏移，所以让我们将其缩放 √2。

```c#
	texelSize * 1.4142136f
```

<font color="red">**在Shader端**</font>，将<font color="green">全局阴影数据的参数</font>添加到 <font color="RoyalBlue">**GetDirectionalShadowAttenuation**</font>。 <font color="red">将<font color="green">表面法线</font>与<font color="green">偏移量</font>相乘以找到<font color="green">法线偏差</font>并将其添加到<font color="green">世界位置</font>，然后再计算<font color="green">阴影 **tile** 空间</font>中的位置。</font>

```c#
float GetDirectionalShadowAttenuation (
	DirectionalShadowData directional, ShadowData global, Surface surfaceWS
) {
	if (directional.strength <= 0.0) {
		return 1.0;
	}
	float3 normalBias = surfaceWS.normal * _CascadeData[global.cascadeIndex].y;
	float3 positionSTS = mul(
		_DirectionalShadowMatrices[directional.tileIndex],
		float4(surfaceWS.position + normalBias, 1.0)
	).xyz;
	float shadow = SampleDirectionalShadowAtlas(positionSTS);
	return lerp(1.0, shadow, directional.strength);
}
```

在 <font color="RoyalBlue">GetDirectionalLight </font>中将额外数据传递给它。

```glsl
	light.attenuation =
		GetDirectionalShadowAttenuation(dirShadowData, shadowData, surfaceWS);
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/normal-bias-sphere.png" alt="sphere" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/normal-bias-cube.png" alt="cube" style="zoom:50%;" /></center>

<center>Normal bias equal to texel size.</center>

#### [4.4、Configurable Biases](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.4)

<font color="red">normal bias法线偏移</font>可以摆脱<font color="blue">阴影acne</font>而不引入明显的新伪影，但它不能消除所有阴影问题。例如，在墙壁下面的地板上可以看到一些不应该出现的阴影线。

这不是自我阴影，而是从墙上探出的阴影影响了它下面的地板。添加一点斜率偏置<font color="red">( **slope-scale** )</font>可以处理这些问题，但没有一个完美的数值。因此，我们将使用现有的<font color="green">**Bias slider**</font>对每个灯进行配置。

在<font color="red">Shadows</font>**中的<font color="RoyalBlue">ShadowedDirectionalLight</font>**结构中添加一个字段。

```c#
struct ShadowedDirectionalLight {
		public int visibleLightIndex;
		public float slopeScaleBias;
	}
```

<font color="red">灯光的bias</font>是通过它的<font color="green">shadowBias</font>属性提供的。把它添加到<font color="blue">ReserveDirectionalShadows</font>的数据中。

>   [Light.shadowBias](https://docs.unity3d.com/ScriptReference/Light-shadowBias.html) Shadow mapping constant bias。

Shadow caster的表面被推到远离光线的世界空间量，以帮助防止自我阴影（"<font color="red">阴影痤疮shadow acne</font>）的工件。

```c#
	shadowedDirectionalLights[ShadowedDirectionalLightCount] =
				new ShadowedDirectionalLight {
					visibleLightIndex = visibleLightIndex,
					slopeScaleBias = light.shadowBias
				};
```

并用它来配置<font color="red">**RenderDirectionalShadows**</font> 中的坡度偏差。

```c#
		buffer.SetGlobalDepthBias(0f, light.slopeScaleBias);
			ExecuteBuffer();
			context.DrawShadows(ref shadowSettings);
			buffer.SetGlobalDepthBias(0f, 0f);
```

让我们也使用灯光现有的<font color="red">Normal Bias slider</font> 来调节我们应用的 <font color="red">Normal bias</font>。让<font color="green">**ReserveDirectionalShadows**</font>返回一个Vector3并使用灯光的<font color="green">shadowNormalBias</font>作为新的Z分量。

```c#
	public Vector3 ReserveDirectionalShadows (
		Light light, int visibleLightIndex
	) {
		if (…) {
			…
			return new Vector3(
				light.shadowStrength,
				settings.directional.cascadeCount * ShadowedDirectionalLightCount++,
				light.shadowNormalBias
			);
		}
		return Vector3.zero;
	}
```

将新的法线偏向添加到<font color="red">DirectionalShadowData </font> 并在<font color="red">Shadows</font>中的<font color="blue">GetDirectionalShadowAttenuation</font>中应用它。

<font color="RoyalBlue">代码在Shadow.hlsl</font>

```c#
struct DirectionalShadowData {
	float strength;
	int tileIndex;
	float normalBias;
};

…

float GetDirectionalShadowAttenuation (…) {
	…
	float3 normalBias = surfaceWS.normal *
		(directional.normalBias * _CascadeData[global.cascadeIndex].y);
	…
}
```

并在<font color="red">Light</font>的<font color="blue">GetDirectionalShadowData</font>中进行配置。

```c#
	data.tileIndex =
		_DirectionalLightShadowData[lightIndex].y + shadowData.cascadeIndex;
		
	data.normalBias = _DirectionalLightShadowData[lightIndex].z;
```

我们现在可以调整每个灯的两个biases。斜率偏压为零，normal bias 为一，这是一个很好的默认值。如果你增加第一个，你可以减少第二个。

但请记住，我们对这些灯光设置的解释与它们最初的目的不同。它们曾经是<font color="red">**clip-space depth bias**</font>和<font color="red"> **world-space shrinking normal bias**</font>。所以当你创建一个新的灯光时，你会得到严重的<font color="RoyalBlue">Peter-Panning </font>，直到你调整biases

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/light-settings.png" alt="settings" style="zoom:67%;" />

<center>
    <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/configured-bias-sphere.png" alt="sphere" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/configured-bias-cube.png" alt="cube" style="zoom:50%;" /></center>

<center>Both biases set to 0.6.</center>

#### [4.5、Shadow Pancaking](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.5)

另一个可能导致伪影的潜在问题是Unity应用了<font color="green">shadow pancaking</font>。这个想法是，在渲染定向光的阴影投射时，近平面会尽可能地向前移动。这增加了深度精度，但这意味着不在摄像机视野内的阴影投射可能最终出现在近平面的前面，这导致它们被剪切，而它们不应该被剪切。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/clipped-shadows.png)

<center>Shadows get clipped.</center>

这可以通过在<font color="red">**ShadowCasterPassVertex**</font>中把顶点位置夹在近平面上来解决，有效地把位于近平面前面的阴影投射器压平，把它们变成贴在近平面上的薄饼。

我们通过取clip空间的最大值Z和W坐标的最大值，或者在定义了<font color="blue">**UNITY_REVERSED_Z**</font>时取其最小值来实现。为了给W坐标使用正确的符号，请将其与<font color="RoyalBlue">UNITY_NEAR_CLIP_VALUE</font>相乘。

<font color="red">UNITY_NEAR_CLIP_VALUE 		<font color="blue">// 剪裁空间中近屏幕面的值</font></font>

```c#
output.positionCS = TransformWorldToHClip(positionWS);

	#if UNITY_REVERSED_Z
		output.positionCS.z =
			min(output.positionCS.z, output.positionCS.w * UNITY_NEAR_CLIP_VALUE);
	#else
		output.positionCS.z =
			max(output.positionCS.z, output.positionCS.w * UNITY_NEAR_CLIP_VALUE);
	#endif
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/clamped-shadows.png)

<center>Shadows get clamped.</center>

这对于完全在近平面两侧的投影者来说是完美的，但是跨越平面的投影者会变形，因为只有它们的一些顶点受到影响。这对小的三角形来说并不明显，但大的三角形最终会产生很大的变形，使它们弯曲，并经常导致它们沉入表面。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/deformed-shadows.png)

<center>Deformed shadows of very long cube.</center>

这个问题可以通过把近平面往后拉一点来缓解。这就是灯光的近平面滑块的作用。在<font color="red">**ShadowedDirectionalLight**</font>中添加一个近平面偏移的字段。

```c#
	struct ShadowedDirectionalLight {
		public int visibleLightIndex;
		public float slopeScaleBias;
		public float nearPlaneOffset;
	}
```

And copy the light's shadowNearPlane property to it.

```c#
	shadowedDirectionalLights[ShadowedDirectionalLightCount] =
				new ShadowedDirectionalLight {
					visibleLightIndex = visibleLightIndex,
					slopeScaleBias = light.shadowBias,
					nearPlaneOffset = light.shadowNearPlane
				};
```

我们通过填写<font color="RoyalBlue">**ComputeDirectionalShadowMatricesAndCullingPrimitives**</font>的最后一个参数来应用它，我们仍然给了一个固定的值0。

```c#
	cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(
				light.visibleLightIndex, i, cascadeCount, ratios, tileSize,
				light.nearPlaneOffset, out Matrix4x4 viewMatrix,
				out Matrix4x4 projectionMatrix, out ShadowSplitData splitData
			);
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/near-plane-offset.png)

<center>With near plane offset.</center>

#### [4.6、PCF Filtering](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.6)

到目前为止，我们只使用了<font color="red">硬阴影</font>，通过对每个 **fragment**的阴影贴图进行一次采样。

阴影比较采样器使用一种特殊形式的双线性插值，在插值之前进行深度比较。这就是所谓的百分比接近滤波--简称PCF--特别是2×2的PCF滤波，因为涉及到四个文本。

但这并不是我们过滤阴影图的唯一方法。我们还可以使用一个更大的滤波器，使阴影更柔和，更少的混叠，尽管也不太准确。让我们增加对2×2、3×3、5×5和7×7过滤的支持。我们将不使用现有的软阴影模式来控制每个光线。我们将使所有方向性的灯光使用相同的滤镜。为此在<font color="red">**ShadowSettings**</font>中添加一个<font color="green">	**FilterMode**</font>枚举，以及一个默认设置为2×2的定向滤镜选项。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/filter.png" alt="img" style="zoom: 80%;" />

<center>filter set to PCF 2x2.</center>

我们将为新的过滤模式创建着色器变体。在<font color="green">Shadows</font>中添加一个带有三个关键字的静态数组。

```c#
static string[] directionalFilterKeywords = {
		"_DIRECTIONAL_PCF3",
		"_DIRECTIONAL_PCF5",
		"_DIRECTIONAL_PCF7",
	};
```

创建一个<font color="red">SetKeywords</font>方法，。在执行缓冲区之前，在<font color="blue">RenderDirectionalShadows</font>中启用或禁用相应的关键字。

>   [CommandBuffer.EnableShaderKeyword](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.EnableShaderKeyword.html) 添加一条命令，以启用一个具有给定名称的全局关键词。

```c#
	void RenderDirectionalShadows () {
		…
		SetKeywords();
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}

	void SetKeywords () {
		int enabledIndex = (int)settings.directional.filter - 1;
		for (int i = 0; i < directionalFilterKeywords.Length; i++) {
			if (i == enabledIndex) {
				buffer.EnableShaderKeyword(directionalFilterKeywords[i]);
			}
			else {
				buffer.DisableShaderKeyword(directionalFilterKeywords[i]);
			}
		}
	}
```

较大的过滤器需要更多的纹理样本。我们需要知道着色器中的<font color="red">atlas 尺寸和纹理尺寸</font>来做到这一点。为这些数据添加一个着色器标识符。

```c#
	cascadeDataId = Shader.PropertyToID("_CascadeData"),
		shadowAtlasSizeId = Shader.PropertyToID("_ShadowAtlasSize"),
		shadowDistanceFadeId = Shader.PropertyToID("_ShadowDistanceFade");
```

And add it to <font color="green">_CustomShadow</font> on the shader side.

```glsl
CBUFFER_START(_CustomShadows)
	…
	float4 _ShadowAtlasSize;
	float4 _ShadowDistanceFade;
CBUFFER_END
```

在其X分量中存储尺寸，在其Y分量中存储texel尺寸。

```c#
	SetKeywords();
		buffer.SetGlobalVector(
			shadowAtlasSizeId, new Vector4(atlasSize, 1f / atlasSize)
		);
```

在Lit的<font color="green">CustomLit</font>通道上添加<font color="RoyalBlue">#pragma multi_compile</font>指令，用于三个关键词，加上和下划线用于匹配2×2过滤器的无关键词选项。

```glsl
	#pragma shader_feature _PREMULTIPLY_ALPHA
	#pragma multi_compile _ _DIRECTIONAL_PCF3 _DIRECTIONAL_PCF5 _DIRECTIONAL_PCF7
	#pragma multi_compile_instancing
```

我们将使用<font color="red">*Core*RP</font>库的<font color="red">Shadow/ShadowSamplingTent .HLSL</font>文件中定义的函数，所以在**Shadows**的顶部包含它。

如果定义了3×3关键字，我们总共需要四个滤镜样本，我们将用<font color="RoyalBlue">**SampleShadow_ComputeSamples_Tent_3x3**</font>函数来设置。我们只需要取四个样本，因为每个样本都使用双线性2×2滤波器。那些在各个方向上偏移了半个纹素的正方形覆盖了3×3个<font color="green">texel</font>的 <font color="green">texels</font>滤波器，中心的权重比边缘的更强。

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Shadow/ShadowSamplingTent.hlsl"

#if defined(_DIRECTIONAL_PCF3)
	#define DIRECTIONAL_FILTER_SAMPLES 4
	#define DIRECTIONAL_FILTER_SETUP SampleShadow_ComputeSamples_Tent_3x3
#endif

#define MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT 4
#define MAX_CASCADE_COUNT 4
```

出于同样的原因，我们可以为 5×5 滤波器提供 9 个样本，为 7×7 滤波器提供 16 个样本，再加上适当命名的函数就足够了。

```glsl
#if defined(_DIRECTIONAL_PCF3)
	#define DIRECTIONAL_FILTER_SAMPLES 4
	#define DIRECTIONAL_FILTER_SETUP SampleShadow_ComputeSamples_Tent_3x3
#elif defined(_DIRECTIONAL_PCF5)
	#define DIRECTIONAL_FILTER_SAMPLES 9
	#define DIRECTIONAL_FILTER_SETUP SampleShadow_ComputeSamples_Tent_5x5
#elif defined(_DIRECTIONAL_PCF7)
	#define DIRECTIONAL_FILTER_SAMPLES 16
	#define DIRECTIONAL_FILTER_SETUP SampleShadow_ComputeSamples_Tent_7x7
#endif
```

为一个<font color="red">阴影tile空间</font>位置创建一个新的<font color="RoyalBlue">FilterDirectionalShadow</font>函数。当<font color="red">DIRECTIONAL_FILTER_SETUP</font> 被定义时，它需要进行多次采样，否则只需调用一次<font color="RoyalBlue">SampleDirectionalShadowAtlas</font>就足够了。

```c#
float FilterDirectionalShadow (float3 positionSTS) {
	#if defined(DIRECTIONAL_FILTER_SETUP)
		float shadow = 0;
		return shadow;
	#else
		return SampleDirectionalShadowAtlas(positionSTS);
	#endif
}
```

滤波器设置函数有四个参数。

首先是float4中的size，前两个部分是<font color="red">the X and Y texel sizes</font>， <font color="red">total texture sizes in Z and W.</font>

然后是<font color="red">原始样本位置</font>

接着是每个<font color="red">样本的权重和位置的输出参数</font>。它们被定义为float和float2数组。

之后，我们可以循环浏览所有的样本，将它们按<font color="red">权重进行累积</font>。

```glsl
#if defined(DIRECTIONAL_FILTER_SETUP)
		float weights[DIRECTIONAL_FILTER_SAMPLES];
		float2 positions[DIRECTIONAL_FILTER_SAMPLES];
		float4 size = _ShadowAtlasSize.yyxx;

		DIRECTIONAL_FILTER_SETUP(size, positionSTS.xy, weights, positions);
		float shadow = 0;
		for (int i = 0; i < DIRECTIONAL_FILTER_SAMPLES; i++) {
			shadow += weights[i] * SampleDirectionalShadowAtlas(
				float3(positions[i].xy, positionSTS.z)
			);
		}
		return shadow;
	#else
```

在<font color="red">GetDirectionalShadowAttenuation</font>中调用这个新函数，而不是直接进入<font color="RoyalBlue">SampleDirectionalShadowAtlas。</font>

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf2.png" alt="PCF 2x2" style="zoom: 33%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf3.png" alt="PCF 3x3" style="zoom: 33%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf5.png" alt="PCF 5x5" style="zoom: 33%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf7.png" alt="PCF 7x7" style="zoom: 33%;" /></center>
<center>PCF 2x2, 3x3, 5x5, and 7x7.</center>

增加<font color="red"> filter size </font>会使阴影更平滑，但也会导致<font color="red">痤疮</font> 再次出现。我们必须增加<font color="red">法线偏差以匹配PCF</font> 的大小。我们可以通过在<font color="RoyalBlue">SetCascadeData</font>中把<font color="green">texel</font>大小乘以<font color="green">filter  + 1</font>来自动做到这一点。

<font color="RoyalBlue">代码在Shadow.cs</font>

```c#
void SetCascadeData (int index, Vector4 cullingSphere, float tileSize) {
		float texelSize = 2f * cullingSphere.w / tileSize;
		float filterSize = texelSize * ((float)settings.directional.filter + 1f);
		…
			1f / cullingSphere.w,
			filterSize * 1.4142136f
		);
	}
```

除此之外，增加采样区域也意味着我们最终会在级联的剔除球体之外采样。我们可以通过在平方之前将球体的半径减掉滤波器的大小来避免这种情况。

```c#
	cullingSphere.w -= filterSize;
	cullingSphere.w *= cullingSphere.w;
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf5-scaled-bias.png" alt="PCF 5x5" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/pcf7-scaled-bias.png" alt="PCF 7x7" style="zoom:50%;" /></center>

<center>PCF 5x5 and 7x7 with scaled bias.</center>

这又解决了阴影痤疮的问题，但增加的滤镜尺寸加剧了应用<font color="red">normal bias</font>的弊端，并使我们先前看到的墙面阴影问题也变得更糟。需要一些<font color="red"> slope-scale bias</font>或更大的图集尺寸来缓解这些假象。

>   我们是不是也应该减少半径以考虑到法线偏差？
>   法线偏差是根据每个光定义的，所以不能根据级联来设置。幸运的是，只有当偏置会将阴影采样推到所选级联之外时，它才会成为一个问题。这只发生在远离摄像机的法线上，这意味着它几乎总是限制在不可见的表面。如果偏置确实导致了问题，那么你可以通过一些可配置的因素来增加半径的减少。

#### [4.7、Blending Cascades](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.7)

较柔和的阴影看起来更好，但它们也会使级联之间的突然过渡更加明显。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/hard-cascade-transition.png" alt="img" style="zoom:67%;" />

<center>硬级联转换；PCF 7x7</center>

我们可以通过在我们混合两者的级联之间添加一个过渡区域来使过渡不那么明显——尽管没有完全隐藏。我们已经有了可用于此目的的级联淡入淡出因子。

首先，添加<font color="RoyalBlue">cascade blend value</font>到<font color="green">ShadowData</font>在<font color="red">Shadows.Cs</font>，我们将使用它在相邻的级联之间进行插值。

```c#
struct ShadowData {
	int cascadeIndex;
	float cascadeBlend;
	float strength;
};
```

最初 在<font color="red">GetShadowData</font>里将 blend 设置为 1，表示所选级联处于最大强度。

然后总是在级联处于循环中时计算<font color="RoyalBlue">fade factor</font>。如果我们在最后一个级联中，它会像以前一样影响强度，否则将其用于混合。

```c#
	data.cascadeBlend = 1.0;
	data.strength = FadedShadowStrength(
		surfaceWS.depth, _ShadowDistanceFade.x, _ShadowDistanceFade.y
	);
	int i;
	for (i = 0; i < _CascadeCount; i++) {
		float4 sphere = _CascadeCullingSpheres[i];
		float distanceSqr = DistanceSquared(surfaceWS.position, sphere.xyz);
		if (distanceSqr < sphere.w) {
			float fade = FadedShadowStrength(
				distanceSqr, _CascadeData[i].x, _ShadowDistanceFade.z
			);
			if (i == _CascadeCount - 1) {
				data.strength *= fade;
			}
			else {
				data.cascadeBlend = fade;
			}
			break;
		}
	}
```

现在在<font color="RoyalBlue">GetDirectionalShadowAttenuation</font>中检查在检索第一个阴影值后，<font color="green">cascade blend</font>是否小于1。如果是的话，我们就处于一个过渡区，而且还必须从下一个级联中取样并在两个值之间插值。

```c#
	float shadow = FilterDirectionalShadow(positionSTS);
	if (global.cascadeBlend < 1.0) {
		normalBias = surfaceWS.normal *
			(directional.normalBias * _CascadeData[global.cascadeIndex + 1].y);
		positionSTS = mul(
			_DirectionalShadowMatrices[directional.tileIndex + 1],
			float4(surfaceWS.position + normalBias, 1.0)
		).xyz;
		shadow = lerp(
			FilterDirectionalShadow(positionSTS), shadow, global.cascadeBlend
		);
	}
	return lerp(1.0, shadow, directional.strength);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/soft-cascade-transition.png" alt="img" style="zoom:50%;" />

<center>Soft cascade transitions.</center>

注意，层<font color="RoyalBlue"> cascade fade ratio</font>适用于每个级联的整个半径，而不仅仅是其可见部分。所以要确保这个比例不会一直延伸到较低的级联。一般来说，这不是一个问题，因为你想保持过渡区域很小。

#### [4.8、Dithered Transition](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.8)

虽然级联之间的混合看起来更好，但它也使我们在混合区域对阴影贴图的采样次数增加了一倍。另一种方法是基于抖动模式，始终从一个级联中取样。这看起来没有那么好，但却便宜得多，尤其是在使用大型过滤器时。

为 " <font color="green">Directional</font>"添加一个级联混合模式选项，支持硬、软或<font color="green">dither</font>的方法。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/cascade-blend-mode.png" alt="img" style="zoom:50%;" />

<center>级联混合模式。</center>

为 <font color="green">soft</font> 和<font color="green"> dither </font>级联混合关键字添加一个静态数组<font color="red">Shadows。</font>

```c#
	static string[] cascadeBlendKeywords = {
		"_CASCADE_BLEND_SOFT",
		"_CASCADE_BLEND_DITHER"
	};
```

调整<font color="red">SetKeywords</font>使其适用于任意关键字数组和索引，然后还设置级联混合关键字。

```c#
	void RenderDirectionalShadows () {
		SetKeywords(
			directionalFilterKeywords, (int)settings.directional.filter - 1
		);
		SetKeywords(
			cascadeBlendKeywords, (int)settings.directional.cascadeBlend - 1
		);
		buffer.SetGlobalVector(
			shadowAtlasSizeId, new Vector4(atlasSize, 1f / atlasSize)
		);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}

	void SetKeywords (string[] keywords, int enabledIndex) {
		//int enabledIndex = (int)settings.directional.filter - 1;
		for (int i = 0; i < keywords.Length; i++) {
			if (i == enabledIndex) {
				buffer.EnableShaderKeyword(keywords[i]);
			}
			else {
				buffer.DisableShaderKeyword(keywords[i]);
			}
		}
	}
```

Add the required multi-compile direction to the <font color="green">CustomLit pass</font>.

```c#
		#pragma multi_compile _ _CASCADE_BLEND_SOFT _CASCADE_BLEND_DITHER
			#pragma multi_compile_instancing
```

为了进行抖动，我们需要一个抖动的浮动值，我们可以把它加到<font color="red">Surface</font>中。

```
struct Surface {
	…
	float dither;
};
```

在<font color="green">LitPassFragment</font>中，有多种方法来生成抖动值。最简单的是使用<font color="green">Core RP</font>库中的<font color="red">InterleavedGradientNoise</font> 函数，它能在<font color="red">屏幕空间的XY位置生成一个旋转的平铺的抖动图案。</font> 

在<font color="red">fragment</font> 函数中，这等于<font color="green">screen-space XY position</font>。它还需要第二个参数，用来制作动画，我们不需要这个参数，可以让它保持为零。

```glsl
	surface.smoothness =
		UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Smoothness);
	surface.dither = InterleavedGradientNoise(input.positionCS.xy, 0);
```

在<font color="red">GetShadowData</font> 中设置级联索引之前，当不使用软混合时，将<font color="green"> cascade blend</font>设置为零。这样一来，整个分支将从那些着色器变体中消除。

```glsl
	if (i == _CascadeCount) {
		data.strength = 0.0;
	}
	#if !defined(_CASCADE_BLEND_SOFT)
		data.cascadeBlend = 1.0;
	#endif
	data.cascadeIndex = i;
```

而当使用<font color="red">dither blend _ 抖动混合</font> 时，如果我们不在最后一个级联中，如果<font color="green">blend </font>小于<font color="green">dither </font>值，就跳到下一个级联。

```c#
	if (i == _CascadeCount) {
		data.strength = 0.0;
	}
	#if defined(_CASCADE_BLEND_DITHER)
		else if (data.cascadeBlend < surfaceWS.dither) {
			i += 1;
		}
	#endif
	#if !defined(_CASCADE_BLEND_SOFT)
		data.cascadeBlend = 1.0;
	#endif
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/dithered-cascades.png" alt="img" style="zoom:50%;" />

<center>Dithered cascades.</center>

抖动混合的可接受程度取决于我们渲染画面时的分辨率。如果使用后期效果来弄脏最终结果，那么它可能会相当有效，例如与时间抗锯齿<font color="red">（temporal anti-aliasin）</font>和动画抖动模式相结合<font color="red">（ animated dither pattern）</font>。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/shadow-quality/dithered-zoomed-in.png" alt="img" style="zoom:50%;" />

​																				Dithered zoomed in.

#### [4.9、Culling Bias](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#4.9)

使用级联阴影贴图的一个缺点是，我们最终会在每个灯光下<font color="red">多次渲染shadow casters</font> 。如果能保证它们的结果总是被一个较小的级联所覆盖，那么尝试从较大的级联中剔除一些阴影投射器是有意义的。

Unity通过将分割数据的<font color="RoyalBlue">shadowCascadeBlendCullingFactor</font>设置为1来实现这一点。在应用于阴影设置之前，在<font color="RoyalBlue">RenderDirectionalShadows</font>中做这个。

>   [shadowCascadeBlendCullingFactor](https://docs.unity3d.com/ScriptReference/Rendering.ShadowSplitData-shadowCascadeBlendCullingFactor.html)
>
>   应用于剔除球体半径的乘数。
>
>   值必须在 0 到 1 的范围内。值越高，Unity 剔除的对象越多。使用较低级联共享更多渲染对象。<font color="red">使用较低的值允许在不同级联之间混合，因为它们随后共享对象。</font>
>

```c#
splitData.shadowCascadeBlendCullingFactor = 1f;
shadowSettings.splitData = splitData;
```

<center><img src="E:\Typora file\SRP\assets\culling-bias-0.png" alt="0" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\culling-bias-1.png" alt="1" style="zoom:50%;" /></center>

<center>Culling bias 0 and 1.</center>

这个值是一个系数，<font color="red">用来调节用于执行剔除的前一个级联的半径</font>。

Unity在剔除时是相当保守的，但是我们应该用级联淡出的比例来减少它，并多加一点，以确保过渡区域的阴影投射者不会被剔除。所以让我们用0.8减去渐变范围，最小为0。

如果你看到层叠过渡周围的阴影出现了洞，那么它必须进一步减少。

```c#
	float cullingFactor =
			Mathf.Max(0f, 0.8f - settings.directional.cascadeFade);
		
		for (int i = 0; i < cascadeCount; i++) {
			…
			splitData.shadowCascadeBlendCullingFactor = cullingFactor;
			…
		}
```

### 5、Transparency（透明度）

我们将通过考虑透明阴影投射器来结束本教程。clip、淡化和透明材质都可以像不透明材质一样接收阴影，但目前只有clip材质可以自己投下正确的阴影。透明物体的行为就像它们是 solid阴影投射器一样。

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/clipped.png" alt="clipped" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/transparent.png" alt="transparent" style="zoom:50%;" /></center>

<center>Clipped and transparent with shadows.</center>

#### [5.1、Shadow Modes](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#5.1)

我们有几种方法可以修改阴影投射器。由于涉及到向<font color="green">depth buffer</font>的写入，我们的阴影是二进制的，要么存在，要么不存在，但这仍然给我们一些灵活性。

它们可以被打开并完全实体化，也可以被剪切，抖动，或者完全关闭。我们可以做到这一点，独立于其他的材质属性，以支持最大的灵活性。

所以让我们为它添加一个单独的<font color="red">_Shadows shader</font>属性。我们可以使用<font color="green">KeywordEnum</font>属性来为它创建一个关键词下拉菜单，默认情况下阴影是打开的。

```c#
	[KeywordEnum(On, Clip, Dither, Off)] _Shadows ("Shadows", Float) = 0
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/shadow-modes.png" alt="img" style="zoom:80%;" />

<center>Shadows enabled.</center>

为这些模式添加一个着色器特性，取代现有的<font color="RoyalBlue">CLIPPING</font>特性。我们只需要三个变体，不使用关键字来表示开和关，<font color="green">SHADOWS_CLIP</font>，和<font color="green">_SHADOWS_DITHER</font>。

```glsl
	//#pragma shader_feature _CLIPPING
			#pragma shader_feature _ _SHADOWS_CLIP _SHADOWS_DITHER
```

在<font color="red">CustomShaderGUI</font>中为阴影创建一个setter属性。

```c#
enum ShadowMode {
		On, Clip, Dither, Off
	}

	ShadowMode Shadows {
		set {
			if (SetProperty("_Shadows", (float)value)) {
				SetKeyword("_SHADOWS_CLIP", value == ShadowMode.Clip);
				SetKeyword("_SHADOWS_DITHER", value == ShadowMode.Dither);
			}
		}
	}
```

然后在预设方法中适当地设置阴影。这将是对不透明的开启，对剪辑的剪辑，让我们对褪色和透明都使用抖动。

#### [5.2、Clipped Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#5.2)

在<font color="red">ShadowCasterPassFragment</font>中，用<font color="blue">SHADOWS_CLIP</font>_的检查来代替<font color="blue">CLIPPED</font>_的检查。

```glsl
	#if defined(_SHADOWS_CLIP)
		clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
	#endif
```

现在可以给透明材料提供剪切的阴影，这可能适合于有大部分完全不透明或透明但需要alpha混合的部分的表面。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/transparent-clipped-shadows.png" alt="img" style="zoom:50%;" />

<center>Transparent with clipped shadows.</center>

#### 5.3、[Dithered Shadows（朦胧的阴影）](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#5.3)

<font color="red">Dithered阴影</font> 的工作方式与<font color="red">clipped阴影</font> 一样，只是标准不同。在这种情况下，我们从表面的<font color="green">Alpha</font>中减去一个<font color="green">dither</font>值，然后在此基础上进行剪辑。我们可以再次使用<font color="RoyalBlue">InterleavedGradientNoise</font>函数。

<font color="red">InterleavedGradientNoise</font> 函数，它能在<font color="red">屏幕空间的XY位置生成一个旋转的平铺的抖动图案。</font> 

```c#
	#if defined(_SHADOWS_CLIP)
		clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
	#elif defined(_SHADOWS_DITHER)
		float dither = InterleavedGradientNoise(input.positionCS.xy, 0);
		clip(base.a - dither);
	#endif
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/dithered-shadows.png" alt="img" style="zoom:50%;" />

 <center>Dithered shadows.</center>

抖动可以用来接近半透明的阴影投射器，但这是一个相当粗糙的方法。硬抖动的阴影看起来很糟糕，但当使用较大的PCF滤波器时，它看起来可能是可以接受的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/dithered-pcf7.png" alt="img" style="zoom:50%;" />

<center>Dithered with PCF7x7.</center>

因为每个 texel的<font color="green"> dither pattern</font>是固定的，重叠的半透明shadow casters不会投射出一个组合的暗影。这种效果和最不透明的阴影投射器一样强。另外，由于产生的图案是有噪声的，当阴影矩阵发生变化时，它受到的时间伪影的影响更大，这可能会使阴影出现颤动( tremble)。只要物体不移动，这种方法对其他有固定投影的光线类型效果更好。对于半透明的物体，通常更实际的做法是使用 clipped的阴影或根本没有阴影。

#### 5.4、[No Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#5.4)

我们可以通过调整一个物体的<font color="green">MeshRenderer</font>组件的<font color="RoyalBlue">Cast Shadows</font>设置来关闭每个物体的阴影投射。

然而，如果你想为所有使用相同材质的物体关闭阴影，这并不实际，所以我们也将支持为每个材质关闭阴影。我们通过禁用材质的<font color="green">ShadowCaster</font>通道来做到这一点。

在<font color="red">CustomShaderGUI</font>中添加一个<font color="blue">SetShadowCasterPass</font>方法，首先检查<font color="red">_Shadows</font>着色器属性是否存在。

如果存在，也要通过其<font color="green">hasMixedValue</font>属性检查所有被选中的材质是否被设置为相同的模式。如果没有模式或者是混合模式，那么就放弃。否则，通过调用<font color="red">SetShaderPassEnabled</font> ，对所有材质启用或禁用<font color="green">ShadowCaster</font>通道，并将通道名称和启用状态作为参数。

```c#
	void SetShadowCasterPass () {
		MaterialProperty shadows = FindProperty("_Shadows", properties, false);
		if (shadows == null || shadows.hasMixedValue) {
			return;
		}
		bool enabled = shadows.floatValue < (float)ShadowMode.Off;
		foreach (Material m in materials) {
			m.SetShaderPassEnabled("ShadowCaster", enabled);
		}
	}
```

最简单的方法是，当材质通过GUI被改变时，总是调用<font color="red">SetShadowCasterPass</font> 来确保<font color="red">Pass</font>被正确设置。

我们可以通过在<font color="green">OnGUI</font>开始时调用<font color="green">EditorGUI.BeginChangeCheck</font>和在其结束时调用<font color="green">EditorGUI.EndChangeCheck</font>来实现。后面的方法返回自我们开始检查以来是否有变化。如果是的话，就设置<font color="green">shadow caster pass.。</font>

[Material.SetShaderPassEnabled](https://docs.unity3d.com/ScriptReference/Material.SetShaderPassEnabled.html) 在每个材质级别启用或禁用着色器通道。

```c#
	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		EditorGUI.BeginChangeCheck();
		…
		if (EditorGUI.EndChangeCheck()) {
			SetShadowCasterPass();
		}
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/not-casting-shadows.png" alt="img" style="zoom:50%;" />

<center>Not casting shadows.</center>

#### 5.5、Unlit Shadow Casters

虽然非光照材质不受光照影响，但你可能希望它们能投射阴影。我们可以通过简单地将Lit中的<font color="green">ShadowCaster</font>通道复制到<font color="RoyalBlue">Unlit Shader</font>中来支持。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/unlit-shadow-casters.png" alt="img" style="zoom:50%;" />

<font color="green">Unlit but casting shadows.</font>

#### 5.6、[Receiving Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/#5.6)

最后，我们还可以让发光的表面忽略阴影，这对<font color="red">holograms</font>等东西可能很有用，或者只是出于艺术目的。在<font color="green">Lit</font>上添加一个<font color="green">_RECEIVE_SHADOWS</font>的关键字切换属性来实现。

```c#
	[Toggle(_RECEIVE_SHADOWS)] _ReceiveShadows ("Receive Shadows", Float) = 1
```

加上<font color="green">CustomLit pass</font>	中的附带<font color="green">Shader</font>功能。

```glsl
	#pragma shader_feature _RECEIVE_SHADOWS
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/receiving-shadows.png" alt="img" style="zoom:50%;" />

<center>Receiving shadows.</center>

我们要做的就是在定义关键词时，在<font color="red">GetDirectionalShadowAttenuation</font>中强制将阴影衰减改为1。

```c#
float GetDirectionalShadowAttenuation (…) {
	#if !defined(_RECEIVE_SHADOWS)
		return 1.0;
	#endif
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-shadows/transparency/not-receiving-shadows.png" alt="img" style="zoom:50%;" />

<center>Casting but not receiving shadows.</center>
