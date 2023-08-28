# 12_HDR

渲染到HDR纹理。
		减少萤火虫的绽放。
		增加散射光斑。
		支持多种色调映射模式。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tutorial-image.jpg)

<center>A combination of dark, bright, and very bright areas.</center>

### 1、High Dynamic Range

到目前为止，在渲染摄像机时，我们都是在<font color="red">low dynamic color range低动态色彩范围</font>下进行的，简称为<font color="green">LDR</font>，这是默认的。这意味着每个颜色通道都用一个0-1的值来表示。在这种模式下，（0,0,0）代表黑色，（1,1,1）代表白色。尽管我们的着色器可以产生超出这个范围的结果，但GPU会在存储它们的同时钳制颜色，就像我们在每个片段函数的末尾使用饱和一样。

你可以使用帧调试器来检查每个绘制调用的渲染目标的类型。普通相机的目标被描述为<font color="red">B8G8R8A8_SRGB</font>。这意味着它是一个RGBA缓冲区，每个通道有<font color="green">8 bits</font>，所以每个像素有<font color="green">32 bits</font>。另外，RGB通道是以sRGB色彩空间存储的。由于我们是在线性色彩空间工作，所以当从缓冲区读写时，GPU会自动在两个空间之间进行转换。渲染完成后，缓冲区被发送到显示器，显示器将其解释为sRGB颜色数据。

只要光的强度不超过它，每个颜色通道的最大值为1就可以了。但入射光的强度并没有一个固有的上限。太阳是一个极其明亮的光源的例子，这就是为什么你不应该直接看它。它的强度远远超过我们在眼睛受损之前所能感知的程度。但是许多普通的光源也会产生强度超过观察者极限的光，特别是在近距离观察时。为了正确处理这种强度，我们必须渲染到<font color="red">high-dynamic-range—HDR—buffers</font>，它支持大于1的值。

#### 1.1、[HDR Reflection Probes](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#1.1)

HDR渲染需要HDR渲染目标。这不仅适用于普通相机，也适用于<font color="green">reflection probes</font>。<font color="green">reflection probes</font>是否包含HDR或LDR数据可以通过其HDR切换选项来控制，该选项默认是启用的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/high-dynamic-range/reflection-probe-hdr.png" alt="img" style="zoom:50%;" />

<center>Reflection probe with HDR enabled.</center>

当反射探头使用HDR时，它可以包含高强度的颜色，这主要是它捕获的镜面反射。你可以通过它们在场景中引起的反射间接地观察它们。不完美的反射削弱了探头的颜色，这使得HDR值突出。

```html
<center>![with](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/high-dynamic-range/reflections-with-hdr.png)![without](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/high-dynamic-range/reflections-without-hdr.png)</center>
```

<center>Reflections with and without HDR.</center>

#### 1.2、[HDR Cameras](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#1.2)

相机也有HDR配置选项，但它本身没有任何功能。它可以设置为关闭或使用图形设置。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/high-dynamic-range/camera-hdr.png)

<center>Camera HDR depending on graphics settings.</center>

使用图形设置模式只表明相机允许HDR渲染。这是否发生由RP决定。我们将通过向<font color="red">CustomRenderPipelineAsset</font>添加一个允许HDR的切换来控制，并将其传递给<font color="DarkOrchid ">pipeline constructor.管道构造器</font>。

```c
	[SerializeField]
	bool allowHDR = true;
	
	…

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			allowHDR, useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows, postFXSettings
		);
	}
```

让<font color="red">CustomRenderPipeline</font>来跟踪它，并将它和其他选项一起传递给相机渲染器。

```c#
	bool allowHDR;

	…

	public CustomRenderPipeline (
		bool allowHDR,
		…
	) {
		this.allowHDR = allowHDR;
		…
	}

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera, allowHDR,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings, postFXSettings
			);
		}
	}
```

然后<font color="red">CameraRenderer</font>会跟踪是否应该使用<font color="green">HDR</font>，也就是在<font color="RoyalBlue">camera and the RP</font>都允许的情况下。

```c#
	bool useHDR;

	public void Render (
		ScriptableRenderContext context, Camera camera, bool allowHDR,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings, PostFXSettings postFXSettings
	) {
		…
		if (!Cull(shadowSettings.maxDistance)) {
			return;
		}
		useHDR = allowHDR && camera.allowHDR;

		…
	}
```

<img src="E:\Typora file\SRP\assets\allow-hdr.png" alt="img" style="zoom:50%;" />

<center>HDR allowed.</center>

#### 1.3、[HDR Render Textures](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#1.3)

<font color="red">HDR渲染</font>只有在与<font color="green">post processing</font>相结合时才有意义，因为我们不能改变最终的<font color="DarkOrchid ">frame buffer format</font>。因此，当我们在<font color="red">CameraRenderer.Setup</font>中创建我们自己的中间帧缓冲区时，我们会在适当的时候使用默认的HDR格式，而不是常规的默认的LDR。

