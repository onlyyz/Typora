# 11_Post Processing

创建一个简单的后期特效堆栈。
		改动渲染的图像。
		在需要时进行后期处理。
		做一个艺术化的绽放效果。

![img](./../../Typora-Note/assets/tutorial-image-1677834262023-50.jpg)

<center>Make it glow!</center>

### 1、Post-FX Stack

大多数时候，渲染的图像并不是按原样显示的。图像是经过后期处理的，得到各种效果--简称FX--应用在上面。常见的特效包括<font color="red">bloom, color grading, depth-of-field, motion blur, and tone mapping.- 绽放、调色、景深、运动模糊和色调映射</font>。这些特效是以叠加的方式应用的，一个在另一个之上。在本教程中，我们将创建一个简单的后期特效堆栈，最初只支持bloom。

#### 1.1、[Settings Asset](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.1)

一个项目可能需要多个<font color="DarkOrchid ">Post-FX</font>堆栈配置，所以我们首先创建一个<font color="green">PostFXSettings</font>资产类型来存储一个<font color="red">stack堆栈</font>的设置。

```c#
using UnityEngine;

[CreateAssetMenu(menuName = "Rendering/Custom Post FX Settings")]
public class PostFXSettings : ScriptableObject { }
```

在本教程中，我们将使用一个单一的堆栈，通过向<font color="green">CustomRenderPipelineAsset</font>添加一个配置选项，将其传递给RP的构造函数，使其可用。

```c#
	[SerializeField]
	PostFXSettings postFXSettings = default;

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows, postFXSettings
		);
	}
```

然后，<font color="red">CustomRenderPipeline</font>必须跟踪FX设置，并在渲染过程中把它们和其他设置一起传递给<font color="green"> camera renderer</font>。

```c#
	PostFXSettings postFXSettings;

	public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		bool useLightsPerObject, ShadowSettings shadowSettings,
		PostFXSettings postFXSettings
	) {
		this.postFXSettings = postFXSettings;
		…
	}

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings, postFXSettings
			);
		}
	}
```

<font color="red">CameraRenderer.Render</font>最初对设置不做任何处理，因为我们还没有一个堆栈。

```c#
public void Render (
		ScriptableRenderContext context, Camera camera,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings, PostFXSettings postFXSettings
	) { … }
```

现在我们可以创建一个空的FX后设置资产，并将其分配给<font color="DarkOrchid "> pipeline asset.</font>。

<img src=".\..\..\Typora-Note\assets\post-fx-settings-assigned.png" alt="img" style="zoom:50%;" />

<center>Assigned post FX settings.</center>

#### 1.2、[Stack Object](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.2)

我们将对堆栈使用与<font color="green"> **Lighting** and **Shadows**.</font>相同的方法。我们为它创建一个类，记录<font color="DarkOrchid ">buffer, context, camera, and post-FX settings - 缓冲区、上下文、摄像机和后期特效设置</font>，用一个公共的<font color="red">Setup</font>方法来初始化它们。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class PostFXStack {

	const string bufferName = "Post FX";

	CommandBuffer buffer = new CommandBuffer {
		name = bufferName
	};

	ScriptableRenderContext context;
	
	Camera camera;

	PostFXSettings settings;

	public void Setup (
		ScriptableRenderContext context, Camera camera, PostFXSettings settings
	) {
		this.context = context;
		this.camera = camera;
		this.settings = settings;
	}
}
```

接下来，添加一个公共属性来指示堆栈是否被激活，只有在有设置的情况下才是如此。我们的想法是，如果没有提供设置，<font color="DarkOrchid "> post-processing</font>应该被跳过。

```c#
	public bool IsActive => settings != null;
