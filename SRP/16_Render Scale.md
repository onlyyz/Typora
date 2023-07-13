# 16_Render Scale

通过一个滑块来调整渲染比例。
		支持每个摄像机的不同渲染比例。
		在所有的后期特效之后，重新调整比例到最终目标。

![img](E:/Typora%20file/SRP/assets/tutorial-image-1678526920604-1.jpg)Comparing different render scales, zoomed in.

### 1、Variable Resolution

一个应用程序在一个固定的分辨率下运行。有些应用程序允许通过设置菜单改变分辨率，但这需要完全重新初始化图形。一个更灵活的方法是保持应用程序的分辨率固定，但改变相机用于渲染的缓冲区的大小。这将影响整个渲染过程，除了最后绘制到帧缓冲区的时候，这时结果会被重新缩放以匹配应用程序的分辨率。

<font color="RoyalBlue">Scaling the buffer </font>的大小可以用来提高性能，通过减少必须处理的片段数量。例如，这可以用于所有的3D渲染，同时保持用户界面在全分辨率下的清晰。比例也可以动态调整，以保持可接受的帧率。

最后，我们还可以增加缓冲区的大小来进行<font color="green">supersample</font>，这样可以减少由有限的分辨率引起的<font color="RoyalBlue">aliasing artifacts</font>。最后一种方法也被称为<font color="red">SSAA</font>，它代表<font color="red">supersampling antialiasing. - 超采样抗锯齿</font>。

#### [1.1、Buffer Settings](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.1)

调整我们渲染的比例会影响到缓冲区的大小，所以我们将为<font color="red">CameraBufferSettings</font>添加一个可配置的渲染比例滑块。应该有一个最小的比例，我们将使用0.1。我们也使用2作为最大值，因为如果我们用单一的双线性插值步骤来重新调整比例的话，超过这个值并不会改善图像质量。超过2会使质量下降，因为我们在<font color="RoyalBlue">downsampling </font>到最终的目标分辨率时，最终会完全跳过许多像素。

```c#
using UnityEngine;

[System.Serializable]
public struct CameraBufferSettings {

	…

	[Range(0.1f, 2f)]
	public float renderScale;
}
```

在<font color="green">CustomRenderPipelineAsset</font>中，默认的渲染比例应该被设置为1。

```c#
	CameraBufferSettings cameraBuffer = new CameraBufferSettings {
		allowHDR = true,
		renderScale = 1f
	};
```

<img src="E:/Typora%20file/SRP/assets/render-scale-slider.png" alt="img" style="zoom:50%;" />

<center>Render scale slider.</center>

#### 1.2、[Scaled Rendering](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.2)

从现在开始，我们还将跟踪我们是否在<font color="green">CameraRenderer</font>中使用缩放渲染。

```c#
	bool useHDR, useScaledRendering;
```

我们不希望配置的<font color="RoyalBlue">render scale</font>影响场景窗口，因为它们是用来编辑的。在适当的时候，通过在<font color="green">PrepareForSceneWindow</font>中关闭<font color="green"> scaled rendering </font>来强制执行。