```c#
			buffer.GetTemporaryRT(
				frameBufferId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Bilinear, useHDR ?
					RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default
			);
```

帧调试器会显示默认的HDR格式是<font color="red">R16G16B16A16_SFloat</font>，这意味着它是一个RGBA缓冲区，每通道16位，所以每个像素64位，是LDR缓冲区的两倍。在这种情况下，每个值都是线性空间的有符号浮点数，而不是被钳制在0-1。

当你进行绘制调用时，你会注意到场景会比最终的结果显得更暗。发生这种情况是因为这些步骤被存储在HDR纹理中。它看起来很暗，因为线性颜色数据被按原样显示，因此被错误地解释为<font color="red">sRGB</font>。

<center><img src="E:\Typora file\SRP\assets\hdr-before-post.png" alt="hdr" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\ldr-before-post.png" alt="ldr" style="zoom:50%;" /></center>
<center>HDR and LDR, before post processing via frame debugger.</center>

#### 1.4、[HDR Post Processing](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#1.4)

在这一点上，结果看起来和以前没有什么不同，因为我们没有对扩大的范围做任何事情，一旦我们渲染到LDR目标，它就会被夹紧。<font color="RoyalBlue">Bloom</font>可能会显得更亮一些，但不会太多，因为颜色在<font color="red"> pre-filtering pass</font>会被<font color="green">clamped</font>。

我们还必须在HDR中进行后期处理，以充分利用它的优势。因此，让我们在<font color="red">CameraRenderer.Render</font>中调用<font color="red">PostFXStack.Setup</font>时传递是否使用<font color="RoyalBlue">HDR</font>。

```c#
	postFXStack.Setup(context, camera, postFXSettings, useHDR);
```

现在<font color="green">PostFXStack</font>也可以跟踪它是否应该使用<font color="RoyalBlue">HDR</font>。

```c#
	bool useHDR;

	…

	public void Setup (
		ScriptableRenderContext context, Camera camera, PostFXSettings settings,
		bool useHDR
	) {
		this.useHDR = useHDR;
		…
	}
```

而我们可以在<font color="green">DoBloom</font>中使用适当的纹理格式。

```c#
	RenderTextureFormat format = useHDR ?
			RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default;
```

<font color="RoyalBlue">HDR and LDR bloom </font>之间的差异可以是戏剧性的，也可以是难以察觉的，这取决于场景的亮度。通常情况下，<font color="green"> bloom threshold </font>被设置为1，所以只有HDR颜色对其有贡献。这样一来，辉光就表示对显示器来说太亮的颜色。

<img src="E:\Typora file\SRP\assets\hdr-bloom.png" alt="img" style="zoom:50%;" />

<center>HDR bloom; threshold 1 and knee 0.</center>

因为<font color="green">bloom</font>是对颜色的平均化，即使是一个非常明亮的像素，最终也会在视觉上影响一个非常大的区域。你可以通过比较<font color="red"> pre-filter </font>和最终结果来看到这一点。即使是一个像素也会产生一个大的圆形光晕。

<img src="E:\Typora file\SRP\assets\hdr-bloom-prefilter.png" alt="img" style="zoom:50%;" />

<center>HDR bloom pre-filtering step.</center>

例如，当一个2×2块的数值0、0、0、1由于降采样而被平均化时，结果将是0.25。但如果HDR版本的平均值为0、0、0和10，结果将是2.5。与LDR相比，似乎0.25的结果被提升到了1。

#### 1.5、[Fighting Fireflies](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#1.5)

HDR的一个缺点是，它可以产生比周围环境更亮的小图像区域。当这些区域的大小为一个像素或更小时，它们会急剧改变相对大小，并在移动过程中突然出现，从而导致闪烁。这些区域被称为萤火虫。当<font color="green"> bloom </font>被应用于这些区域时，效果会变成<font color="red"> stroboscopic.频闪 </font>。

<center><iframe src="https://gfycat.com/ifr/limpingdesertedgoose?controls=0" style="border: none; overflow: hidden; width: 250px; height: calc(100% + 44px);"></iframe></center>

<center>HDR bloom fireflies.</center>

完全消除这个问题需要无限的分辨率，这是不可能的。我们能做的第二件事是在预过滤时更积极地模糊图像，以淡化萤火虫。让我们在<font color="DarkOrchid ">PostFXSettings.BloomSettings</font>中添加一个切换选项来实现这一点。

```c#
	public bool fadeFireflies;
```

<img src="E:\Typora file\SRP\assets\fade-fireflies.png" alt="img" style="zoom:50%;" />

<center>Fade fireflies enabled.</center>

为此添加一个新的<font color="DarkOrchid ">pre-filter fireflies pass</font>。再一次，我不会展示将<font color="red">Pass</font>添加到<font color="green">PostFxStack</font>着色器和<font color="green">PostFXStack.Pass</font>枚举中。在<font color="red">DoBloom</font>中选择适当的Pass进行<font color="red">pre-filtering</font>。