```

而我们需要的最后一部分是一个公共的<font color="red">Render</font>方法，用来渲染堆栈。

通过使用适当的<font color="green">Shader</font>，简单地绘制一个覆盖整个图像的矩形，就可以在整个图像上应用一个效果。

现在我们没有着色器，所以我们将简单地把渲染到这一点的东西复制到<font color="red">camera's frame buffer</font>。这可以通过在<font color="green"> command buffer</font>上调用Blit来实现，并将<font color="DarkOrchid "> source and destination</font>的标识符传递给它。

这些标识符可以用多种格式提供。我们将使用一个整数作为源，我们将为它添加一个参数，并使用<font color="red">BuiltinRenderTextureType.CameraTarget</font>作为目标。然后我们执行并<font color="green"> clear the buffer.</font>

```c#
public void Render (int sourceId) {
		buffer.Blit(sourceId, BuiltinRenderTextureType.CameraTarget);
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

在这种情况下，我们不需要手动开始和结束缓冲区采样，因为我们不需要调用<font color="red">ClearRenderTarget</font>，因为我们完全替换了目的地的东西。

#### 1.3、[Using the Stack](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.3)

<font color="red">CameraRenderer</font>现在需要一个<font color="RoyalBlue"> stack instance </font>，并在<font color="red">Render</font>中对其调用<font color="green">Setup</font>，就像对其<font color="green">Lighting</font>对象一样。

```c#
	Lighting lighting = new Lighting();

	PostFXStack postFXStack = new PostFXStack();

	public void Render (…) {
		…
		lighting.Setup(
			context, cullingResults, shadowSettings, useLightsPerObject
		);
		postFXStack.Setup(context, camera, postFXSettings);
		buffer.EndSample(SampleName);
		Setup();
		…
	}
```

到此为止，我们总是直接渲染到<font color="green"> camera's frame buffer</font>，这些缓冲区要么是用于显示的，要么是配置的渲染纹理。我们对这些没有直接的控制，只应该向它们写入。

所以为了给<font color="red">active stack</font>提供一个<font color="RoyalBlue">source texture </font>，我们必须使用一个<font color="green">render texture </font>作为<font color="red">intermediate frame buffer for the camera - 摄像机的中间帧缓冲区</font>。获取一个渲染纹理并将其设置为渲染目标，就像<font color="RoyalBlue">shadow maps</font>一样，只不过我们将使用<font color="red">RenderTextureFormat.Default</font>格式。在我们清除渲染目标之前做这个。

```c#
	static int frameBufferId = Shader.PropertyToID("_CameraFrameBuffer");
	
	…
	
	void Setup () {
		context.SetupCameraProperties(camera);
		CameraClearFlags flags = camera.clearFlags;

		if (postFXStack.IsActive) {
			buffer.GetTemporaryRT(
				frameBufferId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Bilinear, RenderTextureFormat.Default
			);
			buffer.SetRenderTarget(
				frameBufferId,
				RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
			);
		}

		buffer.ClearRenderTarget(…);
		buffer.BeginSample(SampleName);
		ExecuteBuffer();
	}
```

如果我们有一个活动的堆栈，还要添加一个<font color="red">Cleanup method</font>来释放纹理。我们也可以把<font color="RoyalBlue"> lighting cleanup</font>移到这里。

```c#
	void Cleanup () {
		lighting.Cleanup();
		if (postFXStack.IsActive) {
			buffer.ReleaseTemporaryRT(frameBufferId);
		}
	}
```

在提交前的<font color="green">Render</font>结束时调用<font color="green">Cleanup</font>。在这之前直接<font color="DarkOrchid ">render the stack</font>，如果它是<font color="red">active.</font>。

```c#
	public void Render (…) {
		…
		DrawGizmos();
		if (postFXStack.IsActive) {
			postFXStack.Render(frameBufferId);
		}
		Cleanup();
		//lighting.Cleanup();
		Submit();
	}
```

在这一点上，结果看起来应该没有什么不同，但是增加了一个额外的绘制步骤，从中间缓冲区复制到<font color="red"> final frame buffer</font>。它在帧调试器中被列为<font color="red">**Draw Dynamic**.</font>

<img src=".\..\..\Typora-Note\assets\rendering.png" alt="img" style="zoom:50%;" />

<center>Rendering the FX stack.</center>

#### 1.4、Forced Clearing

当绘制到一个<font color="green">intermediate frame buffer - 中间帧缓冲区</font>时，我们渲染到一个充满任意数据的纹理。当帧调试器处于活动状态时，你可以看到这一点。Unity确保帧调试器在每一帧开始时得到一个清晰的帧缓冲区，但是当渲染到<font color="DarkOrchid ">our own texture</font>时，我们绕过了这一点。

这通常会导致我们在前一帧的结果之上进行绘制，但这并不保证。

如果摄像机的<font color="red">Clear Flags</font>设置为<font color="RoyalBlue">sky box or a solid color</font>，这并不重要，因为我们可以保证完全覆盖之前的数据。但是另外两个选项就不行了。

为了防止随机的结果，当堆栈处于活动状态时总是要清除深度，同时也要清除颜色，除非使用天空盒。

在<font color="green">Setup</font>中设置它

```c#
	CameraClearFlags flags = camera.clearFlags;

		if (postFXStack.IsActive) {
			if (flags > CameraClearFlags.Color) {
				flags = CameraClearFlags.Color;
			}
			…
		}

		buffer.ClearRenderTarget(…);
```

请注意，这使得在使用<font color="red">post-FX stack</font>时，不可能让一个摄像机在另一个摄像机上面渲染而不被清除。有一些方法可以解决这个问题，但这不在本教程的范围之内。

#### 1.5、Gizmos

我们目前是在同一时间绘制所有的<font color="green">gizmos</font>，但是那些应该在后期特效之前和之后渲染的<font color="green">gizmos</font>是有区别的。所以让我们把<font color="red">DrawGizmos</font>方法一分为二。

```c#
	partial void DrawGizmosBeforeFX ();

	partial void DrawGizmosAfterFX ();
	
	…
	
#if UNITY_EDITOR
	
	…
						
	partial void DrawGizmosBeforeFX () {
		if (Handles.ShouldRenderGizmos()) {
			context.DrawGizmos(camera, GizmoSubset.PreImageEffects);
			//context.DrawGizmos(camera, GizmoSubset.PostImageEffects);
		}
	}

	partial void DrawGizmosAfterFX () {
		if (Handles.ShouldRenderGizmos()) {
			context.DrawGizmos(camera, GizmoSubset.PostImageEffects);
		}
	}
```

然后我们就可以在<font color="green">Render</font>中以正确的时间绘制它们。

```c#
	//DrawGizmos();
		DrawGizmosBeforeFX();
		if (postFXStack.IsActive) {
			postFXStack.Render(frameBufferId);
		}
		DrawGizmosAfterFX();
```

注意，当3D图标被用于小工具时，当堆栈处于活动状态时，它们不再被物体遮挡。发生这种情况是因为场景窗口依赖于原始帧缓冲区的<font color="green">depth data</font>，而我们并没有使用这个数据。我们将在将来讨论深度与后期特效的结合

![with](.\..\..\Typora-Note\assets\gizmos-with-fx-1677957598392-7.png) ![without](.\..\..\Typora-Note\assets\gizmos-without-fx-1677957598393-9.png)

<center>3D gizmos with and without FX.</center>

#### 1.6、[Custom Drawing](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.6)

我们目前使用的Blit方法是绘制一个四边形网格--两个三角形--覆盖整个屏幕空间。

但我们可以通过只画一个三角形来获得同样的结果，这样做的工作量会小一些。我们甚至不需要向<font color="green">GPU</font>发送<font color="DarkOrchid ">single-triangle mesh</font>，我们可以按程序生成它。

<!--<u>*因为GPU在小块中平行渲染片段，一些片段最终会沿着三角形的边缘浪费掉。由于四边形有两个三角形，沿对角线的碎片块被渲染了两次，这是不高效的。除此之外，渲染单个三角形可以有更好的本地缓存一致性。*</u>-->

在<font color="red"> our RP's *Shaders*</font>文件夹中创建一个<font color="DarkOrchid ">PostFXStackPasses.hlsl</font>文件。我们将把<font color="red">passes of our stack </font>放在那里。我们将在其中定义的第一件事是<font color="green">Varyings</font>结构，它只需要包含<font color="green">裁剪空间的位置和屏幕空间的UV坐标</font>。

```c#
#ifndef CUSTOM_POST_FX_PASSES_INCLUDED
#define CUSTOM_POST_FX_PASSES_INCLUDED

struct Varyings {
	float4 positionCS : SV_POSITION;
	float2 screenUV : VAR_SCREEN_UV;
};

#endif
```

接下来，创建一个默认的顶点传递，只有一个<font color="green">Vertex identifier</font>作为参数。它是一个<font color="green">unsigned</font>--uint--具有<font color="RoyalBlue">SV_VertexID</font>语义。

使用这个ID来生成顶点位置和UV坐标。X坐标是-1，-1，3；Y坐标是-1，3，-1。为了使可见的UV坐标覆盖0-1的范围，对U使用0, 0, 2，对V使用0, 2, 0。

<img src=".\..\..\Typora-Note\assets\clip-space-triangle.png" alt="img" style="zoom:50%;" />

<center> Triangle covering clip space.</center>

```c#
Varyings DefaultPassVertex (uint vertexID : SV_VertexID) {
	Varyings output;
	output.positionCS = float4(
		vertexID <= 1 ? -1.0 : 3.0,
		vertexID == 1 ? 3.0 : -1.0,
		0.0, 1.0
	);
	output.screenUV = float2(
		vertexID <= 1 ? 0.0 : 2.0,
		vertexID == 1 ? 2.0 : 0.0
	);
	return output;
}
```

增加一个用于简单复制的<font color="green">fragment pass</font>，使其最初返回UV坐标，以便调试。

```c#
float4 CopyPassFragment (Varyings input) : SV_TARGET {
	return float4(input.screenUV, 0.0, 1.0);
}
```

在同一个文件夹中创建一个配套的着色器文件。所有的<font color="green">passes </font>都将使用<font color="DarkOrchid "> no culling and ignore depth</font>，所以我们可以把这些指令直接放在<font color="RoyalBlue">Subshader</font>块中。我们也总是包括我们的<font color="green">Common</font>和<font color="green">PostFXStackPasses</font>文件。

它现在唯一的通道是用于复制，使用我们创建的顶点和片段函数。我们还可以通过使用<font color="red">Name</font>指令给它起个名字，这在同一着色器中组合多个pass时很方便，因为帧调试器会用它来做<font color="green">pass</font>标签而不是数字。

最后，把它的菜单项放在隐藏文件夹下，这样在选择<font color="green">shader for a material.</font>时，它就不会显示出来。

```c#
Shader "Hidden/Custom RP/Post FX Stack" {
	
	SubShader {
		Cull Off
		ZTest Always
		ZWrite Off
		
		HLSLINCLUDE
		#include "../ShaderLibrary/Common.hlsl"
		#include "PostFXStackPasses.hlsl"
		ENDHLSL

		Pass {
			Name "Copy"
			
			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment CopyPassFragment
			ENDHLSL
		}
	}
}
```

我们将简单地通过设置手动将着色器链接到我们的堆栈。

```c#
public class PostFXSettings : ScriptableObject {

	[SerializeField]
	Shader shader = default;
}
```

<img src=".\..\..\Typora-Note\assets\post-fx-shader.png" alt="img" style="zoom:50%;" />

<center>Post FX shader assigned.</center>

但是我们在渲染时需要一个<font color="green">material </font>，所以添加一个公共属性，我们可以用它来直接从设置资产中获得一个<font color="green">material </font>。我们将按需创建它，并设置为隐藏，不保存在项目中。另外，<font color="green">material </font>不能和<font color="green">Asset</font>一起被序列化，因为它是按需创建的。

```c#
[System.NonSerialized]
	Material material;

	public Material Material {
		get {
			if (material == null && shader != null) {
				material = new Material(shader);
				material.hideFlags = HideFlags.HideAndDontSave;
			}
			return material;
		}
	}
```

由于用名字而不是数字来称呼<font color="green">passes</font>很方便，所以在<font color="red">PostFXStack</font>里面创建一个<font color="green">Pass enum </font>，最初只包含<font color="green">copy pass.</font>。

```c#
enum Pass {
		Copy
	}
```

现在我们可以定义我们自己的<font color="green">Draw</font>方法。给它两个<font color="red">RenderTargetIdentifier</font>参数，以表明应该从哪里和向哪里绘制，外加一个<font color="RoyalBlue">pass</font>参数。

在这个方法中，通过<font color="red">_PostFXSource texture</font>使<font color="green">source</font>可用，像以前一样使用目标点作为渲染目标，然后绘制三角形。

我们通过在缓冲区上调用<font color="red">DrawProcedural</font>来做到这一点，参数是一个未使用的<font color="RoyalBlue">unused matrix, the stack material and the pass</font>。在这之后，还有两个参数。首先是我们要绘制的形状，也就是<font color="green">MeshTopology.Triangles</font>。第二是我们想要多少个顶点，一个三角形需要三个顶点。



[CommandBuffer](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.html).SetGlobal[Texture](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.SetGlobalTexture.html) 设置图片ID、ID to GPU

[CommandBuffer](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.html).D[rawProcedural](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.DrawProcedural.html)  draw procedural geometry

```c#
int fxSourceId = Shader.PropertyToID("_PostFXSource");
	
	…
	
	void Draw (
		RenderTargetIdentifier from, RenderTargetIdentifier to, Pass pass
	) {
		buffer.SetGlobalTexture(fxSourceId, from);
		buffer.SetRenderTarget(
			to, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
		buffer.DrawProcedural(
			Matrix4x4.identity, settings.Material, (int)pass,
			MeshTopology.Triangles, 3
		);
	}
```

最后，用我们自己的方法替换对<font color="green">Blit</font>的调用。

```c#
	//buffer.Blit(sourceId, BuiltinRenderTextureType.CameraTarget);
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
```

#### 1.7、[Don't Always Apply FX](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.7)

现在我们应该看到屏幕空间的UV坐标出现在场景窗口。并在游戏窗口中。而且还出现在材质预览中，甚至出现在反射探测器中，一旦它们刷新。

<img src=".\..\..\Typora-Note\assets\reflection-probe-fx.png" alt="img" style="zoom:50%;" />

<center>Reflection probe with FX applied.</center>

我们的想法是，<font color="green">post FX</font>要应用于适当的摄像机，而不是其他相机。我们可以通过检查<font color="green">PostFXStack.Setup</font>中是否有游戏或场景摄像机来执行这一点。如果没有，我们就把设置设为<font color="RoyalBlue">null</font>，这样就会停用该摄像机的堆栈。

```c#
	this.settings =
			camera.cameraType <= CameraType.SceneView ? settings : null;
```

除此之外，还可以在场景窗口中通过其工具栏中的效果下拉菜单来切换<font color="DarkOrchid ">post-processing</font>。有可能同时打开多个场景窗口，这些窗口可以单独启用或禁用后期效果。

为了支持这一点，我们为<font color="green">PostFXStack</font>创建了一个<font color="RoyalBlue"> editor partial class - 编辑器局部类</font>，它有一个<font color="green">ApplySceneViewState</font>方法，在构建时不做任何事情。

它的编辑器版本会检查我们是否在处理一个场景视图的摄像机，如果是的话，如果当前绘制的场景视图的状态已经<font color="DarkOrchid ">image effects disabled</font>，则<font color="red"> disables the stack</font>。

```c#
using UnityEditor;
using UnityEngine;

partial class PostFXStack {

	partial void ApplySceneViewState ();

#if UNITY_EDITOR

	partial void ApplySceneViewState () {
		if (
			camera.cameraType == CameraType.SceneView &&
			!SceneView.currentDrawingSceneView.sceneViewState.showImageEffects
		) {
			settings = null;
		}
	}

#endif
}
```

在<font color="green">Setup</font>结束时调用此方法。

```c#
public partial class PostFXStack {

	…

	public void Setup (…) {
		…
		ApplySceneViewState();
	}
```

#### 1.8、[Copying](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#1.8)

我们通过使用<font color="green">our copy pass</font>返回<font color="red">source color</font>来完成堆栈。为此创建一个<font color="red">GetSource</font>函数，它将进行采样。我们将始终使用一个<font color="RoyalBlue">linear clamp sampler - 线性钳制采样器</font>，所以我们可以明确地声明这一点。

```c#
TEXTURE2D(_PostFXSource);
SAMPLER(sampler_linear_clamp);

float4 GetSource(float2 screenUV) {
	return SAMPLE_TEXTURE2D(_PostFXSource, sampler_linear_clamp, screenUV);
}

float4 CopyPassFragment (Varyings input) : SV_TARGET {
	return GetSource(input.screenUV);
}
```

因为我们的<font color="green">buffer </font>永远不会有<font color="RoyalBlue">mip maps</font>，我们可以通过用<font color="RoyalBlue">SAMPLE_TEXTURE2D</font>替换<font color="green">SAMPLE_TEXTURE2D_LOD</font>来回避自动选择<font color="green">mip map</font>，增加一个额外的参数来强制选择<font color="red">mip map level zero</font>。

```c#
	return SAMPLE_TEXTURE2D_LOD(_PostFXSource, sampler_linear_clamp, screenUV, 0);
```

我们最终得到了原始图像，但在某些情况下它是颠倒的，通常是在场景窗口中。这取决于<font color="green">图形API</font>以及<font color="green">source and destination</font>的类型。

发生这种情况是因为<font color="red">graphics APIs </font>让纹理的V坐标从顶部开始，而另一些<font color="red">graphics APIs </font>则让它从底部开始。Unity通常会隐藏这一点，但不能在所有涉及渲染纹理的情况下这样做。幸运的是，Unity通过<font color="green">_ProjectionParams</font>向量的X分量指示是否需要手动翻转，我们应该在<font color="green">UnityInput</font>中定义。

```c#
float4 _ProjectionParams;
```

如果数值是负的，我们必须翻转<font color="green">DefaultPassVertex</font>中的V坐标。

```c#
Varyings DefaultPassVertex (uint vertexID : SV_VertexID) {
	…
	if (_ProjectionParams.x < 0.0) {
		output.screenUV.y = 1.0 - output.screenUV.y;
	}
	return output;
}
```

### 2、Bloom

<font color="DarkOrchid ">The bloom post effect</font>是用来使事物发光的。

这是有物理学基础的，但经典的<font color="DarkOrchid ">bloom effect </font>是艺术性的，而不是现实的。非现实的<font color="green">bloom effect </font>是非常明显的，因此是一个很好的效果，可以证明我们的后期特效堆栈的工作。

我们将在下一个教程中研究<font color="green"> realistic bloom</font>，届时我们将介绍<font color="red">HDR渲染</font>。现在，我们的目标是实现<font color="green">LDR Bloom</font>的发光效果。

#### 2.1、Bloom Pyramid

<font color="green">Bloom</font>代表颜色的<font color="red"> scattering - 散射</font>，这可以通过模糊图像来实现。明亮的像素会渗入相邻的较暗的像素中，从而看起来像发光。模糊一个纹理的最简单和最快的方法是将它复制到另一个宽度和高度为一半的纹理上。复制过程中的每个样本最终都会在四个源像素之间取样。

通过<font color="green">bilinear filtering</font>，这就是2×2像素的平均块。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/2x2-bilinear-downsampling.png)