[ScriptableRenderContext](https://docs.unity3d.com/ScriptReference/Rendering.ScriptableRenderContext.html).EmitWorldGeometryForSceneView 将UI几何图形发射到Scene视图中进行渲染。

```c#
	partial void PrepareForSceneWindow () {
		if (camera.cameraType == CameraType.SceneView) {
			ScriptableRenderContext.EmitWorldGeometryForSceneView(camera);
			useScaledRendering = false;
		}
	}
```

我们在调用<font color="RoyalBlue">Render</font>中的<font color="green">PrepareForSceneWindow</font>之前确定是否应该使用<font color="green"> render scale</font>。在一个变量中跟踪当前的渲染比例，并检查它是否不是1。

```c#
	float renderScale = bufferSettings.renderScale;
		useScaledRendering = renderScale != 1f;
		PrepareBuffer();
		PrepareForSceneWindow();
```

不过我们应该比这更模糊一些，因为与1的非常微小的偏差既不会有视觉上的差异，也不会有性能上的差异。所以我们只在至少有1%的差异时才使用<font color="RoyalBlue"> render scale</font>。

```c#
		useScaledRendering = renderScale < 0.99f || renderScale > 1.01f;
```

从现在开始，当使用<font color="RoyalBlue"> render scale</font>时，我们还必须使用一个<font color="red"> intermediate buffer 中间缓冲区</font>。所以请在设置中检查它。

```c#
	useIntermediateBuffer = useScaledRendering ||
			useColorTexture || useDepthTexture || postFXStack.IsActive;
```

#### 1.3、[Buffer Size](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.3)

因为我们的<font color="red"> camera's buffer </font>大小现在可能与<font color="RoyalBlue">相机</font>组件所显示的不同，我们必须跟踪我们最终使用的缓冲区大小。我们可以使用一个单一的<font color="red">Vector2Int</font>字段来实现。

[Vector2Int](https://docs.unity3d.com/ScriptReference/Vector2Int.html)

```c#
	Vector2Int bufferSize;
```

剔除成功后，在<font color="RoyalBlue">Render</font>中设置适当的<font color="RoyalBlue">Buffer</font>大小。如果按比例渲染，则<font color="red">scale the camera's pixel </font>宽度和高度，并将结果铸成整数，向下取整。

```c#
if (!Cull(shadowSettings.maxDistance)) {
			return;
		}
		
		useHDR = bufferSettings.allowHDR && camera.allowHDR;
		if (useScaledRendering) {
			bufferSize.x = (int)(camera.pixelWidth * renderScale);
			bufferSize.y = (int)(camera.pixelHeight * renderScale);
		}
		else {
			bufferSize.x = camera.pixelWidth;
			bufferSize.y = camera.pixelHeight;
		}
```

在<font color="green"> `Setup`.</font>中为摄像机附件获取<font color="RoyalBlue">render textures</font>时，使用这个<font color="RoyalBlue">buffer </font>大小。

```c#
		buffer.GetTemporaryRT(
				colorAttachmentId, bufferSize.x, bufferSize.y,
				0, FilterMode.Bilinear, useHDR ?
					RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default
			);
			buffer.GetTemporaryRT(
				depthAttachmentId, bufferSize.x, bufferSize.y,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
```

如果需要的话，还可以用于<font color="RoyalBlue">color and depth textures</font>。

```c#
	void CopyAttachments () {
		if (useColorTexture) {
			buffer.GetTemporaryRT(
				colorTextureId, bufferSize.x, bufferSize.y,
				0, FilterMode.Bilinear, useHDR ?
					RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default
			);
			…
		}
		if (useDepthTexture) {
			buffer.GetTemporaryRT(
				depthTextureId, bufferSize.x, bufferSize.y,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
			…
		}
		…
	}
```

最初在没有任何后期特效的情况下进行尝试。你可以放大游戏窗口，这样你可以更好地看到单个像素，这使得调整后的渲染比例更加明显。

<img src="E:/Typora%20file/SRP/assets/scale-100.png" alt="img" style="zoom:50%;" />

<center>No post FX; render scale 1; game window zoomed in.</center>

降低<font color="red"> render scale </font>会加快渲染速度，同时降低图像质量。增加渲染比例则相反。请记住，在不使用后期特效的情况下，调整渲染比例需要一个中间缓冲区和额外的绘制，所以这会增加一些额外的工作。

<center><img src="E:/Typora%20file/SRP/assets/scale-025.png" alt="0.25" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/scale-050.png" alt="0.50" style="zoom:50%;" /> </center>
<center><img src="E:/Typora%20file/SRP/assets/scale-150.png" alt="1.5" style="zoom:50%;" /> <img src="E:/Typora%20file/SRP/assets/scale-200.png" alt="2" style="zoom:50%;" /></center>

<center>Render scale 0.25, 0.5, 1.5, and 2.</center>

重新缩放到<font color="green"> target buffer</font>的大小是由<font color="red">final draw</font>自动完成的。我们最终得到的是一个简单的双线性上缩或下缩操作。唯一奇怪的结果涉及到HDR值，它似乎打破了插值。你可以看到这种情况发生在上述截图中心的黄色球体上的高光。我们将在后面处理这个问题。

#### 1.4、[Fragment Screen UV](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.4)

调整渲染比例会引入一个错误：对<font color="red"> color and depth textures </font>会出错。你可以从粒子变形中看到这一点，它显然最终使用了不正确的屏幕空间UV坐标。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/distortion-incorrect.png" alt="img" style="zoom:50%;" />

<center>Incorrect distortion, render scale 1.5.</center>

这是因为<font color="RoyalBlue">Unity</font>放在<font color="red">**_ScreenParams**</font>中的值与相机的像素尺寸相匹配，而不是我们所针对的缓冲区的尺寸。我们通过引入一个替代的<font color="RoyalBlue">_CameraBufferSize</font>向量来解决这个问题，这个向量包含了相机调整后的尺寸数据。

```c#
	static int
		bufferSizeId = Shader.PropertyToID("_CameraBufferSize"),
		colorAttachmentId = Shader.PropertyToID("_CameraColorAttachment"),
```

在确定了缓冲区的大小之后，我们在<font color="green">Render</font>中把这些值发送给GPU。我们将使用与<font color="green">Unity</font>用于<font color="red">_TexelSize</font>向量的相同格式，所以宽度和高度的倒数，然后是宽度和高度。

```c#
	if (useScaledRendering) {
			bufferSize.x = (int)(camera.pixelWidth * renderScale);
			bufferSize.y = (int)(camera.pixelHeight * renderScale);
		}
		else {
			bufferSize.x = camera.pixelWidth;
			bufferSize.y = camera.pixelHeight;
		}

		buffer.BeginSample(SampleName);
		buffer.SetGlobalVector(bufferSizeId, new Vector4(
			1f / bufferSize.x, 1f / bufferSize.y,
			bufferSize.x, bufferSize.y,
		));
		ExecuteBuffer();
```

Add the vector to *<font color="green">Fragment</font>*.

```c#
TEXTURE2D(_CameraColorTexture);
TEXTURE2D(_CameraDepthTexture);

float4 _CameraBufferSize;
```

然后用它代替<font color="RoyalBlue">GetFragment</font>中的<font color="green">_ScreenParams</font>。现在我们也可以用乘法来代替除法了。

```c#
	f.screenUV = f.positionSS * _CameraBufferSize.xy;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/distortion-correct.png" alt="img" style="zoom:50%;" />

<center>Correct distortion, render scale 1.5.</center>

#### 1.5、[Scaled Post FX](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.5)

调整渲染比例也应该影响到后期特效，否则我们就会出现非预期的缩放。最稳健的方法是始终使用相同的缓冲区大小，所以我们将它作为一个新的第三个参数传递给<font color="green">CameraRenderer</font>.<font color="green">Render</font>中的<font color="red">PostFXStack.Setup</font>。

```c#
	postFXStack.Setup(
			context, camera, bufferSize, postFXSettings, useHDR, colorLUTResolution,
			cameraSettings.finalBlendMode
		);
```

<font color="green">PostFXStack</font>现在必须跟踪缓冲区的大小。

```c#
	Vector2Int bufferSize;

	…
	
	public void Setup (
		ScriptableRenderContext context, Camera camera, Vector2Int bufferSize,
		PostFXSettings settings, bool useHDR, int colorLUTResolution,
		CameraSettings.FinalBlendMode finalBlendMode
	) {
		this.bufferSize = bufferSize;
		…
	}
```

这必须在<font color="green">DoBloom</font>中使用，而不是直接使用相机的像素大小。

```c#
bool DoBloom (int sourceId) {
		BloomSettings bloom = settings.Bloom;
		int width = bufferSize.x / 2, height = bufferSize.y / 2;
		
		…
		buffer.GetTemporaryRT(
			bloomResultId, bufferSize.x, bufferSize.y, 0,
			FilterMode.Bilinear, format
		);
		…
	}
```

因为<font color="green">bloom</font>是一种依赖于分辨率的效果，调整渲染比例会改变它的外观。这一点最容易通过几次反复的<font color="RoyalBlue">Bloom</font>来体现。减少<font color="RoyalBlue">render scale</font>会使效果变大，而增加<font color="RoyalBlue">render scale</font>会使效果变小。最大迭代的<font color="RoyalBlue">Bloom</font>似乎没有什么变化，但是由于分辨率的变化，在调整<font color="RoyalBlue">render scale</font>的时候会出现<font color="DarkOrchid "> - pulse 脉冲</font>。

<center><img src="assets/bloom-050.png" alt="0.5" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/bloom-100.png" alt="1" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/bloom-200.png" alt="2" style="zoom: 50%;" /></center>

<center>Two iterations of additive bloom; render scale 0.5, 1, and 2.</center>

特别是当<font color="RoyalBlue">render scale</font>被逐渐调整时，可能需要尽可能地保持<font color="RoyalBlue">bloom </font>。这可以通过根据相机而不是缓冲区的大小来确定开花金字塔的起始大小来实现。让我们通过在<font color="RoyalBlue">BloomSettings</font>中添加一个忽略<font color="RoyalBlue">render scale</font>的开关来实现这个配置。

```c#
public struct BloomSettings {

		public bool ignoreRenderScale;
		
		…
	}
```

如果渲染比例应该被忽略，<font color="RoyalBlue">PostFXStack.DoBloom</font>将像以前一样，从摄像机像素的一半开始。这意味着它不再执行默认的降采样到一半的分辨率，而是取决于渲染比例。最终的绽放结果仍应与缩放后的缓冲区大小一致，所以这将在最后引入另一个自动<font color="red">downsample or upsample</font>步骤。

```c#
	bool DoBloom (int sourceId) {
		BloomSettings bloom = settings.Bloom;
		int width, height;
		if (bloom.ignoreRenderScale) {
			width = camera.pixelWidth / 2;
			height = camera.pixelHeight / 2;
		}
		else {
			width = bufferSize.x / 2;
			height = bufferSize.y / 2;
		}
		
		…
	}
```

当忽略渲染比例的时候，现在绽放的效果更加一致，尽管在非常低的比例下，它看起来仍然是不同的，只是因为有这么少的数据可以使用。

<img src="assets/bloom-ignore-render-scale.png" alt="inspector" style="zoom:50%;" />

<center><img src="assets/bloom-ignore-050.png" alt="game 0.5" style="zoom:50%;" /> <img src="assets/bloom-100-1678656581840-3.png" alt="game 1" style="zoom:50%;" /> <img src="assets/bloom-ignore-200.png" alt="game 2" style="zoom:50%;" /></center>

<center>Bloom ignoring render scale; render scale 0.5, 1, and 2.</center>

#### 1.6、[Render Scale per Camera](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#1.6)

我们还可以让每个摄像机使用不同的渲染比例。例如，单个摄像机可以始终以一半或两倍的分辨率进行渲染。这可以是固定的--覆盖RP的全局渲染比例，也可以是应用在上面的，所以它是相对于全局渲染比例的。

在<font color="green">CameraSettings</font>中添加一个渲染比例滑块，其范围与RP资产相同。同时，通过一个新的内部<font color="red">RenderScaleMode</font>枚举类型，添加一个可以设置为继承、倍增或覆盖的渲染比例模式。

```c#
	public enum RenderScaleMode { Inherit, Multiply, Override }

	public RenderScaleMode renderScaleMode = RenderScaleMode.Inherit;

	[Range(0.1f, 2f)]
	public float renderScale = 1f;
```

<img src="assets/render-scale-mode.png" alt="img" style="zoom:50%;" />

<center>Render scale mode.</center>

为了应用每个摄像机的渲染比例，还需要给<font color="green">CameraSettings</font>一个公共的<font color="red">GetRenderScale</font>方法，该方法有一个渲染比例参数，并返回最终比例。因此，它要么返回相同的比例，要么返回相机的比例，要么两者相乘，取决于模式。

```c#
	public float GetRenderScale (float scale) {
		return
			renderScaleMode == RenderScaleMode.Inherit ? scale :
			renderScaleMode == RenderScaleMode.Override ? renderScale :
			scale * renderScale;
	}
```

在<font color="RoyalBlue">CameraRenderer.Render</font>中调用该方法以获得最终的渲染比例，将缓冲区设置中的比例传递给它。

```c#
	float renderScale = cameraSettings.GetRenderScale(bufferSettings.renderScale);
```

如果需要的话，我们也来钳制最终的渲染比例，使其保持在0.1-2的范围内。这样我们就可以防止比例过小或过大，以防它被乘以。

```c#
if (useScaledRendering) {
			renderScale = Mathf.Clamp(renderScale, 0.1f, 2f);
			bufferSize.x = (int)(camera.pixelWidth * renderScale);
			bufferSize.y = (int)(camera.pixelHeight * renderScale);
		}
```

由于我们对所有的渲染比例都使用相同的最小和最大，让我们把它们定义为<font color="green">CameraRenderer</font>的公共常量。我只展示了常量的定义，而不是替换<font color="DarkOrchid ">CameraRenderer、CameraBufferSettings和CameraSettings</font>中的0.1f和2f值。

```c#
	public const float renderScaleMin = 0.1f, renderScaleMax = 2f;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/variable-resolution/different-render-scale-per-camera.png" style="zoom:50%;" />

<center>Different render scale per camera.</center>

### 2、Rescaling

当使用1以外的渲染比例时，除了最后绘制到摄像机的目标缓冲区外，所有的事情都在这个比例上发生。如果没有使用后期特效，这只是一个简单的复制，重新缩放到最终的尺寸。当后期特效被激活时，最后的绘制也隐含地执行了重新缩放。然而，在最终绘制过程中重新缩放有一些缺点。

#### 2.1、[Current Approach](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#2.1)

我们目前的重新缩放方法产生了不希望看到的副作用。首先，正如我们之前已经注意到的，无论是上调还是下调HDR的颜色，亮度大于1的颜色总是有异响。插值只有在<font color="RoyalBlue">LDR</font>中进行时才能产生平滑的结果。<font color="RoyalBlue">HDR</font>插值可以产生仍然大于1的结果，这根本不会出现混合。例如，0和10的平均值是5。在LDR中，会出现0和1的平均数是1，而我们期望它是0.5。

<center><img src="assets/hdr-050.png" alt="hdr 0.5" style="zoom:50%;" /> <img src="assets/hdr-200.png" alt="hdr 2" style="zoom:50%;" /></center> 

<center><img src="assets/ldr-050.png" alt="ldr 0.5" style="zoom:50%;" /> <img src="assets/ldr-200.png" alt="without" style="zoom:50%;" /></center>

<center>Color interpolation with and without HDR; render scale 0.5 and 2.</center>

在最后一关中重新调整比例的第二个问题是，<font color="red">color correction颜色校正</font>被应用于内插的颜色而不是原始颜色。这可能会引入不想要的色带。最明显的是在阴影和高光之间插值时出现的中间色调。这可以通过对中间调子应用非常强的颜色调整而变得非常明显，例如使它们变成红色。

<center><img src="assets/midtones-render-scale-050.png" alt="0.5" style="zoom:50%;" /> <img src="assets/midtones-render-scale-100.png" alt="1" style="zoom:50%;" /> <img src="assets/midtones-render-scale-200.png" alt="2" style="zoom:50%;" /></center>

<center>Strong red midtones; render scale 0.5, 1, and 2.</center>

#### 2.2、[Rescaling in LDR](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#2.2)

<font color="DarkOrchid "> sharp HDR edges and color correction artifacts - 尖锐的HDR边缘和色彩校正伪影</font>都是在<font color="RoyalBlue">color correction and tone mapping - 色彩校正和色调映射</font>之前插值HDR颜色造成的。

因此，解决方案是在调整后的渲染比例下进行这两项工作，然后再进行另一个复制传递，重新调整<font color="green">LDR颜色</font>的比例。在<font color="green">PostFXStack</font>着色器中添加一个新的<font color="red"> final rescale pas</font>来处理这最后一步。它是一个简单的复制通道，也有可配置的混合模式。像往常一样，在<font color="green">PostFXStack.Pass</font>枚举中添加一个条目。

```glsl
	Pass {
			Name "Final Rescale"

			Blend [_FinalSrcBlend] [_FinalDstBlend]
			
			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment CopyPassFragment
			ENDHLSL
		}
```

现在我们有两个<font color="green"> final passes</font>，这需要我们为<font color="green">DrawFinal</font>添加一个通道参数。

```c#
	void DrawFinal (RenderTargetIdentifier from, Pass pass) {
		…
		buffer.DrawProcedural(
			Matrix4x4.identity, settings.Material,
			(int)pass, MeshTopology.Triangles, 3
		);
	}
```

现在我们要在<font color="green">DoColorGradingAndToneMapping</font>中使用哪种方法，取决于我们是否在调整渲染比例。我们可以通过比较缓冲区的大小和摄像机的像素大小来检查这一点。检查一下宽度就够了。如果它们相等，我们就像以前一样绘制最终通道，现在明确地将<font color="red">Pass.Final</font>作为一个参数。

```c
void DoColorGradingAndToneMapping (int sourceId) {
		…

		if (bufferSize.x == camera.pixelWidth) {
			DrawFinal(sourceId, Pass.Final);
		}
		else {}
		
		buffer.ReleaseTemporaryRT(colorGradingLUTId);
	}
```

但是如果我们需要<font color="RoyalBlue">rescale </font>，那么我们就必须绘制两次。首先获得一个与当前缓冲区大小相匹配的新的临时<font color="green">render texture</font>。

由于我们在其中存储了<font color="RoyalBlue">LDR颜色</font>，所以我们可以用默认的<font color="green">render texture</font>格式来满足需要。

然后在最后一次绘制时执行常规绘制，最后的混合模式设置为 "<font color="red">One Zero</font>"。之后，用最后的<font color="DarkOrchid ">final rescale pass</font>进行最后的绘制，接着释放<font color="green">intermediate buffer.</font>。

```c#
if (bufferSize.x == camera.pixelWidth) {
			DrawFinal(sourceId, Pass.Final);
		}
		else {
			buffer.SetGlobalFloat(finalSrcBlendId, 1f);
			buffer.SetGlobalFloat(finalDstBlendId, 0f);
			buffer.GetTemporaryRT(
				finalResultId, bufferSize.x, bufferSize.y, 0,
				FilterMode.Bilinear, RenderTextureFormat.Default
			);
			Draw(sourceId, finalResultId, Pass.Final);
			DrawFinal(finalResultId, Pass.FinalRescale);
			buffer.ReleaseTemporaryRT(finalResultId);
		}
```

有了这些变化，HDR颜色也出现了正确的插值。

<center><img src="assets/rescaling-ldr-050.png" alt="0.5" style="zoom:50%;" /> <img src="assets/rescaling-ldr-200.png" alt="2" style="zoom:50%;" /></center>

<center>Rescaling in LDR; render scale 0.5 and 2.</center> 

而且调色不再引入渲染比例1时不存在的色带。

<center><img src="assets/rescaling-after-cc-midtones-050.png" alt="0.5" style="zoom:50%;" /> <img src="assets/rescaling-after-cc-midtones-200.png" alt="2" style="zoom:50%;" /></center>

<center>Rescaling after color correction; strong red midtones; render scale 0.5 and 2.</center>

请注意，这只是在使用后期特效时解决了问题。否则没有调色，我们假设也没有HDR。

#### 2.3、[Bicubic Sampling](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#2.3)

当渲染比例降低的时候，图像就会变得很块。我们添加了一个选项，可以使用<font color="red"> bicubic upsampling </font>来改善<font color="RoyalBlue">Bloom</font>的质量，当重新缩放到最终渲染目标时，我们也可以这样做。在<font color="green">CameraBufferSettings</font>中增加一个切换选项。

```c#
	public bool bicubicRescaling;
```

<img src="assets/bicubic-rescaling-toggle.png" alt="img" style="zoom:50%;" />

<center>Bicubic rescaling toggle.</center>

在<font color="green">PostFXStackPasses</font>中增加了一个新的<font color="green">FinalPassFragmentRescale</font>函数，以及一个<font color="red">_CopyBicubic</font>属性，以控制它是使用<font color="red"> bicubic or regular sampling.</font>。

```glsl
bool _CopyBicubic;

float4 FinalPassFragmentRescale (Varyings input) : SV_TARGET {
	if (_CopyBicubic) {
		return GetSourceBicubic(input.screenUV);
	}
	else {
		return GetSource(input.screenUV);
	}
}
```

改变最后的<font color="RoyalBlue">buffer settings Pass</font>，使用这个函数而不是复制函数。

```glsl
	#pragma fragment FinalPassFragmentRescale
```

将属性标识符添加到<font color="green">PostFXStack</font>中，并使其跟踪是否启用了<font color="DarkOrchid "> bicubic rescaling </font>，这是由<font color="green">Setup</font>的一个新参数配置的。

```c#
	int
		copyBicubicId = Shader.PropertyToID("_CopyBicubic"),
		finalResultId = Shader.PropertyToID("_FinalResult"),
		…
	
	bool bicubicRescaling;

	…

	public void Setup (
		ScriptableRenderContext context, Camera camera, Vector2Int bufferSize,
		PostFXSettings settings, bool useHDR, int colorLUTResolution,
		CameraSettings.FinalBlendMode finalBlendMode, bool bicubicRescaling
	) {
		this.bicubicRescaling = bicubicRescaling;
		…
	}
```

在<font color="RoyalBlue">CameraRenderer.Render</font>中传递缓冲区设置。

```c#
	postFXStack.Setup(
			context, camera, bufferSize, postFXSettings, useHDR, colorLUTResolution,
			cameraSettings.finalBlendMode, bufferSettings.bicubicRescaling
		);
```

并在执行最后的重新缩放之前，在<font color="green">PostFXStack.DoColorGradingAndToneMapping</font>中适当地设置着色器属性。

```c#
	buffer.SetGlobalFloat(copyBicubicId, bicubicRescaling ? 1f : 0f);
			DrawFinal(finalResultId, Pass.FinalRescale);
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/rescaling/bilinear-rescaling-0.25.png" alt="bilinear" style="zoom:50%;" /> <img src="assets/bicubic-rescaling-025.png" alt="bicubic" style="zoom:50%;" /></center>

<center>Bilinear and bicubic rescaling; render scale 0.25.</center>

#### 2.4、[Only Bicubic Upscaling](https://catlikecoding.com/unity/tutorials/custom-srp/render-scale/#2.4)

<font color="DarkOrchid ">Bicubic rescaling </font>在<font color="green">upscaling</font>时总是能提高质量，但在<font color="green">downscaling </font>时，差别一定不明显。对于渲染比例2来说，它总是无用的，<font color="RoyalBlue">因为每个最终的像素是四个像素的平均值，与双线性插值完全相同</font>。因此，让我们把<font color="green">BufferSettings</font>中的切换键换成三种模式：<font color="red">: off, up only, and both up and down.</font>

```c#
	public enum BicubicRescalingMode { Off, UpOnly, UpAndDown }

	public BicubicRescalingMode bicubicRescaling;
```

改变<font color="green">PostFXStack</font>中的类型以匹配。

```c#
	CameraBufferSettings.BicubicRescalingMode bicubicRescaling;

	…

	public void Setup (
		…
		CameraBufferSettings.BicubicRescalingMode bicubicRescaling
	) { … }
```

最后，改变<font color="green">DoColorGradingAndToneMapping</font>，使二次方取样只用于<font color="green"> up-and-down</font>模式，或者如果我们正在使用缩小的渲染比例，只用于<font color="red"> up-only</font>模式。

```c#
	bool bicubicSampling =
				bicubicRescaling == CameraBufferSettings.BicubicRescalingMode.UpAndDown ||
				bicubicRescaling == CameraBufferSettings.BicubicRescalingMode.UpOnly &&
				bufferSize.x < camera.pixelWidth;
			buffer.SetGlobalFloat(copyBicubicId, bicubicSampling ? 1f : 0f);
```

<img src="assets/bicubic-rescaling-up-only.png" alt="img" style="zoom:50%;" />

<center>Only bicubic rescaling for upsampling.</center>