```c
	Draw(
			sourceId, bloomPrefilterId, bloom.fadeFireflies ?
				Pass.BloomPrefilterFireflies : Pass.BloomPrefilter
		);
```

褪去萤火虫的最直接的方法是将我们的2×2的<font color="green"> downsample filter of the pre-filtering pass</font>发展成一个大的6×6的箱式滤波器。我们可以用9个样本来做，在平均化之前对每个样本单独应用<font color="DarkOrchid ">bloom threshold</font>。在<font color="green">PostFXStackPasses</font>中添加所需的<font color="RoyalBlue">BloomPrefilterFirefliesPassFragment</font>函数。

<img src="C:\Users\admin\Desktop\PPT\6x6-box-filter.png" alt="img" style="zoom:50%;" />

<center>6×6 box filter.</center>

```glsl
float4 BloomPrefilterFirefliesPassFragment (Varyings input) : SV_TARGET {
	float3 color = 0.0;
	float2 offsets[] = {
		float2(0.0, 0.0),
		float2(-1.0, -1.0), float2(-1.0, 1.0), float2(1.0, -1.0), float2(1.0, 1.0),
		float2(-1.0, 0.0), float2(1.0, 0.0), float2(0.0, -1.0), float2(0.0, 1.0)
	};
	for (int i = 0; i < 9; i++) {
		float3 c =
			GetSource(input.screenUV + offsets[i] * GetSourceTexelSize().xy * 2.0).rgb;
		c = ApplyBloomThreshold(c);
		color += c;
	}
	color *= 1.0 / 9.0;
	return float4(color, 1.0);
}
```

但这并不足以解决这个问题，因为非常明亮的像素只是被分散在一个更大的区域。为了淡化萤火虫，我们将根据颜色的亮度，用一个权衡的平均值来代替。<font color="DarkOrchid "> A color's luminance</font>是它的感知亮度。我们将使用<font color="RoyalBlue">Core Library.</font>的颜色HLSL文件中定义的<font color="red">Luminance</font>函数来实现。

```
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Color.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Filtering.hlsl"
```

采样权重是<font color="RoyalBlue">1/(l + 1)</font>，l是 luminance

因此，对于亮度0，权重是1，对于亮度1，权重是1/2，对于3是¼，对于7是⅛，以此类推。

<img src="E:\Typora file\SRP\assets\luminance-weight.png" alt="img" style="zoom:50%;" />

<center>Luminance-based weights.</center>

最后，我们将样本的总和除以这些权重的总和。这有效地将萤火虫的亮度分散到所有其他样本中。如果这些其他样本是黑暗的，那么萤火虫就会褪色。例如，0、0、0和10的加权平均值是0.29。

```
float4 BloomPrefilterFirefliesPassFragment (Varyings input) : SV_TARGET {
	float3 color = 0.0;
	float weightSum = 0.0;
	…
	for (int i = 0; i < 9; i++) {
		…
		float w = 1.0 / (Luminance(c) + 1.0);
		color += c * w;
		weightSum += w;
	}
	color /= weightSum;
	return float4(color, 1.0);
}
```

<img src="E:\Typora file\SRP\assets\luminance-weighed-average.png" alt="img" style="zoom:50%;" />

<center>Luminance-based weighed average.</center>

因为我们在最初的预过滤步骤之后进行了高斯模糊处理，所以我们可以跳过与中心直接相邻的四个样本，将样本量从九个减少到五个。

<img src="C:\Users\admin\Desktop\PPT\x-filter.png" alt="img" style="zoom:50%;" />

<center>6×6 cross filter.</center>

```c++
float2 offsets[] = {
		float2(0.0, 0.0),
		float2(-1.0, -1.0), float2(-1.0, 1.0), float2(1.0, -1.0), float2(1.0, 1.0)//,
		//float2(-1.0, 0.0), float2(1.0, 0.0), float2(0.0, -1.0), float2(0.0, 1.0)
	};
	for (int i = 0; i < 5; i++) { … }
```

这将使单像素萤火虫变成<font color="red">×-shape - ×形图案</font>，并在预过滤步骤中将单像素水平或垂直线分割成两条独立的线，但在第一个模糊步骤之后，这些图案就消失了。

<center><img src="E:\Typora file\SRP\assets\average-5.png" alt="5" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\average-9.png" alt="9" style="zoom:50%;" /></center><center>Pre-filtering step with five and nine samples; half resolution.</center>

这并没有完全消除萤火虫，但却大大降低了它们的强度，使它们不再那么明显，除非绽放强度设置得比1高得多。

<center><iframe src="https://gfycat.com/ifr/cloudypracticalchipmunk?controls=0" style="border: none; overflow: hidden; width: 250px; height:250px;"></iframe></center>

<center>Faded fireflies.</center>

### 2、[Scattering Bloom](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#2)