<center>Bilinear downsampling 4×4 to 2×2.</center>

这样做一次只能模糊一点。因此，我们重复这个过程，逐步降低采样，直到一个理想的水平，有效地建立一个纹理的金字塔。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/texture-pyramid.png)

<center>Pyramid with four textures, halving dimensions each level.</center>

我们需要跟踪堆栈中的纹理，但有多少纹理取决于金字塔有多少层，这取决于源图像的大小。让我们在<font color="green">PostFXStack</font>中定义最多16个级别，这足以将65,536×65,526的纹理一直缩减到一个像素。

```c#
	const int maxBloomPyramidLevels = 16;
```

为了跟踪金字塔中的纹理，我们需要<font color="green">texture identifiers</font>。我们将使用属性名称<font color="red">_BloomPyramid0、_BloomPyramid1</font>，以此类推。但我们不要明确地写出所有这16个名字。相反，我们将在一个构造方法中获得这些标识符，并且只跟踪第一个标识符。

这样做是因为<font color="DarkOrchid ">Shader.PropertyToID</font>只是按照请求新属性名称的顺序依次分配标识符。我们只需要确保所有的标识符都是一次性请求的，因为在编辑器和构建中，每个应用会话的数量都是固定的。

```c#
int bloomPyramidId;
	
	…
	
	public PostFXStack () {
		bloomPyramidId = Shader.PropertyToID("_BloomPyramid0");
		for (int i = 1; i < maxBloomPyramidLevels; i++) {
			Shader.PropertyToID("_BloomPyramid" + i);
		}
	}
```