现在我们有了<font color="green"> HDR bloom</font>，让我们考虑一下它的一个更现实的应用。这个想法是，相机并不完美。他们的镜头并不能正确地聚焦所有的光线。一部分光线会被散射到更大的区域，有点像我们目前的绽放效果。相机越好，散射越少。与我们的加法<font color="green">bloom effect </font>的最大区别是，散射并不增加光线，它只是扩散光线。散射在视觉上可以从轻微的光亮到笼罩整个图像的轻雾。

眼睛也不是完美的，光线会以一种复杂的方式在眼睛里散射。它发生在所有的入射光线上，但只有在明亮的时候才会真正引人注目。例如，在黑暗的背景下看一个小的明亮的光源，如晚上的灯笼，或在明亮的日子里看太阳的反射，它是明显的。

我们将看到的不是均匀的圆形模糊的光芒，而是许多点状的不对称的开始状图案，这些图案也有色调的变化，是我们自己的眼睛所特有的。但我们的<font color="green">bloom effect </font>将代表一个无特征的相机，具有均匀的散射。

<img src="E:\Typora file\SRP\assets\cars-scattering-bloom.jpg" alt="img" style="zoom:50%;" />

<center>Bloom caused by scattering in camera.</center>

#### 2.1、Bloom Mode

我们将支持经典的<font color="DarkOrchid ">additive and energy-conserving scattering bloom</font>。在<font color="green">PostFXSettings.BloomSettings</font>中为这些模式添加一个枚举选项。同时添加一个0-1的滑块来控制光线被散射的程度。

```c#
		public enum Mode { Additive, Scattering }

		public Mode mode;

		[Range(0f, 1f)]
		public float scatter;
```

<img src="E:\Typora file\SRP\assets\scattering-mode.png" alt="img" style="zoom:50%;" />

<center>Scattering mode chosen and set to 0.5.</center>

将现有的<font color="green">BloomCombine</font>通道重命名为<font color="red">BloomAdd</font>，并引入一个新的<font color="DarkOrchid ">BloomScatter Pass</font>。确保枚举和传递的顺序保持字母顺序。然后在组合阶段在<font color="green">DoBloom</font>中使用适当的<font color="RoyalBlue">Pass</font>。在散射的情况下，我们将使用<font color="RoyalBlue">scatter amount</font>来代替1的强度，我们仍然使用配置的强度来进行最终绘制。

```c#
	Pass combinePass;
		if (bloom.mode == PostFXSettings.BloomSettings.Mode.Additive) {
			combinePass = Pass.BloomAdd;
			buffer.SetGlobalFloat(bloomIntensityId, 1f);
		}
		else {
			combinePass = Pass.BloomScatter;
			buffer.SetGlobalFloat(bloomIntensityId, bloom.scatter);
		}
		
		if (i > 1) {
			buffer.ReleaseTemporaryRT(fromId - 1);
			toId -= 5;
			for (i -= 1; i > 0; i--) {
				buffer.SetGlobalTexture(fxSource2Id, toId + 1);
				Draw(fromId, toId, combinePass);
				…
			}
		}
		else {
			buffer.ReleaseTemporaryRT(bloomPyramidId);
		}
		buffer.SetGlobalFloat(bloomIntensityId, bloom.intensity);
		buffer.SetGlobalTexture(fxSource2Id, sourceId);
		Draw(fromId, BuiltinRenderTextureType.CameraTarget, combinePass);
```

<font color="green">BloomScatter Pass</font> 的功能与<font color="RoyalBlue">BloomAdd Pass</font>的功能相同，只是它根据强度在<font color="red">high-resolution and low-resolution sources </font>之间进行插值，而不是添加它们。

因此，散射量为0意味着只使用最低的Bloom金字塔级别，而散射量为1意味着只使用最高的。在0.5的情况下，连续几级的贡献最终为0.5、0.25、0.125、0.125（如果有四个级别）。

```c#
float4 BloomScatterPassFragment (Varyings input) : SV_TARGET {
	float3 lowRes;
	if (_BloomBicubicUpsampling) {
		lowRes = GetSourceBicubic(input.screenUV).rgb;
	}
	else {
		lowRes = GetSource(input.screenUV).rgb;
	}
	float3 highRes = GetSource2(input.screenUV).rgb;
	return float4(lerp(highRes, lowRes, _BloomIntensity), 1.0);
}
```

<center><iframe src="https://gfycat.com/ifr/excellentevilbullmastiff?controls=0" style="border: none; overflow: hidden; width: 250px; height: 250px;"></iframe></center>

<center>Varying bloom scatter; intensity 20 light inside structure; max iterations 16.</center>

<font color="green">Scattering bloom</font>并不会使图像变亮。它可能会使上面的例子看起来变暗，但那是因为它只显示了原始图像的一个裁剪部分。然而，能量保护并不完美，因为高斯滤镜被夹在图像的边缘，这意味着边缘像素的贡献被放大了。我们可以对此进行补偿，但不会这样做，因为这通常并不明显。

#### 2.2、[Scatter Limits](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#2.2)

因为<font color="DarkOrchid ">scatter</font>值为0和1时，除了一个金字塔级别外，其他的都被消除了，所以使用这些值没有意义。所以让我们把散点滑块的范围缩小到0.05-0.95。这使得默认值0无效，所以明确地用一个值来初始化<font color="green">BloomSettings</font>。让我们使用0.07，这与<font color="red">URP和HDRP</font>使用的<font color="green">scatter</font>默认值相同。

```
public struct BloomSettings {

		…

		[Range(0.05f, 0.95f)]
		public float scatter;
	}

	[SerializeField]
	BloomSettings bloom = new BloomSettings {
		scatter = 0.7f
	};
```

另外，强度大于1对于<font color="green">scattering bloom</font>是不合适的，因为那会增加光线。所以我们将在<font color="DarkOrchid ">DoBloom</font>中夹住它，把最大值限制在0.95，这样原始图像就会一直对结果有所贡献。

```c
	float finalIntensity;
		if (bloom.mode == PostFXSettings.BloomSettings.Mode.Additive) {
			combinePass = Pass.BloomAdd;
			buffer.SetGlobalFloat(bloomIntensityId, 1f);
			finalIntensity = bloom.intensity;
		}
		else {
			combinePass = Pass.BloomScatter;
			buffer.SetGlobalFloat(bloomIntensityId, bloom.scatter);
			finalIntensity = Mathf.Min(bloom.intensity, 0.95f);
		}

		if (i > 1) {
			…
		}
		else {
			buffer.ReleaseTemporaryRT(bloomPyramidId);
		}
		buffer.SetGlobalFloat(bloomIntensityId, finalIntensity);
```

<img src="E:\Typora file\SRP\assets\intensity-05-scatter-07.png" alt="img" style="zoom:50%;" />

<center>Intensity 0.5 and scatter 0.7.</center>

#### 2.3、[Threshold](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#2.3)

<font color="green">Scattering bloom </font>比<font color="green"> additive bloom.</font>要微妙得多。它通常也用于低强度的情况。这意味着，就像真正的相机一样，只有在非常明亮的光线下，散射效果才真正明显，尽管所有的光线都被散射了。

虽然这并不现实，但仍有可能应用一个阈值来消除较暗像素的<font color="DarkOrchid ">scattering</font>。这可以在使用较强的<font color="green">bloom</font>时保持图像的清晰度。然而，这消除了光线，从而使图像变暗。

<img src="E:\Typora file\SRP\assets\threshold-too-dark.png" alt="img" style="zoom:50%;" />

<center>Threshold 1, knee 0, and Intensity 1.</center>

我们必须对丢失的散射光进行补偿。我们通过创建一个额外的<font color="red">BloomScatterFinal</font>通道来做到这一点，我们用它来完成<font color="red">scattering bloom</font>的最后绘制。

```c#
	Pass combinePass, finalPass;
		float finalIntensity;
		if (bloom.mode == PostFXSettings.BloomSettings.Mode.Additive) {
			combinePass = finalPass = Pass.BloomAdd;
			buffer.SetGlobalFloat(bloomIntensityId, 1f);
			finalIntensity = bloom.intensity;
		}
		else {
			combinePass = Pass.BloomScatter;
			finalPass = Pass.BloomScatterFinal;
			buffer.SetGlobalFloat(bloomIntensityId, bloom.scatter);
			finalIntensity = Mathf.Min(bloom.intensity, 1f);
		}

		…
		Draw(fromId, BuiltinRenderTextureType.CameraTarget, finalPass);
	}
```

这个通道的函数是其他<font color="green">scatter pass </font>函数的副本，但有一点不同。它将缺失的光线添加到<font color="DarkOrchid ">low-resolution pass</font>中，通过添加高分辨率的光线，然后再减去它，但要对其应用<font color="RoyalBlue"> bloom threshold </font>。

这不是一个完美的重建--它不是一个加权平均数，并且忽略了由于<font color="green">fading fireflies</font>而损失的光线--但已经足够接近了，并且不会在原始图像中增加光线。

```c#
float4 BloomScatterFinalPassFragment (Varyings input) : SV_TARGET {
	float3 lowRes;
	if (_BloomBicubicUpsampling) {
		lowRes = GetSourceBicubic(input.screenUV).rgb;
	}
	else {
		lowRes = GetSource(input.screenUV).rgb;
	}
	float3 highRes = GetSource2(input.screenUV).rgb;
	lowRes += highRes - ApplyBloomThreshold(highRes);
	return float4(lerp(highRes, lowRes, _BloomIntensity), 1.0);
}
```

<img src="E:\Typora file\SRP\assets\scatter-final.png" alt="img" style="zoom:50%;" />

<center>Threshold with scatter final pass.</center>

### 3、[Tone Mapping](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#3)