现在创建一个<font color="green">DoBloom</font>方法，为一个给定的<font color="green"> source identifier.</font>应用<font color="DarkOrchid "> bloom effect</font>。

首先，将相机的像素宽度和高度减半，并选择默认的<font color="RoyalBlue">render texture format - 渲染纹理格式</font>。最初我们将从<font color="green">source</font>复制到金字塔的第一个纹理。追踪这些标识符。

```c#
	void DoBloom (int sourceId) {
		buffer.BeginSample("Bloom");
		int width = camera.pixelWidth / 2, height = camera.pixelHeight / 2;
		RenderTextureFormat format = RenderTextureFormat.Default;
		int fromId = sourceId, toId = bloomPyramidId;
		buffer.EndSample("Bloom");
	}
```

然后在所有的金字塔级别中循环。

每次迭代，首先检查一个层次是否会变质。如果是，我们就在这一点上停止。如果没有，就得到一个新的渲染纹理，复制到它，使其成为新的<font color="green">source</font>，增加目标，并再次将尺寸减半。在循环外声明循环迭代器变量，因为我们以后需要它。

```c#
	int fromId = sourceId, toId = bloomPyramidId;

		int i;
		for (i = 0; i < maxBloomPyramidLevels; i++) {
			if (height < 1 || width < 1) {
				break;
			}
			buffer.GetTemporaryRT(
				toId, width, height, 0, FilterMode.Bilinear, format
			);
			Draw(fromId, toId, Pass.Copy);
			fromId = toId;
			toId += 1;
			width /= 2;
			height /= 2;
		}
```

一旦金字塔完成，就把最终结果复制到相机目标上。然后递减迭代器并向后循环，释放所有我们要求的纹理。

```c#
	for (i = 0; i < maxBloomPyramidLevels; i++) { … }

		Draw(fromId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);

		for (i -= 1; i >= 0; i--) {
			buffer.ReleaseTemporaryRT(bloomPyramidId + i);
		}
		buffer.EndSample("Bloom");
```

现在我们可以在<font color="green">Render</font>中用<font color="red">bloom效果</font>代替简单的拷贝。

```
	public void Render (int sourceId) {
		//Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
		DoBloom(sourceId);
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

#### 2.2、[Configurable Bloom](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.2)

我们现在模糊了很多，最终的结果几乎是一致的。你可以通过帧调试器检查中间的步骤。这些步骤作为终点显得更加有用，所以我们要让它有可能提前停止。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/progressive-downsampling.png)



<center>Three iterations of progressive downsampling.</center>

我们可以通过两种方式做到这一点。首先，我们可以限制模糊迭代的数量。第二，我们可以将降级限制设置为一个更高的值。让我们支持这两种方法，在<font color="green">PostFXSettings</font>中添加一个<font color="red">BloomSettings</font>配置结构，为它们提供选项。通过一个<font color="green">getter</font>属性使其公开可用。

```c#
	[System.Serializable]
	public struct BloomSettings {

		[Range(0f, 16f)]
		public int maxIterations;

		[Min(1f)]
		public int downscaleLimit;
	}

	[SerializeField]
	BloomSettings bloom = default;

	public BloomSettings Bloom => bloom;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/bloom-settings.png" alt="img" style="zoom:50%;" />