虽然我们可以用HDR渲染，但对于普通相机来说，最终的帧缓冲区总是LDR。因此，色彩通道在1处被切断。实际上，最终图像的白点在1处。极亮的颜色最终看起来与那些完全饱和的颜色没有什么不同。例如，我做了一个场景，有多个光照度，物体的发射量不同，远远超过1。最强的发射是8，最亮的光的强度是200。

<img src="E:\Typora file\SRP\assets\without-fx.png" alt="img" style="zoom:50%;" />

<center>Scene without post FX; only realtime lighting.</center>

如果不使用任何后期特效，就很难甚至不可能分辨出哪些物体和灯光是非常明亮的。我们可以使用<font color="green">bloom</font>来使其明显。例如，我使用<font color="DarkOrchid "> threshold 1, knee 0.5, intensity 0.2, and scatter 0.7 with max iterations.</font>

<center><img src="E:\Typora file\SRP\assets\bloom-additive-1677838495573-167.png" alt="additive" style="zoom: 50%;" /> <img src="E:\Typora file\SRP\assets\bloom-scattering-1677838495574-169.png" alt="scattering" style="zoom:50%;" /></center>

<center>With bloom, additive and scattering.</center>

发光的物体显然应该是明亮的，但我们仍然没有感觉到它们相对于场景的其他部分有多亮。要做到这一点，我们需要调整图像的亮度--增加其白点--使最亮的颜色不再超过1。我们可以通过均匀地使整个图像变暗来做到这一点，但这将使大部分图像变得非常暗，我们将无法清楚地看到它。理想的情况是，我们对非常明亮的颜色进行大量调整，而对暗色只进行少量调整。因此，我们需要一个非均匀的颜色调整。这种颜色调整并不代表光本身的物理变化，而是它的观察方式。例如，我们的眼睛对深色的色调比浅色的色调更敏感。

从HDR到LDR的转换被称为<font color="green"> tone mapping</font>，它来自于摄影和胶片冲洗。传统的照片和胶片也有一个有限的范围和不均匀的感光性，所以开发了许多技术来进行转换。没有单一的正确方法来执行色调映射。不同的方法可以用来设置最终结果的气氛，就像经典的电影外观。

#### 3.1、[Extra Post FX Step](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#3.1)

我们在绽放后的一个新的<font color="green"> post FX</font>步骤中进行<font color="green">tone mapping</font>。为此，在<font color="green">PostFXStack</font>中添加一个<font color="red">DoToneMapping</font>方法，最初只是将<font color="green">source </font>复制到相机目标上。

```
	void DoToneMapping(int sourceId) {
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
	}
```

我们需要调整<font color="DarkOrchid ">bloom</font>的结果，所以得到一个新的全分辨率临时渲染纹理，并将其作为<font color="green">DoBloom</font>的最终目标。同时让它返回是否绘制了任何东西，而不是在效果被跳过时直接绘制到摄像机目标。

```c#
	int
		bloomBicubicUpsamplingId = Shader.PropertyToID("_BloomBicubicUpsampling"),
		bloomIntensityId = Shader.PropertyToID("_BloomIntensity"),
		bloomPrefilterId = Shader.PropertyToID("_BloomPrefilter"),
		bloomResultId = Shader.PropertyToID("_BloomResult"),
		…;

	…
	
	bool DoBloom (int sourceId) {
		//buffer.BeginSample("Bloom");
		PostFXSettings.BloomSettings bloom = settings.Bloom;
		int width = camera.pixelWidth / 2, height = camera.pixelHeight / 2;
		
		if (
			bloom.maxIterations == 0 || bloom.intensity <= 0f ||
			height < bloom.downscaleLimit * 2 || width < bloom.downscaleLimit * 2
		) {
			//Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
			//buffer.EndSample("Bloom");
			return false;
		}
		
		buffer.BeginSample("Bloom");
		…
		buffer.SetGlobalFloat(bloomIntensityId, finalIntensity);
		buffer.SetGlobalTexture(fxSource2Id, sourceId);
		buffer.GetTemporaryRT(
			bloomResultId, camera.pixelWidth, camera.pixelHeight, 0,
			FilterMode.Bilinear, format
		);
		Draw(fromId, bloomResultId, finalPass);
		buffer.ReleaseTemporaryRT(fromId);
		buffer.EndSample("Bloom");
		return true;
	}
```

调整 "<font color="red">Render</font>"，如果 "<font color="RoyalBlue">Bloom</font>"处于激活状态，它将在 "<font color="RoyalBlue">Bloom</font> "结果上执行<font color="RoyalBlue"> tone mapping</font>，然后释放 "<font color="RoyalBlue">Bloom</font>"结果纹理。否则，让它直接在原始源上应用<font color="RoyalBlue"> tone mapping</font>，完全跳过绽放。

```c#
	public void Render (int sourceId) {
		if (DoBloom(sourceId)) {
			DoToneMapping(bloomResultId);
			buffer.ReleaseTemporaryRT(bloomResultId);
		}
		else {
			DoToneMapping(sourceId);
		}
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
```