<center>Default bloom settings.</center>

让<font color="green">PostFXStack.DoBloom</font>使用这些设置来限制自己。

```c
	PostFXSettings.BloomSettings bloom = settings.Bloom;
		int width = camera.pixelWidth / 2, height = camera.pixelHeight / 2;
		RenderTextureFormat format = RenderTextureFormat.Default;
		int fromId = sourceId, toId = bloomPyramidId;

		int i;
		for (i = 0; i < bloom.maxIterations; i++) {
			if (height < bloom.downscaleLimit || width < bloom.downscaleLimit) {
				break;
			}
			buffer.GetTemporaryRT(
				toId, width, height, 0, FilterMode.Bilinear, format
			);
			…
		}

```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/down-3.png" alt="3 steps" style="zoom:50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/down-5.png" alt="5 steps" style="zoom:50%;" /></center>

<center>Progressive bilinear downsampling, 3 and 5 steps.</center>

#### 2.3、[Gaussian Filtering](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.3)

用一个小的2×2的<font color="green">filter</font>进行下采样会产生非常块的结果。通过使用更大的<font color="green">filter kernel</font>，例如近似的9×9的<font color="green">Gaussian filter</font>，效果可以得到很大的改善。如果我们将其与<font color="green">bilinear downsampling </font>结合起来，就会使其翻倍，有效地达到18×18。这就是<font color="RoyalBlue">Universal RP</font>和<font color="RoyalBlue">HDRP</font>用于它们的<font color="red">bloom</font>。

虽然这个操作混合了81个样本，但它是可分离的，这意味着它可以分成水平和垂直两部分，混合每一行或每一列的九个样本。因此，我们只需要采样18次，但每次迭代都要进行两次抽样。

让我们从水平方向的传递开始。在<font color="green">PostFXStackPasses</font>中为它创建一个新的<font color="red">BloomHorizontalPassFragment</font>函数。它以当前的UV坐标为中心，累积一排9个样本。我们还将同时进行<font color="RoyalBlue">downsample</font>，因此每个偏移步骤是<font color="DarkOrchid "> source texel </font>的两倍。从左边开始的样本权重是<font color="green">0.01621622，0.05405405，0.12162162，0.19459459</font>，然后中间是<font color="green">0.22702703</font>，另一边则是相反。

```c++
float4 _PostFXSource_TexelSize;

float4 GetSourceTexelSize () {
	return _PostFXSource_TexelSize;
}

…

float4 BloomHorizontalPassFragment (Varyings input) : SV_TARGET {
	float3 color = 0.0;
	float offsets[] = {
		-4.0, -3.0, -2.0, -1.0, 0.0, 1.0, 2.0, 3.0, 4.0
	};
	float weights[] = {
		0.01621622, 0.05405405, 0.12162162, 0.19459459, 0.22702703,
		0.19459459, 0.12162162, 0.05405405, 0.01621622
	};
	for (int i = 0; i < 9; i++) {
		float offset = offsets[i] * 2.0 * GetSourceTexelSize().x;
		color += GetSource(input.screenUV + float2(offset, 0.0)).rgb * weights[i];
	}
	return float4(color, 1.0);
}
```

也为它在<font color="red">PostFXStack</font>着色器中添加一个通道。我把它放在复制通道的上面，以保持它们的字母顺序。

```c#
Pass {
			Name "Bloom Horizontal"
			
			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment BloomHorizontalPassFragment
			ENDHLSL
		}
```

在<font color="red">PostFXStack.Pass</font>枚举中也添加一个条目，同样以相同的顺序。

```c#
enum Pass {
		BloomHorizontal,
		Copy
	}
```

现在我们可以在<font color="green">DoBloom</font>中<font color="green">downsampling step</font>时使用<font color="red">bloom-horizontal pass</font>。

```c#
			Draw(fromId, toId, Pass.BloomHorizontal);
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/down-horizontal-3.png" alt="3 steps" style="zoom:50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/down-horizontal-5.png" alt="5 steps" style="zoom:50%;" /></center>

在这一点上，结果显然是水平拉伸的，但它看起来很有希望。我们可以通过复制<font color="DarkOrchid ">BloomHorizontalPassFragment</font>，重命名它，并从行切换到列来创建垂直通道。我们在第一次传递中降低了采样率，但这次我们保持相同的尺寸来完成<font color="RoyalBlue">Gaussian filter</font>，所以文本大小的偏移不应该是双倍的。

```glsl
float4 BloomVerticalPassFragment (Varyings input) : SV_TARGET {
	float3 color = 0.0;
	float offsets[] = {
		-4.0, -3.0, -2.0, -1.0, 0.0, 1.0, 2.0, 3.0, 4.0
	};
	float weights[] = {
		0.01621622, 0.05405405, 0.12162162, 0.19459459, 0.22702703,
		0.19459459, 0.12162162, 0.05405405, 0.01621622
	};
	for (int i = 0; i < 9; i++) {
		float offset = offsets[i] * GetSourceTexelSize().y;
		color += GetSource(input.screenUV + float2(0.0, offset)).rgb * weights[i];
	}
	return float4(color, 1.0);
}
```

也要为它添加<font color="RoyalBlue">a pass and enum</font>。从现在起，我不会再显示这些步骤了。

我们现在需要在每个金字塔层的中间增加一个步骤，为此我们还必须保留<font color="green">texture identifiers</font>。

我们可以通过简单地将<font color="DarkOrchid ">PostFXStack</font>构造函数中的循环限制增加一倍来做到这一点。由于我们没有引入其他着色器的属性名称，所以标识符都会按顺序排列，否则就需要重新启动Unity。

```c#
	public PostFXStack () {
		bloomPyramidId = Shader.PropertyToID("_BloomPyramid0");
		for (int i = 1; i < maxBloomPyramidLevels * 2; i++) {
			Shader.PropertyToID("_BloomPyramid" + i);
		}
	}
```

在<font color="red">DoBloom</font>中，<font color="DarkOrchid "> destination identifier</font>现在必须从高处开始，在每个<font color="green">downsampling step</font>步骤后增加两个。然后中间的纹理可以放在中间。水平绘制到中间，接着是垂直绘制到目的地。我们还必须释放额外的纹理，最简单的方法是从最后一个金字塔源头开始倒退。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/hv-downsamling.png" alt="img" style="zoom:50%;" />



<center>Horizontal to next level, vertical at same level.</center>

```c#
	void DoBloom (int sourceId) {
		…
		int fromId = sourceId, toId = bloomPyramidId + 1;
		
		for (i = 0; i < bloom.maxIterations; i++) {
			…
			int midId = toId - 1;
			buffer.GetTemporaryRT(
				midId, width, height, 0, FilterMode.Bilinear, format
			);
			buffer.GetTemporaryRT(
				toId, width, height, 0, FilterMode.Bilinear, format
			);
			Draw(fromId, midId, Pass.BloomHorizontal);
			Draw(midId, toId, Pass.BloomVertical);
			fromId = toId;
			toId += 2;
			…
		}

		Draw(fromId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);

		for (i -= 1; i >= 0; i--) {
			buffer.ReleaseTemporaryRT(fromId);
			buffer.ReleaseTemporaryRT(fromId - 1);
			fromId -= 2;
		}
		buffer.EndSample("Bloom");
	}