#### 3.2、[Tone Mapping Mode](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#3.2)

音调映射有多种方法，我们将支持几种方法，因此在PostFXSettings中添加一个ToneMappingSettings配置结构，其中的Mode枚举选项最初只包含None。

```
	[System.Serializable]
	public struct ToneMappingSettings {

		public enum Mode { None }

		public Mode mode;
	}

	[SerializeField]
	ToneMappingSettings toneMapping = default;

	public ToneMappingSettings ToneMapping => toneMapping;
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/tone-mapping-mode.png)

<center>Tone mapping mode set to none.</center>

#### 3.3、[Reinhard](https://catlikecoding.com/unity/tutorials/custom-srp/hdr/#3.3)

我们的<font color="green"> tone mapping</font>的目的是降低图像的亮度，使原本统一的白色区域显示出各种颜色，显示出原本失去的细节。这就像你的眼睛适应了一个突然变亮的环境，直到你能重新看清楚。但我们不想统一缩小整个图像，因为这将使较深的颜色无法辨别，用过亮换取曝光不足。因此，我们需要一种非线性转换，它不会大量减少暗部数值，但会大量减少高部数值。在极端情况下，零仍然是零，而接近无穷大的数值则减少为1。

一个简单的函数可以完成这个，函数方程是<font color="red">C/(1 +C)</font>，C是颜色通道。

这个函数以最简单的形式被称为莱因哈德色调映射操作，最初由<font color="DarkOrchid ">Mark Reinhard</font>提出，只是他把它应用于亮度，而我们将把它单独应用于每个颜色通道

<img src="E:\Typora file\SRP\assets\reinhard-graph.png" alt="img" style="zoom:50%;" />

<center>Reinhard tone mapping.</center>

在<font color="red">ToneMappingSettings.Mode</font>中添加一个<font color="DarkOrchid ">Reinhard</font>的选项，在<font color="green">Reinhard</font>之后。然后让这个枚举从-1开始，这样Reinhard的值就是0。

```
	public enum Mode { None = -1, Reinhard }
```

接下来，添加一个<font color="green">ToneMappingReinhard Pass</font> ，让<font color="green">PostFXStack.DoTonemapping</font>在适当的时候使用它。具体来说，如果模式是负的，就进行简单的复制，否则就应用<font color="DarkOrchid ">Reinhard</font>色调映射。

```c#
	void DoToneMapping(int sourceId) {
		PostFXSettings.ToneMappingSettings.Mode mode = settings.ToneMapping.mode;
		Pass pass = mode < 0 ? Pass.Copy : Pass.ToneMappingReinhard;
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, pass);
	}
```

<font color="green">ToneMappingReinhardPassFragment</font>着色器函数只是应用了这个函数。

```
float4 ToneMappingReinhardPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb /= color.rgb + 1.0;
	return color;
}
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/bloom-additive.png" alt="additive" style="zoom:50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/bloom-scattering.png" alt="scattering" style="zoom:50%;" /></center>

<center>Top no tone mapping, bottom Reinhard, both with additive and scattering bloom.</center>

<center>上无色调映射，下有莱因哈德，既加色又散射绽放。</center>

这个方法可行，但由于精度的限制，对于非常大的数值可能会出问题。出于同样的原因，非常大的数值比无限大的数值更早地在1处结束。因此，让我们在执行色调映射之前对颜色进行钳制。60的限制避免了我们将支持的所有模式的任何潜在问题。

```glsl
	color.rgb = min(color.rgb, 60.0);
	color.rgb /= color.rgb + 1.0;
```

#### 3.4、Neutral

莱茵哈德色调映射的白点理论上是无限的，但它可以被调整，以便提前达到最大值，这就削弱了调整的效果。

这种替代方法是将公式转换为![CodeCogsEqn](E:\Typora file\SRP\assets\CodeCogsEqn-1678107298936-1.gif)

其中w 是白颜色的点

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/adjusted-reinhard-graph.png" alt="img" style="zoom: 33%;" />

<center>Reinhard with white point at infinity and 4.</center>

我们可以为此增加一个配置选项，但莱因哈德并不是我们唯一可以使用的功能。一个更有趣的，经常被使用的是
![CodeCogsEqn (1)](E:\Typora file\SRP\assets\CodeCogsEqn (1)-1678107466048-3.gif)

在这个情况中X是输入的颜色通道，其他值是配置曲线的常数，最后的颜色是![CodeCogsEqn (2)](E:\Typora file\SRP\assets\CodeCogsEqn (2).gif)

C是输入的颜色通道,<font color="RoyalBlue"> **e** an exposure bias, and **w** the white point</font>它可以产生一个s型曲线，其<font color="green">toe region</font>区域从黑色向上弯曲到中间的线性部分，最后是<font color="DarkOrchid "> shoulder region</font>，在接近白色时变平。