```

<center><img src=".\..\..\Typora-Note\assets\down-gaussian-3.png" alt="3 steps" style="zoom:50%;" /> <img src=".\..\..\Typora-Note\assets\down-gaussian-5.png" alt="5 steps" style="zoom:50%;" /></center>

<center>Complete Gaussian, 3 and 5 steps.</center>

我们的<font color="green"> downsampling filter</font>现在完成了，看起来比简单的<font color="RoyalBlue">bilinear filtering</font>好很多，但代价是更多的<font color="green">texture samples</font>。

幸运的是，我们可以通过使用<font color="DarkOrchid ">bilinear filtering</font>在<font color="DarkOrchid ">Gaussian</font>取样点之间的适当偏移处取样来减少一点样本量。这就把九个样本减少到了五个。我们可以在<font color="RoyalBlue">BloomVerticalPassFragment</font>中使用这个技巧。在两个方向上的偏移量变为<font color="RoyalBlue">3.23076923和1.38461538</font>，权重为<font color="RoyalBlue">0.07027和0.31621622</font>。

```c#
	float offsets[] = {
		-3.23076923, -1.38461538, 0.0, 1.38461538, 3.23076923
	};
	float weights[] = {
		0.07027027, 0.31621622, 0.22702703, 0.31621622, 0.07027027
	};
	for (int i = 0; i < 5; i++) {
		float offset = offsets[i] * GetSourceTexelSize().y;
		color += GetSource(input.screenUV + float2(0.0, offset)).rgb * weights[i];
	}
```

我们不能在<font color="green">BloomHorizontalPassFragment</font>中这样做，因为我们已经在该通道中使用<font color="green">bilinear filtering</font>进行下采样了。它的九个样本中的每一个都是2×2的<font color="green">source pixels</font>的平均值。

#### 2.4、[Additive Blurring](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.4)

使用<font color="DarkOrchid ">bloom</font>金字塔的顶部作为最终图像，可以产生一个统一的混合，看起来不像是有什么东西在发光。我们可以通过对金字塔向下逐步<font color="red">upsampling</font>，将所有级别的图像累积到一个单一的图像中，来获得所需的结果。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/additive-progressive-upsampling.png" alt="img" style="zoom:50%;" />

<center>Additive progressive upsampling, reusing textures.</center>

我们可以使用加法混合来结合两张图片，但让我们对所有<font color="DarkOrchid ">passes</font>使用相同的混合模式，而不是添加第二个<font color="RoyalBlue"> source texture</font>。在<font color="green">PostFXStack</font>中为它申请一个标识符。

```c#
	int
		fxSourceId = Shader.PropertyToID("_PostFXSource"),
		fxSource2Id = Shader.PropertyToID("_PostFXSource2");
```

然后在<font color="green">DoBloom</font>中完成金字塔后不再直接执行最后的绘制。取而代之的是，释放最后一次迭代的<font color="DarkOrchid "> horizontal draw</font>所用的纹理，并将<font color="red">destination</font>设置为低一级的水平绘制所用的纹理。

```c#
//Draw(fromId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
		buffer.ReleaseTemporaryRT(fromId - 1);
		toId -= 5;