上述函数是由John Hable设计的。它首次被用于《解忧杂货店》（Uncharted 2）（见幻灯片142和143）。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/uncharted2-graph.png" alt="img" style="zoom:33%;" />

<center>Reinhard and Uncharted 2 tone mapping.</center>

<font color="red">URP和HDRP</font>使用这个函数的变体，有自己的配置值，白点为5.3，但它们也使用白度的曝光偏差.![CodeCogsEqn (3)](E:\Typora file\SRP\assets\CodeCogsEqn (3).gif)

这导致有效<font color="DarkOrchid ">white point</font>大约为4.035。它用于<font color="red">neutral tone mapping</font>选项，可以通过<font color="RoyalBlue">*Color* Core Library HLSL file</font>中的<font color="red">NeutralTonemap</font>函数获得。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/hdr/tone-mapping/reinhard-neutral-graph.png" alt="img" style="zoom:33%;" />

<center>Reinhard white point infinite and 4, and neutral tone mapping.</center>

让我们为这种<font color="green">tone mapping</font>模式添加一个选项。把它放在模式枚举中的 "无 "后面和 "莱茵哈德 "前面。

```c#
	public enum Mode { None = -1, Neutral, Reinhard }
```

然后为它创建另一个通道。<font color="green">PostFXStack.DoToneMapping</font>现在可以通过将模式添加到中性选项（如果它不是无的话）来找到正确的通道。

```
Pass pass =
			mode < 0 ? Pass.Copy : Pass.ToneMappingNeutral + (int)mode;
```

然后，<font color="green">ToneMappingNeutralPassFragment</font>函数只需要调用<font color="green">NeutralTonemap</font>函数。

```
float4 ToneMappingNeutralPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = min(color.rgb, 60.0);
	color.rgb = NeutralTonemap(color.rgb);
	return color;
}
```

<center><img src="E:\Typora file\SRP\assets\reinhard-additive-1677838914003-193.png" alt="reinhard additive" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\reinhard-scattering-1677838914003-195.png" alt="reinhard scattering" style="zoom:50%;" /></center>
<center><img src="E:\Typora file\SRP\assets\neutral-additive.png" alt="neutral additive" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\neutral-scattering.png" alt="neutral scattering" style="zoom:50%;" /></center>

<center>Top Reinhard, bottom neutral.</center>

你可以添加配置选项来调整你自己的曲线，但我们将继续我们最后的色调映射模式。

#### 3.5、ACES

在本教程中，我们将支持的最后一种模式是<font color="green">ACES tone mapping</font>，URP和HDRP也使用它。ACES是<font color="red">Academy Color Encoding System</font>（学院色彩编码系统）的简称，它是交换数字图像文件、管理色彩工作流程以及为交付和归档创建母版的全球标准。我们只使用它的色调映射方法，由Unity实施。

首先，把它添加到Mode枚举中，直接放在<font color="red">None</font>之后，以保持其他的字母顺序。

```c#
public enum Mode { None = -1, ACES, Neutral, Reinhard }
```

添加通道并调整 <font color="green">PostFXStack.DoToneMapping</font>，使其以 ACES 开头。

```c#
	Pass pass =
			mode < 0 ? Pass.Copy : Pass.ToneMappingACES + (int)mode;
```

新的<font color="red">ToneMappingACESPassFragment</font>函数可以简单地使用核心库中的<font color="red">AcesTonemap</font>函数。它是通过颜色包含的，但有一个单独的<font color="red">ACES HLSL</font>文件，你可以研究一下。该函数的输入颜色必须是<font color="green">ACES</font>颜色空间，为此我们可以使用<font color="RoyalBlue">unity_to_ACES</font>函数。

```glsl
float4 ToneMappingACESPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = min(color.rgb, 60.0);
	color.rgb = AcesTonemap(unity_to_ACES(color.rgb));
	return color;
}
```

<center><img src="E:\Typora file\SRP\assets\neutral-additive-1677839016987-203.png" alt="neutral additive" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\neutral-scattering-1677839016987-205.png" alt="neutral scattering" style="zoom:50%;" /></center>
<center><img src="E:\Typora file\SRP\assets\aces-additive.png" alt="ACES additive" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\aces-scattering.png" alt="ACES scattering" style="zoom:50%;" /></center>
<center><img src="E:\Typora file\SRP\assets\bloom-additive-1677839016988-209.png" alt="additive" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\bloom-scattering-1677839016988-211.png" alt="scattering" style="zoom:50%;" /></center>

<center>Top neutral, middle ACES, bottom no tone mapping.</center>

ACES与其他模式最明显的区别是，它为非常明亮的颜色增加了一个色相偏移，将它们推向白色。当相机或眼睛被过多的光线淹没时也会发生这种情况。结合绽放，现在可以清楚地看到哪些表面是最亮的。另外，ACES的色调映射将暗色降低了一些，这增强了对比度。其结果是一种电影般的外观。