```

当我们<font color="DarkOrchid ">loop back </font>时，我们在每个迭代中再次画出，方向相反，每一级的结果作为第二个来源。这只在第一层之前有效，所以我们必须提前一步停止。之后，以<font color="RoyalBlue">original image</font>为第二<font color="RoyalBlue">source</font>，绘制到<font color="red"> final destination </font>。

```c#
for (i -= 1; i > 0; i--) {
			buffer.SetGlobalTexture(fxSource2Id, toId + 1);
			Draw(fromId, toId, Pass.Copy);
			buffer.ReleaseTemporaryRT(fromId);
			buffer.ReleaseTemporaryRT(toId + 1);
			fromId = toId;
			toId -= 2;
		}

		buffer.SetGlobalTexture(fxSource2Id, sourceId);
		Draw(fromId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
		buffer.ReleaseTemporaryRT(fromId);
		buffer.EndSample("Bloom");
```

为了让这个工作顺利进行，我们需要让<font color="DarkOrchid ">shader passes</font>可以使用辅助源。

```c#
TEXTURE2D(_PostFXSource);
TEXTURE2D(_PostFXSource2);
SAMPLER(sampler_linear_clamp);

…

float4 GetSource2(float2 screenUV) {
	return SAMPLE_TEXTURE2D_LOD(_PostFXSource2, sampler_linear_clamp, screenUV, 0);
}
```

并引入一个新的<font color="green">bloom combination pass</font>，对两种纹理进行采样和添加。和以前一样，我只展示了片段程序，没有展示新的着色器通道和新的枚举条目。

```c#
float4 BloomCombinePassFragment (Varyings input) : SV_TARGET {
	float3 lowRes = GetSource(input.screenUV).rgb;
	float3 highRes = GetSource2(input.screenUV).rgb;
	return float4(lowRes + highRes, 1.0);
}
```

在<font color="red">upsampling</font>时使用<font color="green">new pass </font>。

```c#
for (i -= 1; i > 0; i--) {
			buffer.SetGlobalTexture(fxSource2Id, toId + 1);
			Draw(fromId, toId, Pass.BloomCombine);
			…
		}

		buffer.SetGlobalTexture(fxSource2Id, sourceId);
		Draw(
			bloomPyramidId, BuiltinRenderTextureType.CameraTarget,
			Pass.BloomCombine
		);
```

<center><img src=".\..\..\Typora-Note\assets\additive-3.png" alt="3 steps" style="zoom:50%;" /> <img src=".\..\..\Typora-Note\assets\additive-5.png" alt="5 steps" style="zoom:50%;" /></center>

<center>Additive upsampling, 3 and 5 steps.</center>

我们终于有了一个看起来像一切都在发光的效果。但是我们的新方法只有在至少有两次迭代的情况下才有效。如果我们最终只进行了一次迭代，那么我们应该跳过整个<font color="red">upsampling 上采样</font>阶段，只需要释放用于第一次水平传递的纹理。

```c#
	if (i > 1) {
			buffer.ReleaseTemporaryRT(fromId - 1);
			toId -= 5;
			for (i -= 1; i > 0; i--) {
				…
			}
		}
		else {
			buffer.ReleaseTemporaryRT(bloomPyramidId);
		}
```

而如果我们最终完全跳过了<font color="red"> bloom</font>，我们就必须中止并进行复制。

```c#
	int width = camera.pixelWidth / 2, height = camera.pixelHeight / 2;
		
		if (
			bloom.maxIterations == 0 ||
			height < bloom.downscaleLimit || width < bloom.downscaleLimit
		) {
			Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
			buffer.EndSample("Bloom");
			return;
		}
```

#### [2.5、Bicubic Upsampling](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.5)

虽然高斯滤波产生了平滑的结果，但我们在上采样时仍然进行了双线性滤波，这可能会使发光体出现块状的外观。这在原始图像收缩率高的地方最为明显，特别是在运动时。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/upsampling-bilinear.png" alt="img" style="zoom:50%;" />

<center>White glow on black background appears blocky.</center>

我们可以通过切换到<font color="red">bicubic filtering</font>来平滑这些<font color="red">artifacts - 伪影</font>。对此没有硬件支持，但我们可以使用<font color="green">Core RP Library</font>的<font color="RoyalBlue">Filtering include</font>文件中定义的<font color="RoyalBlue">SampleTexture2DBic</font>函数。用它来创建我们自己的<font color="red">GetSourceBic</font>函数，通过传递<font color="RoyalBlue">texture and sampler state, UV coordinates, plus the texel size vector with the size pairs swapped - <font color="DarkOrchid ">纹理和采样器状态、UV坐标，加上尺寸对交换的texel尺寸向量</font></font>。

除此之外，它还有一个最大纹理坐标的参数，就是1，后面还有一个不用的参数，可以是0。

```c#
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Filtering.hlsl"

…

float4 GetSourceBicubic (float2 screenUV) {
	return SampleTexture2DBicubic(
		TEXTURE2D_ARGS(_PostFXSource, sampler_linear_clamp), screenUV,
		_PostFXSource_TexelSize.zwxy, 1.0, 0.0
	);
}
```

在<font color="green">bloom-combine</font>通道中使用新的函数，这样我们就可以用<font color="red">bicubic  filtering.</font>进行upample。

```c#
float4 BloomCombinePassFragment (Varyings input) : SV_TARGET {
	float3 lowRes = GetSourceBicubic(input.screenUV).rgb;
	float3 highRes = GetSource2(input.screenUV).rgb;
	return float4(lowRes + highRes, 1.0);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/upsampling-bicubic.png" alt="img" style="zoom:50%;" />

<center>Bicubic upsampling yields smoother glow.</center>

<font color="green">Bicubic sampling</font>会产生更好的效果，但需要四个<font color="red">weighed  texture</font>样本，而不是一个样本。所以让我们通过着色器布尔值使其成为可选项，以防不需要它。这与<font color="RoyalBlue">Universal RP和HDRP</font>的<font color="red">High Quality bloom toggles</font>是对应的。

```c#
bool _BloomBicubicUpsampling;

float4 BloomCombinePassFragment (Varyings input) : SV_TARGET {
	float3 lowRes;
	if (_BloomBicubicUpsampling) {
		lowRes = GetSourceBicubic(input.screenUV).rgb;
	}
	else {
		lowRes = GetSource(input.screenUV).rgb;
	}
	float3 highRes = GetSource2(input.screenUV).rgb;
	return float4(lowRes + highRes, 1.0);
}
```

将它的切换选项添加到 <font color="red">PostFXSettings.BloomSettings</font>。

```
		public bool bicubicUpsampling;
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/bicubic-upsampling-toggle.png)

<center>Bicubic upsampling toggle.</center>

并在我们开始上采样之前，在<font color="green">PostFXStack.DoBloom</font>中把它传递给GPU。

```c#
	int
		bloomBucibicUpsamplingId = Shader.PropertyToID("_BloomBicubicUpsampling"),
		fxSourceId = Shader.PropertyToID("_PostFXSource"),
		fxSource2Id = Shader.PropertyToID("_PostFXSource2");
	
	…
	
	void DoBloom (int sourceId) {
		…
		
		buffer.SetGlobalFloat(
			bloomBucibicUpsamplingId, bloom.bicubicUpsampling ? 1f : 0f
		);
		if (i > 1) { … }
		…
	}
```

#### 2.6、[Half Resolution](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.6)

<font color="red">Bloom</font>可能需要大量的时间去调制所有的纹理采样和绘制。降低成本的一个简单方法是在一半的分辨率下生成它。因为这个效果是柔和的，我们可以不这样做。

这将改变效果的外观，因为我们实际上是跳过了第一次迭代。

首先，在决定跳过<font color="red">bloom</font>时，我们应该先看一步。实际上，在最初的检查中，<font color="RoyalBlue"> downscale</font>的限制是加倍的。

```c#
		if (
			bloom.maxIterations == 0 ||
			height < bloom.downscaleLimit * 2 || width < bloom.downscaleLimit * 2
		) {
			Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
			buffer.EndSample("Bloom");
			return;
		}
```

第二，我们需要为半幅图像申请一个纹理，我们将把它作为新的起点。它不是<font color="RoyalBlue">bloom pyramid</font>的一部分，所以我们要为它申请一个新的<font color="red"> identifier </font>。我们用它来做<font color="DarkOrchid ">pre-filtering </font>步骤，所以给它起个合适的名字。

```c#
int
		bloomBucibicUpsamplingId = Shader.PropertyToID("_BloomBicubicUpsampling"),
		bloomPrefilterId = Shader.PropertyToID("_BloomPrefilter"),
		…
```

回到<font color="red">DoBloom</font>，将源码复制到一个<font color="RoyalBlue">pre-filter texture </font>，并将其用于金字塔的起点，同时将宽度和高度再次减半。在上升到金字塔之后，我们就不需要<font color="RoyalBlue">pre-filter texture </font>了，所以可以在这时释放它。

```c#
RenderTextureFormat format = RenderTextureFormat.Default;
		buffer.GetTemporaryRT(
			bloomPrefilterId, width, height, 0, FilterMode.Bilinear, format
		);
		Draw(sourceId, bloomPrefilterId, Pass.Copy);
		width /= 2;
		height /= 2;

		int fromId = bloomPrefilterId, toId = bloomPyramidId + 1;
		int i;
		for (i = 0; i < bloom.maxIterations; i++) {
			…
		}

		buffer.ReleaseTemporaryRT(bloomPrefilterId);
```

<center><img src=".\..\..\Typora-Note\assets\half-res-2.png" alt="2 steps" style="zoom:50%;" /> <img src=".\..\..\Typora-Note\assets\half-res-4.png" alt="4 steps" style="zoom:50%;" /></center>

<center>Bloom at half resolution, 2 and 4 steps.</center>

#### 2.7、[Threshold](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.7)

绽放通常在艺术上只用于使某些东西发光，但我们的效果目前适用于所有东西，不管它有多亮。虽然这没有物理意义，但我们可以通过引入一个亮度阈值来限制对效果的贡献。

我们不能突然从效果中消除颜色，因为这将引入尖锐的边界，而在这里预期会有一个渐进的过渡。我们将颜色乘以权重

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/bloom/threshold-graph.png" alt="img" style="zoom:50%;" />

其中<font color="RoyalBlue">b是亮度，t是计算的阈值</font>。我们将使用最大的颜色的RGB的通道B，当阈值为0时，结果总是1，这使颜色保持不变。随着阈值的增加，权重曲线将向下弯曲，从而变为零，在<font color="red"> b≤t</font>时候

由于曲线的形状，它被称为膝盖曲线。



<center>Thresholds 0.25, 0.5, 0.75, and 1.</center>

这条曲线在一个角度上达到零，这意味着尽管过渡比钳子更平滑，但仍然有一个突然的截止点。这就是为什么它也被称为硬膝。我们可以通过改变权重来控制膝盖的形状

![CodeCogsEqn](.\..\..\Typora-Note\assets\CodeCogsEqn.gif)

其中

![CodeCogsEqn (1)](.\..\..\Typora-Note\assets\CodeCogsEqn (1)-1678029245461-3.gif)

k是Knee 0 - 1的滑块

<img src=".\..\..\Typora-Note\assets\knee-graph.png" alt="img" style="zoom:50%;" />

<center>Threshold 1 with knee 0, 0.25, 0.5, 0.75, and 1.</center>

让我们把<font color="RoyalBlue">threshold and knee</font>滑块都添加到<font color="DarkOrchid ">PostFXSettings.BloomSettings</font>中。我们将把配置的阈值当作伽玛值，因为这在视觉上更直观，所以我们在把它发送到GPU时必须把它转换成<font color="red"> linear space</font>。

我们把它做成开放式的，即使阈值大于0在这一点上会消除所有的颜色，因为我们只限于LDR。

```c#
	[Min(0f)]
		public float threshold;

		[Range(0f, 1f)]
		public float thresholdKnee;
```

我们将通过一个向量将阈值发送到GPU，我们将其命名为<font color="green">_BloomThreshold</font>。在<font color="red">PostFXStack</font>中为它申请一个<font color="red">identifier</font>。

```c#
		bloomPrefilterId = Shader.PropertyToID("_BloomPrefilter"),
		bloomThresholdId = Shader.PropertyToID("_BloomThreshold"),
```

我们可以计算出权重函数的常数部分，并把它们放在我们的向量的四个组成部分中，以保持<font color="green">Shader</font>的简单

<font color="DarkOrchid ">X是 t 、Y是 -t +tk 、Z是2tk、Z是被除数</font>

我们将在一个新的<font color="red"> pre-filter pass</font>中使用它，取代<font color="red">DoBloom</font>中的初始复制通道，从而将阈值应用于2×2像素的平均值，同时我们将图像大小减半。

```c#
	Vector4 threshold;
		threshold.x = Mathf.GammaToLinearSpace(bloom.threshold);
		threshold.y = threshold.x * bloom.thresholdKnee;
		threshold.z = 2f * threshold.y;
		threshold.w = 0.25f / (threshold.y + 0.00001f);
		threshold.y -= threshold.x;
		buffer.SetGlobalVector(bloomThresholdId, threshold);

		RenderTextureFormat format = RenderTextureFormat.Default;
		buffer.GetTemporaryRT(
			bloomPrefilterId, width, height, 0, FilterMode.Bilinear, format
		);
		Draw(sourceId, bloomPrefilterId, Pass.BloomPrefilter);
```

在PostFXShaderPasses中添加阈值向量和一个将其应用于颜色的函数，然后是使用它的新通行证函数。

```c++
float4 _BloomThreshold;

float3 ApplyBloomThreshold (float3 color) {
	float brightness = Max3(color.r, color.g, color.b);
	float soft = brightness + _BloomThreshold.y;
	soft = clamp(soft, 0.0, _BloomThreshold.z);
	soft = soft * soft * _BloomThreshold.w;
	float contribution = max(soft, brightness - _BloomThreshold.x);
	contribution /= max(brightness, 0.00001);
	return color * contribution;
}

float4 BloomPrefilterPassFragment (Varyings input) : SV_TARGET {
	float3 color = ApplyBloomThreshold(GetSource(input.screenUV).rgb);
	return float4(color, 1.0);
}
```

<img src=".\..\..\Typora-Note\assets\threshold-inspector.png" alt="inspector" style="zoom:50%;" />

<img src=".\..\..\Typora-Note\assets\threshold-scene.png" alt="scene" style="zoom:67%;" />

<center>Four iterations with threshold and knee 0.5.</center>

#### 2.8、[Intensity](https://catlikecoding.com/unity/tutorials/custom-srp/post-processing/#2.8)

我们通过添加一个强度滑块，来控制整体<font color="red">strength of the bloom</font>来结束本教程。我们不会给它一个限制，所以如果需要的话，也可以把整个图像<font color="green">bloom</font>。

```
	[Min(0f)]
		public float intensity;
```

<img src=".\..\..\Typora-Note\assets\intensity.png" alt="img" style="zoom:50%;" />

<center>Open-ended intensity.</center>

如果强度设置为零，我们就可以跳过绽放，所以在<font color="red">DoBloom</font>的开始时要检查这一点。

```c#
		if (
			bloom.maxIterations == 0 || bloom.intensity <= 0f ||
			height < bloom.downscaleLimit * 2 || width < bloom.downscaleLimit * 2
		) {
			Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
			buffer.EndSample("Bloom");
			return;
		}
```

否则将强度传递给GPU，使用一个新的标识符<font color="red">_BloomIntensity</font>。我们将使用它在<font color="green">combine pass</font>中对低分辨率图像进行加权，所以我们不需要创建一个额外的通道。

除了最后绘制到摄像机目标外，所有的绘制都将其设置为1。

```c#
bloomIntensityId = Shader.PropertyToID("_BloomIntensity"),		

...
    
buffer.SetGlobalFloat(bloomIntensityId, 1f);
		if (i > 1) {
			…
		}
		else {
			buffer.ReleaseTemporaryRT(bloomPyramidId);
		}
		buffer.SetGlobalFloat(bloomIntensityId, bloom.intensity);
		buffer.SetGlobalTexture(fxSource2Id, sourceId);
		Draw(fromId, BuiltinRenderTextureType.CameraTarget, Pass.BloomCombine);
```

现在我们只需要将BloomCombinePassFragment中的低分辨率颜色与强度相乘。

```c#
bool _BloomBicubicUpsampling;
float _BloomIntensity;

float4 BloomCombinePassFragment (Varyings input) : SV_TARGET {
	…
	return float4(lowRes * _BloomIntensity + highRes, 1.0);
}
```

<center><img src=".\..\..\Typora-Note\assets\intensity-05.png" alt="0.5" style="zoom:50%;" /> <img src=".\..\..\Typora-Note\assets\intensity-5.png" alt="5" style="zoom:50%;" /></center>

<center>Intensity 0.5 and 5.</center>