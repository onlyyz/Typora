# 15_Particles

支持翻书、近似褪色、柔和和变形粒子。
		确定片段深度，用于正视和透视投影。
		复制和采样颜色和深度缓冲区。

![img](E:/Typora%20file/SRP/assets/tutorial-image-1678468275180-74.jpg)

<center>Using particles to create a messy atmosphere.</center>

### 1、Unlit Particles

粒子系统可以使用任何材料，所以我们的RP已经可以渲染它们，但有限制。在本教程中，我们将只考虑非光照粒子。光照粒子的工作方式是一样的，只是有更多的着色器属性和照明计算。

我为粒子设置了一个新的场景，它是已经存在的测试场景的一个变体。它有几个长的垂直立方体和一个明亮的黄色灯泡，作为粒子系统的背景。

<img src="E:/Typora%20file/SRP/assets/without-particles.png" alt="img" style="zoom:50%;" />

<center>Scene without particles and without post FX.</center>

#### 1.1、Particle System

通过GameObject / Effects / Particle System创建一个粒子系统，并将其定位在地平面以下一点。我假设你已经知道如何配置粒子系统了，所以就不做详细介绍了。如果没有，请查看Unity的文档，了解具体的模块和它们的设置。

默认的系统使粒子向上移动，并填充一个圆锥状的区域。如果我们把我们的非光照材料分配给它，粒子将显示为与摄像机平面对齐的白色实心方块。它们突然出现，又突然消失，但由于它们开始时低于平面，所以看起来像是从地面上升起。

<img src="E:/Typora%20file/SRP/assets/default-particle-system.png" alt="img" style="zoom:50%;" />

<center>Default particle system with unlit material, positioned below ground.</center>

#### 1.2、Unlit Particles Shader

我们可以将我们的非光照度用于粒子，但让我们为它们制作一个专用的。它从非光照度的副本开始，其菜单项改为<font color="green">*Custom RP/Particles/Unlit*</font>度。另外，由于粒子总是动态的，所以它不需要一个元通道。

```glsl
Shader "Custom RP/Particles/Unlit" {
	
	…
	
	SubShader {
		…

		//Pass {
			//Tags {
				//"LightMode" = "Meta"
			//}

			//…
		//}
	}

	CustomEditor "CustomShaderGUI"
}
```

用这个着色器为非光照粒子创建一个专用材质，然后让粒子系统使用它。目前，它等同于早期的非光照材质。也可以将粒子系统设置为渲染网格，即使有阴影，如果<font color="green">material and the particle </font>系统都启用了阴影的话。然而，GPU实例化不起作用，因为粒子系统使用程序化绘制，我们不会在本教程中介绍。相反，所有的粒子网格被合并成一个单一的网格，就像广告牌粒子一样。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/particles/unlit-particles/particles-sphere.png" alt="img" style="zoom:50%;" />

<center>Sphere mesh particles, with shadows.</center>

从现在开始，我们将只关注没有阴影的广告牌粒子。这里是一个单一粒子的基础图，包含一个简单的平滑渐变的白盘

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/particles/unlit-particles/particles-single.png" alt="img" style="zoom:50%;" />

<center>Base map for single particle, on black background.</center>

当使用该纹理的淡化粒子时，我们得到一个简单的效果，看起来有点像白烟从地面冒出来。为了使它更有说服力，把<font color="green">emission rate </font>设置为100左右。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/particles/unlit-particles/textured-particles.png" alt="img" style="zoom:50%;" />

<center>Textured billboard particles, emission rate set to 100.</center>

#### 1.3、[Vertex Colors](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#1.3)

有可能每个粒子使用不同的颜色。最简单的演示方法是将起始颜色设置为在黑色和白色之间随机挑选。然而，这样做目前并不能改变粒子的外观。为了使其发挥作用，我们必须向我们的着色器添加对<font color="RoyalBlue">vertex colors</font>的支持。与其为粒子创建新的HLSL文件，不如在<font color="red">UnlitPass</font>中添加对它的支持。

第一步是添加一个具有<font color="red">COLOR</font>语义的<font color="green">float4</font>顶点属性。

```glsl
struct Attributes {
	float3 positionOS : POSITION;
	float4 color : COLOR;
	float2 baseUV : TEXCOORD0;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

把它也添加到<font color="RoyalBlue">Varyings</font>中，并通过<font color="red">UnlitPassVertex</font>传递，但只有在<font color="DarkOrchid ">_VERTEX_COLORS</font>被定义的情况下。这样我们就可以根据需要启用或禁用顶点颜色支持。

```glsl
struct Varyings {
	float4 positionCS : SV_POSITION;
	#if defined(_VERTEX_COLORS)
		float4 color : VAR_COLOR;
	#endif
	float2 baseUV : VAR_BASE_UV;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex (Attributes input) {
	…
	#if defined(_VERTEX_COLORS)
		output.color = input.color;
	#endif
	output.baseUV = TransformBaseUV(input.baseUV);
	return output;
}
```

接下来，在<font color="green">UnlitInput</font>中为<font color="red">InputConfig</font>添加一个颜色，默认设置为不透明的白色，并将其作为<font color="green">GetBase</font>的结果的因素。

```c#
struct InputConfig {
	float4 color;
	float2 baseUV;
};

InputConfig GetInputConfig (float2 baseUV) {
	InputConfig c;
	c.color = 1.0;
	c.baseUV = baseUV;
	return c;
}

…

float4 GetBase (InputConfig c) {
	float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, c.baseUV);
	float4 baseColor = INPUT_PROP(_BaseColor);
	return baseMap * baseColor * c.color;
}
```

回到<font color="green">UnlitPass</font>，如果<font color="red">UnlitPassFragment</font>中存在内插的顶点颜色，将其复制到<font color="green">config</font>中。

```glsl
	InputConfig config = GetInputConfig(input.baseUV);
	#if defined(_VERTEX_COLORS)
		config.color = input.color;
	#endif
```

为了最终给<font color="green">UnlitParticles</font>添加对顶点颜色的支持，给它添加一个切换着色器的属性。

```glsl
		[HDR] _BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
		[Toggle(_VERTEX_COLORS)] _VertexColors ("Vertex Colors", Float) = 0
```

以及定义该关键字的相应<font color="green">shader feature</font>。如果你想让普通的Unlit着色器也支持顶点颜色，你也可以这样做。

```glsl
			#pragma shader_feature _VERTEX_COLORS
```

<img src="E:/Typora%20file/SRP/assets/vertex-colors-without-sorting-1678468704961-93.png" style="zoom:50%;" />

<img src="E:/Typora%20file/SRP/assets/vertex-colors-with-sorting.png" alt="with" style="zoom:50%;" />

<center>Using vertex colors, without and with sorting by distance.</center>

我们现在得到了彩色的颗粒。在这一点上，粒子的分类成为一个问题。如果所有的粒子都有相同的颜色，它们的绘制顺序并不重要，但如果它们不同，我们就需要按距离来排序，以获得正确的结果。请注意，当基于距离进行排序时，粒子可能会因为视角位置的变化而突然交换绘制顺序，就像任何透明物体一样。

#### 1.4、[Flipbooks](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#1.4)

广告牌粒子可以通过循环使用不同的基础贴图来实现动画。Unity把这些称为<font color="green">flipbook particles</font>。这是通过使用<font color="red"> texture atlas</font>在一个规则的网格中铺设来实现的，就像这个含有4×4网格的循环噪声图案的纹理。

<img src="assets/Particles%20Flipbook.png" alt="Particles Flipbook" style="zoom:50%;" />

<center>Base map for particle flipbook, on black background.</center>

创建一个新的不发光的粒子材料，使用<font color="green"> flipbook map,</font>，然后复制我们的粒子系统，让它使用该<font color="green"> flipbook materia</font>。停用单粒子版本，这样我们就只能看到翻书系统。由于每个粒子现在代表一个小云，所以把它们的大小增加到2个左右。启用粒子系统的<font color="DarkOrchid ">*Texture Sheet Animation* </font>，将其配置为4×4的翻书，使其从随机帧开始，并在粒子的生命周期内经历一个周期。

通过在50%的时间内沿X和Y方向随机翻转粒子，从一个任意的旋转开始，并使粒子以随机速度旋转，可以增加额外的变化。

<img src="E:/Typora%20file/SRP/assets/particles-with-flipbook.png" alt="img" style="zoom:50%;" />

<center>Flipbook particle system.</center>

#### 1.5、[Flipbook Blending](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#1.5)

当系统处于运动状态时，很明显，粒子在几帧中循环，因为<font color="green">flipbook frame</font>的帧率非常低。对于寿命为5秒的粒子来说，每秒有3.2帧。这可以通过在连续的帧之间进行混合而得到平滑。这需要我们向着色器发送第二对UV坐标和一个动画混合系数。我们通过在渲染器模块中启用自定义顶点流来做到这一点。添加<font color="green">UV2和AnimBlend</font>。你也可以删除法线流，因为我们不需要它。

<img src="E:/Typora%20file/SRP/assets/custom-vertex-streams.png" alt="img" style="zoom:50%;" />

<center>Custom vertex streams.</center>

在添加流之后，将显示一个错误，表明粒子系统和它目前使用的着色器之间不匹配。在我们的着色器中使用这些流之后，这个错误就会消失。为<font color="green">UnlitParticle</font>添加一个着色器关键字切换属性，以控制我们是否支持<font color="red"> flipbook blending</font>。

```glsl
	[Toggle(_VERTEX_COLORS)] _VertexColors ("Vertex Colors", Float) = 0
	[Toggle(_FLIPBOOK_BLENDING)] _FlipbookBlending ("Flipbook Blending", Float) = 0
```

连同伴随的<font color="green">shader feature</font>。

```glsl
#pragma shader_feature _FLIPBOOK_BLENDING
```

如果<font color="green"> flipbook blending</font>被激活，两个UV对都通过<font color="red">TEXCOORD0</font>提供，所以它必须是一个float4而不是float2。混合系数是通过<font color="green">TEXCOORD1</font>提供的单个浮点数。

```glsl
struct Attributes {
	float3 positionOS : POSITION;
	float4 color : COLOR;
	#if defined(_FLIPBOOK_BLENDING)
		float4 baseUV : TEXCOORD0;
		float flipbookBlend : TEXCOORD1;
	#else
		float2 baseUV : TEXCOORD0;
	#endif
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

如果需要，我们将把新数据作为一个单一的<font color="red">float3 flipbookUVB</font>字段添加到<font color="green">Varyings</font>中。

```glsl
struct Varyings {
	…
	float2 baseUV : VAR_BASE_UV;
	#if defined(_FLIPBOOK_BLENDING)
		float3 flipbookUVB : VAR_FLIPBOOK;
	#endif
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

调整<font color="green">UnlitPassVertex</font>，使其在适当的时候复制所有相关数据。

```glsl
Varyings UnlitPassVertex (Attributes input) {
	…
	output.baseUV.xy = TransformBaseUV(input.baseUV.xy);
	#if defined(_FLIPBOOK_BLENDING)
		output.flipbookUVB.xy = TransformBaseUV(input.baseUV.zw);
		output.flipbookUVB.z = input.flipbookBlend;
	#endif
	return output;
}
```

在<font color="green">InputConfig</font>中也添加<font color="red">flipbookUVB</font>，以及一个布尔值来表示是否启用<font color="DarkOrchid ">flipbook</font>混合，这在默认情况下不是这样的。

```glsl
struct InputConfig {
	float4 color;
	float2 baseUV;
	float3 flipbookUVB;
	bool flipbookBlending;
};

InputConfig GetInputConfig (float2 baseUV) {
	…
	c.flipbookUVB = 0.0;
	c.flipbookBlending = false;
	return c;
}
```

如果启用了<font color="green">flipbook blending</font>，我们必须在<font color="green">GetBase</font>中对基础贴图进行第二次采样，使用<font color="red">flipbook UV</font>，然后根据混合系数从第一次采样到第二次采样进行插值。

```glsl
float4 GetBase (InputConfig c) {
	float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, c.baseUV);
	if (c.flipbookBlending) {
		baseMap = lerp(
			baseMap, SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, c.flipbookUVB.xy),
			c.flipbookUVB.z
		);
	}
	float4 baseColor = INPUT_PROP(_BaseColor);
	return baseMap * baseColor * c.color;
}
```

为了最终激活翻书混合，在适当的时候覆盖<font color="green">UnlitPassFragment</font>的默认配置。

```glsl
#if defined(_VERTEX_COLORS)
		config.color = input.color;
	#endif
	#if defined(_FLIPBOOK_BLENDING)
		config.flipbookUVB = input.flipbookUVB;
		config.flipbookBlending = true;
	#endif
```

<center><iframe src="https://gfycat.com/ifr/limpwelltododutchsmoushond?controls=0" style="border: none; overflow: hidden; width: 250px; height:250px;"></iframe></center>

<center>Flipbook blending.</center>

#### 2、[Fading Near Camera](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#2)

当摄像机在一个粒子系统内时，粒子最终会非常接近摄像机的近处，也会从一边传到另一边。粒子系统有一个<font color="green">Renderer / Max Particle Size</font>属性，可以防止单个广告牌粒子覆盖太多的窗口。一旦它们达到了最大的可见尺寸，它们就会出现滑出的情况，而不是在接近近处平面时变大。

另一种处理接近近平面的粒子的方法是根据它们的碎片深度来淡化它们。这在通过表现大气效果的粒子系统时看起来会更好。

#### 2.1、[Fragment Data](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#2.1)

我们已经在片段函数中提供了<font color="green">fragment depth </font>。它是通过具有<font color="red">SV_POSITION</font>语义的float4提供的。我们已经将它的XY分量用于抖动，但现在让我们正式说明，我们正在使用<font color="green">fragment data.</font>

在顶点函数中，<font color="green">SV_POSITION</font>表示顶点的<font color="red">clip-space position</font>，作为4D同质坐标。但在片段函数中，<font color="green">SV_POSITION</font>代表片段的<font color="red">screen-space</font>--也被称为<font color="red">window-space</font>--的位置。空间转换是由GPU执行的。为了明确这一点，我们把所有<font color="DarkOrchid ">Varyings</font>结构中的<font color="green">postionCS</font>重命名为<font color="RoyalBlue">positionCS_SS</font>。

```glsl
	float4 positionCS_SS : SV_POSITION;
```

在附带的<font color="green">vertex functions </font>中也要进行调整。

```glsl
output.positionCS_SS = TransformWorldToHClip(positionWS);
```

接下来，我们将介绍一个新的<font color="red">Fragment HLSL</font> include文件，其中包含一个<font color="green">Fragment</font>结构和一个<font color="green">GetFragment</font>函数，该函数在给定一个<font color="green">float4</font>随后返回该<font color="RoyalBlue">fragment</font>的<font color="DarkOrchid ">screen-space position vector</font>。最初，片段只有一个2D位置，它来自屏幕空间位置的XY分量。这些是带有0.5偏移的<font color="green">texel</font>坐标。屏幕左下角的texel是（0.5, 0.5），它右边的texel是（1.5, 0.5），以此类推。

```glsl
#ifndef FRAGMENT_INCLUDED
#define FRAGMENT_INCLUDED

struct Fragment {
	float2 positionSS;
};

Fragment GetFragment (float4 positionSS) {
	Fragment f;
	f.positionSS = positionSS.xy;
	return f;
}

#endif
```

把这个文件包含在所有其他包含语句之后的<font color="green">Common</font>中，然后调整<font color="red">ClipLOD</font>，使其第一个参数是<font color="green">Fragment</font>而不是float4。

```c#
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Packing.hlsl"

#include "Fragment.hlsl"

…

void ClipLOD (Fragment fragment, float fade) {
	#if defined(LOD_FADE_CROSSFADE)
		float dither = InterleavedGradientNoise(fragment.positionSS, 0);
		clip(fade + (fade < 0.0 ? dither : -dither));
	#endif
}
```

在这一点上，让我们也在<font color="green">Common</font>中定义常见的<font color="RoyalBlue"> linear and point clamp sampler </font>状态，因为我们以后会在多个地方使用这些东西。在包含<font color="red">Fragment</font>之前就这样做。

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Packing.hlsl"

SAMPLER(sampler_linear_clamp);
SAMPLER(sampler_point_clamp);

#include "Fragment.hlsl"
```

然后从<font color="green">PostFXStackPasses</font>中删除通用采样器的定义，因为它现在是重复的，会导致编译器错误。

```glsl
TEXTURE2D(_PostFXSource2);
//SAMPLER(sampler_linear_clamp);
```

接下来，在<font color="green">LitInput</font>和<font color="green">UnlitInput</font>的<font color="red">InputConfig</font>结构中都添加一个片段。然后将屏幕空间位置向量作为第一个参数添加到<font color="green">GetInputConfig</font>函数中，这样它们就可以用它来调用<font color="RoyalBlue">GetFragment</font>。

```glsl
struct InputConfig {
	Fragment fragment;
	…
};

InputConfig GetInputConfig (float4 positionSS, …) {
	InputConfig c;
	c.fragment = GetFragment(positionSS);
	…
}
```

在我们调用<font color="green">GetInputConfig</font>的所有地方添加该参数。

```glsl
	InputConfig config = GetInputConfig(input.positionCS_SS, …);
```

然后调整<font color="green">LitPassFragment</font>，让它在得到配置后调用<font color="RoyalBlue">ClipLOD</font>，这样它就可以把片段传给它。同时将片段的位置传递给<font color="red">InterleavedGradientNoise</font>，而不是直接使用<font color="DarkOrchid ">input.positionCS_SS</font>。

```c#
float4 LitPassFragment (Varyings input) : SV_TARGET {
	UNITY_SETUP_INSTANCE_ID(input);
	//ClipLOD(input.positionSS.xy, unity_LODFade.x);
	InputConfig config = GetInputConfig(input.positionCS_SS, input.baseUV);
	ClipLOD(config.fragment, unity_LODFade.x);
	
	…
	surface.dither = InterleavedGradientNoise(config.fragment.positionSS, 0);
	…
}
```

<font color="green">ShadowCasterPassFragment</font>也必须被改变，以便它在获得配置后夹住。

```glsl
void ShadowCasterPassFragment (Varyings input) {
	UNITY_SETUP_INSTANCE_ID(input);
	//ClipLOD(input.positionCS.xy, unity_LODFade.x);
	InputConfig config = GetInputConfig(input.positionCS_SS, input.baseUV);
	ClipLOD(config.fragment, unity_LODFade.x);
	
	float4 base = GetBase(config);
	#if defined(_SHADOWS_CLIP)
		clip(base.a - GetCutoff(config));
	#elif defined(_SHADOWS_DITHER)
		float dither = InterleavedGradientNoise(input.positionSS.xy, 0);
		clip(base.a - dither);
	#endif
}
```

#### 2.2、[Fragment Depth](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#2.2)

为了淡化相机附近的粒子，我们需要知道<font color="red">fragment's depth</font>。所以要给<font color="RoyalBlue">Fragment</font>添加一个<font color="DarkOrchid ">depth field</font>。

```c#
struct Fragment {
	float2 positionSS;
	float depth;
};
```

<font color="red">fragment's depth</font>被存储在<font color="RoyalBlue">screen-space position vector</font>的最后一个分量中。它是用于执行透视分割以将三维位置投射到屏幕上的值。这是视图空间的深度，所以它是与摄像机XY平面的距离，而不是它的近平面。

```c#
Fragment GetFragment (float4 positionCS_SS) {
	Fragment f;
	f.positionSS = positionSS.xy;
	f.depth = positionSS.w;
	return f;
}
```

我们可以通过直接返回<font color="green">LitPassFragment</font>和<font color="red">UnlitPassFragment</font>中的片段深度来验证这一点是否正确，缩放后我们可以看到它是一个灰度梯度。

```
InputConfig config = GetInputConfig(input.positionCS_SS, input.baseUV);
	return float4(config.fragment.depth.xxx / 20.0, 1.0);
```

<img src="E:/Typora%20file/SRP/assets/fragment-depth.png" alt="img" style="zoom:50%;" />

<center>Fragment depth, divided by 20.</center>

#### 2.3、[Orthographic Depth](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#2.3)

上述方法只在使用透视摄像机时有效。当使用<font color="green">orthographic camera </font>时，没有透视划分，因此屏幕空间位置向量的最后一个分量总是1。

我们可以通过在<font color="green">UnityInput</font>中添加一个float4 <font color="RoyalBlue">unity_OrthoParams</font>字段来确定我们是否在处理一个正射相机，Unity通过这个字段将正射相机的信息传达给GPU。

```c#
float4 unity_OrthoParams;
float4 _ProjectionParams;
```

如果是正视摄像机，它的最后一个分量将是1，否则将是0。在<font color="green">Common</font>中添加一个<font color="RoyalBlue">IsOrthographicCamera</font>函数，使用这个事实，在包含<font color="green">Fragment</font>之前定义，所以我们可以在那里使用它。

如果你永远不会使用<font color="green">orthographic cameras</font>，你可以把它<font color="red"> hard-code </font>为返回<font color="DarkOrchid ">false</font>，或者通过<font color="RoyalBlue">shader keyword</font>来控制它。

```c#
bool IsOrthographicCamera () {
	return unity_OrthoParams.w;
}

#include "Fragment.hlsl"
```

对于正交相机来说，我们能做的最好的事情就是依靠<font color="RoyalBlue">screen-space position vector</font>的<font color="red">Z分量，</font>它包含了<font color="DarkOrchid ">fragment</font>的转换后的<font color="RoyalBlue">clip-space depth</font>。这是用于深度比较的原始值，如果启用了深度写入功能，它会被写入深度缓冲区。它是一个在0-1范围内的值，对于<font color="RoyalBlue"> orthographic projections - 正交投影</font>来说是线性的。

为了把它转换为<font color="RoyalBlue">view-space depth</font>，我们必须用摄像机的近远距离来缩放它，然后加上近平面距离。近距离和远距离被存储在<font color="red">_ProjectionParams的Y和Z部分</font>。

[_ProjectionParams](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html) `x` 为 1.0（如果当前使用 翻转投影矩阵）, `y` 是相机的近平面， `z` 是相机的远平面和 `w` 是 1/远平面。

如果使用反转的<font color="RoyalBlue">depth buffer</font>，我们还需要反转原始深度。在一个新的<font color="green">OrthographicDepthBufferToLinear</font>函数中做到这一点，在包括<font color="red">Fragment</font>之前也定义在<font color="green">Common</font>中。

```c#
float OrthographicDepthBufferToLinear (float rawDepth) {
	#if UNITY_REVERSED_Z
		rawDepth = 1.0 - rawDepth;
	#endif
	return (_ProjectionParams.z - _ProjectionParams.y) * rawDepth + _ProjectionParams.y;
}

#include "Fragment.hlsl"
```

现在，<font color="green">GetFragment</font>可以检查是否使用了正射相机，如果是的话，就依靠<font color="RoyalBlue">OrthographicDepthBufferToLinear</font>来确定片段深度。

```c#
	f.depth = IsOrthographicCamera() ?
		OrthographicDepthBufferToLinear(positionSS.z) : positionSS.w;
```

<img src="E:/Typora%20file/SRP/assets/orthographic-depth.png" alt="img" style="zoom:50%;" />

<center>Fragment depth for orthographic camera.</center>

在验证了两种相机类型的片段深度都是正确的之后，从LitPassFragment和UnlitPassFragment中移除调试可视化。

```c#
	//return float4(config.fragment.depth.xxx / 20.0, 1.0);
```

#### 2.4、[Distance-Based Fading](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#2.4)

回到<font color="green">UnlitParticles</font>着色器，添加一个<font color="red">Near Fade</font>关键字的切换属性，以及使其距离和范围可配置的属性。距离决定了粒子应该在离摄像机平面多近的地方完全消失。

这是<font color="green">camera plane</font>，而不是它的<font color="RoyalBlue">near place</font>。所以应该使用一个至少是靠近平面的值。1是一个合理的默认值。范围控制<font color="RoyalBlue">transition region</font>的长度，在这个范围内，粒子将线性地淡出。同样，1是一个合理的默认值，至少它必须是一个小的正值。

```c#
		[Toggle(_NEAR_FADE)] _NearFade ("Near Fade", Float) = 0
		_NearFadeDistance ("Near Fade Distance", Range(0.0, 10.0)) = 1
		_NearFadeRange ("Near Fade Range", Range(0.01, 10.0)) = 1
```

增加一个着色器功能，以启用近似褪色的功能。

```c#
		#pragma shader_feature _NEAR_FADE
```

然后在<font color="red">UnlitInput</font>的<font color="RoyalBlue">UnityPerMaterial</font>缓冲区中包括距离和范围。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float, _NearFadeDistance)
	UNITY_DEFINE_INSTANCED_PROP(float, _NearFadeRange)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
	UNITY_DEFINE_INSTANCED_PROP(float, _ZWrite)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

接下来，在<font color="green">InputConfig</font>中添加一个<font color="RoyalBlue">boolean nearFade</font>字段，以控制近距离渐变是否被激活，默认情况下它不是。

```glsl
struct InputConfig {
	…
	bool nearFade;
};

InputConfig GetInputConfig (float4 positionCC_SS, float2 baseUV) {
	…
	c.nearFade = false;
	return c;
}
```

在摄像机附近的淡化是通过简单地减少<font color="green"> fragment's base alpha</font>来完成的。

<font color="DarkOrchid ">attenuation factor</font>等于<font color="green">fragment depth minus the fade distance片段深度减去淡化距离</font>，然后除以<font color="RoyalBlue"> fade range淡化范围</font>。

由于结果可能是负的，所以在将其计入<font color="green">base map</font>的<font color="RoyalBlue">alpha</font>之前，要对其进行饱和处理。在适当的时候，在<font color="red">GetBase</font>中这样做。

```glsl
if (c.flipbookBlending) { … }
	if (c.nearFade) {
		float nearAttenuation = (c.fragment.depth - INPUT_PROP(_NearFadeDistance)) /
			INPUT_PROP(_NearFadeRange);
		baseMap.a *= saturate(nearAttenuation);
	}
```

最后，为了激活该功能，如果定义了<font color="green">_NEAR_FADE</font>关键字设置为真，在<font color="green">UnlitPassFragment</font>中把片段的<font color="red">nearFade</font>字段。

```glsl
#if defined(_FLIPBOOK_BLENDING)
		config.flipbookUVB = input.flipbookUVB;
		config.flipbookBlending = true;
	#endif
	#if defined(_NEAR_FADE)
		config.nearFade = true;
	#endif
```

<center><iframe src="https://gfycat.com/ifr/inbornhollowantarcticgiantpetrel?controls=0" style="border: none; overflow: hidden; width: 350px; height:350px;"></iframe></center>

<center>Adjusting near fade distance.</center>

### 3、Soft Particles

当广告牌粒子与几何体相交时，尖锐的过渡在视觉上很刺眼，而且使它们的平面性质很明显。解决这个问题的方法是使用软粒子，当它们后面有不透明的几何体时，软粒子就会淡出。为了使其发挥作用，粒子的碎片深度必须与先前在摄像机缓冲区的同一位置上绘制的任何东西的深度进行比较。这意味着我们必须对深度缓冲区进行采样。

#### 3.1、[Separate Depth Buffer](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.1)

到目前为止，我们总是为摄像机使用一个单一的帧缓冲器，其中包含<font color="DarkOrchid ">color and depth information</font>。这是典型的帧缓冲区配置，但<font color="DarkOrchid ">color and depth information</font>总是存储在不同的缓冲区中，被称为<font color="red"> frame buffer attachments. - 帧缓冲区附件</font>。为了访问深度缓冲区，我们需要单独定义这些附件。

第一步是用两个标识符替换<font color="green">CameraRenderer</font>中的<font color="red">**_CameraFrameBuffer**</font>，我们将其命名为<font color="DarkOrchid ">**_CameraColorAttachment**</font>和<font color="DarkOrchid ">**_CameraDepthAttachment**</font>。

```c#
	//static int frameBufferId = Shader.PropertyToID("_CameraFrameBuffer");
	static int
		colorAttachmentId = Shader.PropertyToID("_CameraColorAttachment"),
		depthAttachmentId = Shader.PropertyToID("_CameraDepthAttachment");
```

在Render中，我们现在必须把<font color="DarkOrchid "> color attachment - 颜色附件</font>传递给<font color="green">PostFXStack.Render</font>，这在功能上与我们之前所做的相当。

```c#
if (postFXStack.IsActive) {
			postFXStack.Render(colorAttachmentId);
		}
```

在设置中，我们现在必须得到两个缓冲区而不是一个复合缓冲区。<font color="green"> color buffer has no depth</font>，而深度缓冲区的格式是<font color="RoyalBlue">RenderTextureFormat.Depth</font>，其过滤模式是<font color="red">FilterMode.Point</font>，因为混合深度数据没有意义。

两个附件都可以通过一次调用<font color="DarkOrchid ">SetRenderTarget</font>来设置，对每个附件使用相同的加载和存储动作。

SetRenderTarget

```c#
	if (postFXStack.IsActive) {
			if (flags > CameraClearFlags.Color) {
				flags = CameraClearFlags.Color;
			}
			buffer.GetTemporaryRT(
				colorAttachmentId, camera.pixelWidth, camera.pixelHeight,
				0, FilterMode.Bilinear, useHDR ?
					RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default
			);
			buffer.GetTemporaryRT(
				depthAttachmentId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
			buffer.SetRenderTarget(
				colorAttachmentId,
				RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store,
				depthAttachmentId,
				RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
			);
		}
```

这两个缓冲区也必须被释放。一旦这样做了，我们的RP仍然以同样的方式工作，但现在有了帧<font color="green">buffer attachments</font>，我们可以单独访问。

```c#
	void Cleanup () {
		lighting.Cleanup();
		if (postFXStack.IsActive) {
			buffer.ReleaseTemporaryRT(colorAttachmentId);
			buffer.ReleaseTemporaryRT(depthAttachmentId);
		}
	}
```

#### 3.2、[Copying Depth](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.2)

我们不能在<font color="green">depth buffer</font>被用于渲染的同时对其进行采样。我们必须对它进行复制。所以引入一个<font color="red">_CameraDepthTexture</font>标识符，并添加一个布尔字段来指示我们是否在使用深度纹理。我们应该只在需要的时候才去复制深度，这一点我们会在得到<font color="RoyalBlue"> camera settings.</font>后在<font color="green">Render </font>中确定。但我们最初会简单地总是启用它。

```c#
static int
		colorAttachmentId = Shader.PropertyToID("_CameraColorAttachment"),
		depthAttachmentId = Shader.PropertyToID("_CameraDepthAttachment"),
		depthTextureId = Shader.PropertyToID("_CameraDepthTexture");

	…

	bool useDepthTexture;

	public void Render (…) {
		…
		CameraSettings cameraSettings =
			crpCamera ? crpCamera.Settings : defaultCameraSettings;

		useDepthTexture = true;

		…
	}
```

创建一个新的<font color="red">CopyAttachments</font>方法，如果需要的话，可以获得一个临时的重复的深度纹理，并将深度附件数据复制到它上面。这可以通过调用命令缓冲区上的<font color="green">CopyTexture</font>与源纹理和目标纹理来完成。这比通过全屏绘制调用来做要有效得多。还要确保在<font color="red">Cleanup</font>中释放多余的深度纹理。

```c#
void Cleanup () {
		…
		if (useDepthTexture) {
			buffer.ReleaseTemporaryRT(depthTextureId);
		}
	}

	…

	void CopyAttachments () {
		if (useDepthTexture) {
			buffer.GetTemporaryRT(
				depthTextureId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
			buffer.CopyTexture(depthAttachmentId, depthTextureId);
			ExecuteBuffer();
		}
	}
```

我们只复制一次附件，<font color="RoyalBlue">在所有不透明的几何体被绘制之后</font>，所以在<font color="green">Render</font>中的<font color="red">skybox</font>之后。这意味着深度纹理只有在渲染透明物体时才能使用。

```c#
		context.DrawSkybox(camera);
		CopyAttachments();
```

#### 3.3、[Copying Depth Without Post FX](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.3)

复制深度只有在我们有<font color="DarkOrchid ">depth attachment</font>可以复制的情况下才有效，目前只有在启用<font color="green">post FX </font>的情况下才有效。为了在没有<font color="green">post FX </font>的情况下实现这个功能，我们还需要在使用<font color="red">depth texture</font>时使用一个中间帧缓冲器。

引入一个<font color="RoyalBlue">useIntermediateBuffer</font>布尔字段来跟踪这一点，在可能获得附件之前在设置中初始化。现在，当使用<font color="DarkOrchid ">depth texture is used or post FX</font>时，就应该这样做。清理工作也会受到同样的影响。

```c#
bool useDepthTexture, useIntermediateBuffer;
	
	…
	
	void Setup () {
		context.SetupCameraProperties(camera);
		CameraClearFlags flags = camera.clearFlags;
		
		useIntermediateBuffer = useDepthTexture || postFXStack.IsActive;
		if (useIntermediateBuffer) {
			if (flags > CameraClearFlags.Color) {
				flags = CameraClearFlags.Color;
			}
			…
		}

		…
	}

	void Cleanup () {
		lighting.Cleanup();
		if (useIntermediateBuffer) {
			buffer.ReleaseTemporaryRT(colorAttachmentId);
			buffer.ReleaseTemporaryRT(depthAttachmentId);
		//}
			if (useDepthTexture) {
				buffer.ReleaseTemporaryRT(depthTextureId);
			}
		}
	}
```

但是现在，当没有激活<font color="green">post FX </font>时，渲染失败了，因为我们只渲染到了<font color="green">intermediate buffer</font>。

我们必须执行最后的拷贝到<font color="RoyalBlue">camera's target.</font>。不幸的是，我们只能使用<font color="DarkOrchid ">CopyTexture</font>来复制到<font color="red">render texture</font>，而不是复制到<font color="RoyalBlue">final frame buffer</font>。

我们可以使用<font color="green">PostFX copy pass</font>来完成这个任务，但是这个步骤是专门针对<font color="red"> camera renderer </font>的，所以我们要为它创建一个专门的<font color="red">CameraRenderer Shader</font>。它的启动方式与<font color="green">PostFX</font>着色器相同，但只有一个<font color="DarkOrchid ">copy pass</font>，并且包括它自己的<font color="red">HLSL文件</font>。

```c#
Shader "Hidden/Custom RP/Camera Renderer" {
	
	SubShader {
		Cull Off
		ZTest Always
		ZWrite Off
		
		HLSLINCLUDE
		#include "../ShaderLibrary/Common.hlsl"
		#include "CameraRendererPasses.hlsl"
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

新的<font color="green">CameraRendererPasses </font>HLSL文件具有与<font color="red">PostFXStackPasses</font>相同的<font color="RoyalBlue">Varyings</font>结构和<font color="RoyalBlue">DefaultPassVertex</font>函数。它也有一个<font color="red">_SourceTexture</font>纹理和一个<font color="RoyalBlue">CopyPassFragment</font>函数，简单地返回采样的源纹理。

```c#
#ifndef CUSTOM_CAMERA_RENDERER_PASSES_INCLUDED
#define CUSTOM_CAMERA_RENDERER_PASSES_INCLUDED

TEXTURE2D(_SourceTexture);

struct Varyings { … };

Varyings DefaultPassVertex (uint vertexID : SV_VertexID) { … }

float4 CopyPassFragment (Varyings input) : SV_TARGET {
	return SAMPLE_TEXTURE2D_LOD(_SourceTexture, sampler_linear_clamp, input.screenUV, 0);
}

#endif
```

接下来，给<font color="green">CameraRenderer</font>添加一个材质域。为了初始化它，创建一个带有着色器参数的公共构造方法，让它以着色器为参数调用<font color="red">CoreUtils.CreateEngineMaterial</font>。

该方法创建了一个新的材质，并将其设置为在<font color="RoyalBlue">hidden in the editor编辑器中隐藏</font>，以确保它不会被保存为资产，所以我们不需要自己明确地做这个。如果着色器丢失，它还会记录一个错误。

[CoreUtils](https://docs.unity3d.com/Packages/com.unity.render-pipelines.core@10.0/api/UnityEngine.Rendering.CoreUtils.html)

```c#
Material material;

	public CameraRenderer (Shader shader) {
		material = CoreUtils.CreateEngineMaterial(shader);
	}
```

同时添加一个公共的<font color="red">Dispose</font>方法，通过将材料传递给<font color="green">CoreUtils.Destroy</font>来摆脱它。该方法会定期或立即销毁该材料，这取决于Unity是否处于游戏模式。

我们需要这样做，因为<font color="RoyalBlue">每当RP资产被修改时，新的RP实例和渲染器就会被创建，这可能会导致编辑器中创建许多材质。</font>

```c#
	public void Dispose () {
		CoreUtils.Destroy(material);
	}
```

现在<font color="green">CustomRenderPipeline</font>必须在构造它的<font color="red">renderer</font>时提供一个<font color="red">Shader</font>。所以我们将在它自己的<font color="DarkOrchid ">constructor method - 构造方法</font>中进行，同时为相机渲染器着色器添加一个参数。

```c#
	CameraRenderer renderer; // = new CameraRenderer();

	…

	public CustomRenderPipeline (
		bool allowHDR,
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		bool useLightsPerObject, ShadowSettings shadowSettings,
		PostFXSettings postFXSettings, int colorLUTResolution, Shader cameraRendererShader
	) {
		…
		renderer = new CameraRenderer(cameraRendererShader);
	}
```

而且从现在开始，当<font color="RoyalBlue"> renderer </font>自己被<font color="RoyalBlue">disposed</font>时，它也必须对其调用<font color="red">Dispose</font>。我们已经为它创建了一个<font color="red">Dispose</font>方法，但只针对编辑器代码。把这个方法重命名为<font color="green">DisposeForEditor</font>，并且只让它重置<font color="green"> light mapping delegate</font>。

```c#
partial void DisposeForEditor ();

#if UNITY_EDITOR
	
	…
	
	partial void DisposeForEditor () {
		//base.Dispose(disposing);
		Lightmapping.ResetDelegate();
	}
```

然后添加一个新的<font color="red">Dispose</font>方法，这个方法不是只针对编辑器的，它调用它的<font color="RoyalBlue"> base implementation - 基本实现</font>、编辑器的版本，<font color="DarkOrchid ">finally disposes the renderer.</font>

```c#
	protected override void Dispose (bool disposing) {
		base.Dispose(disposing);
		DisposeForEditor();
		renderer.Dispose();
	}
```

而在顶层，<font color="green">CustomRenderPipelineAsset</font>必须获得一个着色器配置属性，并将其传递给<font color="DarkOrchid "> pipeline constructor</font>。然后我们就可以最后挂上着色器了。

```c#
	[SerializeField]
	Shader cameraRendererShader = default;

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			allowHDR, useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows, postFXSettings, (int)colorLUTResolution,
			cameraRendererShader
		);
	}
```

<img src="E:/Typora%20file/SRP/assets/camera-renderer-shader.png" alt="img" style="zoom:50%;" />

<center>Camera renderer shader assigned.</center>

在这一点上，<font color="red">CameraRenderer</font>有一个功能性的材质。同时给它添加<font color="green">_SourceTexture</font>标识符，并给它一个类似于<font color="green">PostFXStack</font>中的<font color="green">Draw</font>方法，只是没有<font color="RoyalBlue">pass</font>参数，因为我们此时只有一个<font color="RoyalBlue">pass</font>。

```c#
static int
		colorAttachmentId = Shader.PropertyToID("_CameraColorAttachment"),
		depthAttachmentId = Shader.PropertyToID("_CameraDepthAttachment"),
		depthTextureId = Shader.PropertyToID("_CameraDepthTexture"),
		sourceTextureId = Shader.PropertyToID("_SourceTexture");
	
	…
	
	void Draw (RenderTargetIdentifier from, RenderTargetIdentifier to) {
		buffer.SetGlobalTexture(sourceTextureId, from);
		buffer.SetRenderTarget(
			to, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
		buffer.DrawProcedural(
			Matrix4x4.identity, material, 0, MeshTopology.Triangles, 3
		);
	}
```

为了最后修复我们的渲染器，通过调用<font color="green">Draw</font>将颜色附件复制到<font color="red">Render</font>中的<font color="red">camera target</font>上，如果后期特效没有激活，但我们确实使用了一个<font color="red"> intermediate buffer.</font>

```c#
	if (postFXStack.IsActive) {
			postFXStack.Render(colorAttachmentId);
		}
		else if (useIntermediateBuffer) {
			Draw(colorAttachmentId, BuiltinRenderTextureType.CameraTarget);
			ExecuteBuffer();
		}
```

#### 3.4、[Reconstructing View-Space Depth](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.4)

为了对深度纹理进行采样，我们需要<font color="RoyalBlue"> UV coordinates of the fragment - 片段的UV</font>，这是在屏幕空间中。我们可以通过它的<font color="RoyalBlue">position</font>除以<font color="RoyalBlue">screen pixel dimensions, - 屏幕像素尺寸</font>来找到这些坐标，Unity通过float4 <font color="red">_ScreenParams</font>的XY分量来提供这些坐标，所以把它加到<font color="green">UnityInput</font>中。

[_ScreenParams](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html) `x` 是相机目标纹理的宽度 **像素**
, `y` 是相机目标纹理的高度（以像素为单位）， `z` 为 1.0 + 1.0/宽度和 `w` 为 1.0 + 1.0/高度。

```c#
float4 unity_OrthoParams;
float4 _ProjectionParams;
float4 _ScreenParams;
```

然后我们可以将<font color="RoyalBlue"> UV of the fragment - 片段的UV</font>和<font color="RoyalBlue">buffer depth</font>添加到<font color="red">Fragment</font>中。

通过<font color="RoyalBlue">SAMPLE_DEPTH_TEXTURE_LOD</font>宏，用<font color="DarkOrchid "> point clamp sampler</font>对摄像机深度纹理进行取样，从而获取<font color="DarkOrchid ">buffer depth </font>。这个宏的作用与<font color="red">SAMPLE_TEXTURE2D_LOD</font>相同，但只返回R通道。

```c#
TEXTURE2D(_CameraDepthTexture);

struct Fragment {
	float2 positionSS;
	float2 screenUV;
	float depth;
	float bufferDepth;
};

Fragment GetFragment (float4 positionCS_SS) {
	Fragment f;
	f.positionSS = positionSS.xy;
	f.screenUV = f.positionSS / _ScreenParams.xy;
	f.depth = IsOrthographicCamera() ?
		OrthographicDepthBufferToLinear(positionSS.z) : positionSS.w;
	f.bufferDepth =
		SAMPLE_DEPTH_TEXTURE_LOD(_CameraDepthTexture, sampler_point_clamp, f.screenUV, 0);
	return f;
}
```

这就给了我们原始的<font color="red"> depth buffer</font>值。为了将其转换为<font color="RoyalBlue">view-space depth</font>，我们可以再次调用<font color="green">OrthographicDepthBufferToLinear</font>，如果是正视相机的话，就像对当前<font color="green">fragment's depth</font>一样。

透视深度也必须被转换，为此我们可以使用<font color="red">LinearEyeDepth</font>。它需要<font color="green">_ZBufferParams</font>作为第二个参数。

[_ZBufferParams](https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html) 用于线性化 Z 缓冲区值。 `x` 是（1远/近）， `y` 是（远/近）， `z` 是 （x/far） 和 `w` 是 （y/far）。

```glsl
	f.bufferDepth = LOAD_TEXTURE2D(_CameraDepthTexture, f.positionSS).r;
	f.bufferDepth = IsOrthographicCamera() ?
		OrthographicDepthBufferToLinear(f.bufferDepth) :
		LinearEyeDepth(f.bufferDepth, _ZBufferParams);
```

<font color="red">_ZBufferParams</font>是Unity提供的另一个<font color="green">float4</font>，包含从原始深度到线性深度的转换系数。把它添加到<font color="red">UnityInput</font>中。

```glsl
float4 unity_OrthoParams;
float4 _ProjectionParams;
float4 _ScreenParams;
float4 _ZBufferParams;
```

为了检查我们对缓冲区深度的采样是否正确，在<font color="green">UnlitPassFragment</font>中按比例返回，就像我们之前测试片段深度一样。

```glsl
InputConfig config = GetInputConfig(input.positionCS_SS, input.baseUV);
return float4(config.fragment.bufferDepth.xxx / 20.0, 1.0);
```

<img src="E:/Typora%20file/SRP/assets/buffered-depth-perspective.png" alt="perspective" style="zoom:50%;" />

 <center><img src="E:/Typora%20file/SRP/assets/buffered-depth-orthographic.png" alt="orthographic" style="zoom:50%;" /></center>

<center>Buffer depth, perspective and orthographic projections.</center>

一旦明确了采样深度是正确的，就删除调试的可视化内容。

```glsl
//return float4(config.fragment.bufferDepth.xxx / 20.0, 1.0);
```

#### 3.5、[Optional Depth Texture](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.5)

复制深度需要额外的工作，特别是在没有使用后期特效的情况下，因为这也需要中间缓冲区和一个额外的拷贝到摄像机目标。

因此，让我们对我们的RP是否支持复制深度进行配置。我们将为此创建一个新的<font color="green">CameraBufferSettings</font>结构，放在自己的文件中，用于分组所有与摄像机缓冲区有关的设置。

除了复制深度的切换外，还将允许<font color="red">HDR</font>的切换也放在其中。此外，还引入了一个单独的开关来控制<font color="RoyalBlue">rendering reflection</font>时是否复制深度。这很有用，因为反射是在没有后期特效的情况下渲染的，而且粒子系统也不会出现在反射中，所以为<font color="DarkOrchid ">depth for reflections</font>是很昂贵的，而且可能没有用。我们让它成为可能是因为深度也可以用于其他效果，这在反射中可能是可见的。即使如此，请记住，<font color="green"> depth buffer</font>在每个立方体贴图的反射面都是不同的，所以沿着立方体贴图的边缘会有深度接缝。

```c#
[System.Serializable]
public struct CameraBufferSettings {

	public bool allowHDR;

	public bool copyDepth, copyDepthReflections;
}
```

用这些相机缓冲区设置替换<font color="red">CustomRenderPipelineAsset</font>的当前HDR切换。

```c#
	//[SerializeField]
	//bool allowHDR = true;

	[SerializeField]
	CameraBufferSettings cameraBuffer = new CameraBufferSettings {
		allowHDR = true
	};

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			cameraBuffer, useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows, postFXSettings, (int)colorLUTResolution,
			cameraRendererShader
		);
	}
```

把这个变化也应用于<font color="red">CustomRenderPipeline</font>。

```c#
	//bool allowHDR;
	CameraBufferSettings cameraBufferSettings;

	…

	public CustomRenderPipeline (
		CameraBufferSettings cameraBufferSettings,
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		bool useLightsPerObject, ShadowSettings shadowSettings,
		PostFXSettings postFXSettings, int colorLUTResolution, Shader cameraRendererShader
	) {
		this.colorLUTResolution = colorLUTResolution;
		//this.allowHDR = allowHDR;
		this.cameraBufferSettings = cameraBufferSettings;
		…
	}

	protected override void Render (ScriptableRenderContext context, Camera[] cameras) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera, cameraBufferSettings,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings, postFXSettings, colorLUTResolution
			);
		}
	}
```

<font color="red">CameraRenderer.Render</font>现在必须根据它是否在渲染反射来使用适当的设置。

```c#
	public void Render (
		ScriptableRenderContext context, Camera camera,
		CameraBufferSettings bufferSettings,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings, PostFXSettings postFXSettings,
		int colorLUTResolution
	) {
		…
		
		//useDepthTexture = true;
		if (camera.cameraType == CameraType.Reflection) {
			useDepthTexture = bufferSettings.copyDepthReflection;
		}
		else {
			useDepthTexture = bufferSettings.copyDepth;
		}

		…
		useHDR = bufferSettings.allowHDR && camera.allowHDR;

		…
	}
```

<img src="E:/Typora%20file/SRP/assets/camera-buffer-settings.png" alt="img" style="zoom:50%;" />

<center>Camera buffer settings, with HDR and non-reflection copy depth enabled.</center>

除了整个RP的设置，我们还可以在<font color="red">CameraSettings</font>中添加一个复制深度的切换，默认情况下是启用的。

```c#
	public bool copyDepth = true;
```

<img src="E:/Typora%20file/SRP/assets/camera-copy-depth.png" alt="img" style="zoom:50%;" />

<center>Camera copy depth toggle.</center>

那么对于普通相机来说，只有在RP和相机都启用了深度纹理的情况下才会使用，这与HDR的控制方式类似。

```c#
	if (camera.cameraType == CameraType.Reflection) {
			useDepthTexture = bufferSettings.copyDepthReflection;
		}
		else {
			useDepthTexture = bufferSettings.copyDepth && cameraSettings.copyDepth;
		}
```

#### 3.6、[Missing Texture](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.6)

由于<font color="green"> depth texture</font>是可选的，它可能不存在。当着色器对它进行采样时，结果将是随机的。它可能是一个空的纹理或一个旧的副本，可能是另一个相机的。也有可能在不透明的渲染阶段，着色器过早地对深度纹理进行采样。

我们至少可以做的是确保无效的采样会产生一致的结果。我们通过在<font color="red">CameraRender</font>的构造方法中创建一个默认的缺失纹理来做到这一点。没有针对纹理的<font color="green">CoreUtils</font>方法，所以我们将自己设置其隐藏标志为<font color="red">HideFlags.HideAndDontSave</font>。将其命名为<font color="green">Missing</font>，这样在通过帧调试器检查着色器属性时，就能明显看出使用了错误的纹理。让它成为一个简单的1×1纹理，所有通道设置为0.5。当渲染器被弃置时，还要适当地销毁它。

```c#
	Texture2D missingTexture;

	public CameraRenderer (Shader shader) {
		material = CoreUtils.CreateEngineMaterial(shader);
		missingTexture = new Texture2D(1, 1) {
			hideFlags = HideFlags.HideAndDontSave,
			name = "Missing"
		};
		missingTexture.SetPixel(0, 0, Color.white * 0.5f);
		missingTexture.Apply(true, true);
	}

	public void Dispose () {
		CoreUtils.Destroy(material);
		CoreUtils.Destroy(missingTexture);
	}
```

在 "<font color="green">Setup</font>"结束时，将缺失的纹理用于深度纹理。

```c#
	void Setup () {
		…
		buffer.BeginSample(SampleName);
		buffer.SetGlobalTexture(depthTextureId, missingTexture);
		ExecuteBuffer();
	}
```

#### 3.7、[Fading Particles Nearby Background](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.7)

现在我们有了一个功能性的深度纹理，我们可以继续前进，最终支持软粒子。第一步是为<font color="red">UnlitParticles</font>的软粒子关键字切换、<font color="RoyalBlue">distance, and a range </font>添加着色器属性，类似于<font color="RoyalBlue">near fade properties</font>。在这种情况下，距离是从粒子后面的东西开始测量的，所以我们默认将其设置为零。

```glsl
	[Toggle(_SOFT_PARTICLES)] _SoftParticles ("Soft Particles", Float) = 0
		_SoftParticlesDistance ("Soft Particles Distance", Range(0.0, 10.0)) = 0
		_SoftParticlesRange ("Soft Particles Range", Range(0.01, 10.0)) = 1
```

也为它添加<font color="green"> shader feature </font>。

```glsl
		#pragma shader_feature _SOFT_PARTICLES
```

就像对于<font color="RoyalBlue">near fading</font>，如果定义了关键字，在<font color="red">UnlitPassFragment</font>中设置一个适当的配置字段为真。

```glsl
#if defined(_NEAR_FADE)
		config.nearFade = true;
	#endif
	#if defined(_SOFT_PARTICLES)
		config.softParticles = true;
	#endif
```

在<font color="red">UnlitInput</font>中，将新的着色器属性添加到<font color="green">UnityPerMaterial</font>，并将该字段添加到<font color="green">InputConfig</font>。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _SoftParticlesDistance)
	UNITY_DEFINE_INSTANCED_PROP(float, _SoftParticlesRange)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
	UNITY_DEFINE_INSTANCED_PROP(float, _ZWrite)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

#define INPUT_PROP(name) UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, name)

struct InputConfig {
	…
	bool softParticles;
};

InputConfig GetInputConfig (float4 positionCC_SS, float2 baseUV) {
	…
	c.softParticles = false;
	return c;
}
```

然后在<font color="red">GetBase</font>中应用另一个<font color="green">near attenuation</font>，这次是基于片段的<font color="green">buffer depth</font>减去它自己的<font color="RoyalBlue">depth</font>。

```glsl
	if (c.nearFade) {
		float nearAttenuation = (c.fragment.depth - INPUT_PROP(_NearFadeDistance)) /
			INPUT_PROP(_NearFadeRange);
		baseMap.a *= saturate(nearAttenuation);
	}
	if (c.softParticles) {
		float depthDelta = c.fragment.bufferDepth - c.fragment.depth;
		float nearAttenuation = (depthDelta - INPUT_PROP(_SoftParticlesDistance)) /
			INPUT_PROP(_SoftParticlesRange);
		baseMap.a *= saturate(nearAttenuation);
	}
```

<center><iframe src="https://gfycat.com/ifr/marrieduglyaardvark?controls=0" style="border: none; overflow: hidden; width: 350px; height: 350px;"></iframe></center>

<center>Soft particles, adjusting fade range.</center>

#### 3.8、[No Copy Texture Support](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.8)

这一切都很好，但只有在通过<font color="red">CopyTexture</font>直接复制纹理的情况下才支持，至少在基本水平上是如此。大多数情况下是这样的，但对于WebGL 2.0来说不是这样。因此，如果我们也想支持WebGL 2.0，我们就必须退回到通过我们的着色器进行复制，这样做效率较低，但至少可以工作。

通过<font color="green">CameraRenderer</font>中的一个静态布尔字段来跟踪<font color="red">CopyTexture</font>是否被支持。最初将其设置为<font color="green">false</font>，这样我们就可以测试后退的方法，尽管我们的开发机器都支持它。

```c#
	static bool copyTextureSupported = false;
```

在<font color="red">CopyAttachments</font>中，如果支持的话，通过<font color="green">CopyTexture</font>复制深度，否则就退回到使用我们的<font color="green">Draw</font>方法。

```c#
	void CopyAttachments () {
		if (useDepthTexture) {
			buffer.GetTemporaryRT(
				depthTextureId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
			if (copyTextureSupported) {
				buffer.CopyTexture(depthAttachmentId, depthTextureId);
			}
			else {
				Draw(depthAttachmentId, depthTextureId);
			}
			ExecuteBuffer();
		}
	}
```

这起初不能产生正确的结果，因为<font color="green">Draw</font>改变了渲染目标，所以进一步的绘制会出错。我们必须在之后将渲染目标设回<font color="red">camera buffer</font>，再次加载我们的附件。

```c#
	if (copyTextureSupported) {
				buffer.CopyTexture(depthAttachmentId, depthTextureId);
			}
			else {
				Draw(depthAttachmentId, depthTextureId);
				buffer.SetRenderTarget(
					colorAttachmentId,
					RenderBufferLoadAction.Load, RenderBufferStoreAction.Store,
					depthAttachmentId,
					RenderBufferLoadAction.Load, RenderBufferStoreAction.Store
				);
			}
```

第二件出错的事情是，深度根本就没有被复制，因为我们的复制通道只写到默认的着色器目标，而这个目标是针对颜色数据的，不是深度。为了复制深度，我们需要给<font color="green">CameraRenderer</font>着色器添加第二个<font color="red"> copy-depth pass</font>，以写入深度而不是颜色。我们通过将它的<font color="green">ColorMask</font>设置为0并打开<font color="red">ZWrite</font>来实现。它还需要一个特殊的片段函数，我们将其命名为<font color="green">CopyDepthPassFragment</font>。

```c#
	Pass {
			Name "Copy Depth"

			ColorMask 0
			ZWrite On
			
			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment CopyDepthPassFragment
			ENDHLSL
		}
```

新的<font color="green">fragment </font>函数必须对深度进行采样，并将其作为一个具有<font color="red">SV_DEPTH</font>语义的<font color="green">**float**</font>返回，而不是具有<font color="green">SV_TARGET</font>语义的浮点数4。这样，我们对原始深度缓冲区的值进行采样，并直接将其用于新的片段的深度。

```glsl
float4 CopyPassFragment (Varyings input) : SV_TARGET {
	return SAMPLE_TEXTURE2D_LOD(_SourceTexture, sampler_linear_clamp, input.screenUV, 0);
}

float CopyDepthPassFragment (Varyings input) : SV_DEPTH {
	return SAMPLE_DEPTH_TEXTURE_LOD(_SourceTexture, sampler_point_clamp, input.screenUV, 0);
}
```

接下来，回到<font color="green">CameraRenderer</font>，为<font color="red">Draw</font>添加一个布尔参数，以指示我们是否从<font color="green">drawing from and to depth</font>，默认设置为假。如果是，就使用第二遍，而不是第一遍。

```c#
	public void Draw (
		RenderTargetIdentifier from, RenderTargetIdentifier to, bool isDepth = false
	) {
		…
		buffer.DrawProcedural(
			Matrix4x4.identity, material, isDepth ? 1 : 0, MeshTopology.Triangles, 3
		);
	}
```

然后在<font color="DarkOrchid ">CopyAttachments</font>复制<font color="green">depth buffer</font>的时候表明我们正在处理深度问题。

```c#
		Draw(depthAttachmentId, depthTextureId, true);
```

在确定这种方法也能工作之后，通过检查<font color="red">SystemInfo.copyTextureSupport</font>来确定是否支持<font color="RoyalBlue">CopyTexture</font>。任何大于无的支持水平都是足够的。

```c#
static bool copyTextureSupported =
	SystemInfo.copyTextureSupport > CopyTextureSupport.None。
```

#### 3.9、[Gizmos and Depth](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#3.9)

现在我们有了绘制深度的方法，我们可以在结合后期特效或使用深度纹理时使用它来使我们的小玩意再次具有深度意识。在<font color="red">DrawGizmosBeforeFX</font>中，在绘制第一个小精灵之前，如果我们使用中间缓冲区，就把深度复制到摄像机目标上。

```c#
partial void DrawGizmosBeforeFX () {
		if (Handles.ShouldRenderGizmos()) {
			if (useIntermediateBuffer) {
				Draw(depthAttachmentId, BuiltinRenderTextureType.CameraTarget, true);
				ExecuteBuffer();
			}
			context.DrawGizmos(camera, GizmoSubset.PreImageEffects);
		}
	}
```

![img](E:/Typora%20file/SRP/assets/gizmos-depth.png)

<center>Gizmos recognizing depth.</center>

### 4、[Distortion]()

我们还将支持Unity粒子的另一个特性是失真，它可以用来创造诸如由热引起的大气折射的效果。这需要对颜色缓冲区进行采样，就像我们已经对深度缓冲区进行采样一样，但要加上一个UV偏移。

#### 4.1、[Color Copy Texture](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#4.1)

我们首先在<font color="green">CameraBufferSettings</font>中加入复制颜色的切换，同样是为普通相机和反射相机单独设置。

```c#
	public bool copyColor, copyColorReflection, copyDepth, copyDepthReflection;
```

<img src="E:/Typora%20file/SRP/assets/camera-buffer-settings-color.png" alt="img" style="zoom:50%;" />

<center>Copying color and depth.</center>

让<font color="green">copying color</font>也可以在每个相机上配置。

```c#
	public bool copyColor = true, copyDepth = true;
```

<img src="E:/Typora%20file/SRP/assets/camera-copy-color.png" alt="img" style="zoom:50%;" />

<center>Also enabled for camera.</center>

<font color="red">CameraRendering</font>现在还必须跟踪<font color="DarkOrchid ">color texture</font>的标识符以及是否使用了<font color="DarkOrchid ">color texture</font>。

```c#
colorTextureId = Shader.PropertyToID("_CameraColorTexture"),
		depthTextureId = Shader.PropertyToID("_CameraDepthTexture"),
		sourceTextureId = Shader.PropertyToID("_SourceTexture");
	
	…
	
	bool useColorTexture, useDepthTexture, useIntermediateBuffer;

	…

	public void Render (…) {
		…

		if (camera.cameraType == CameraType.Reflection) {
			useColorTexture = bufferSettings.copyColorReflection;
			useDepthTexture = bufferSettings.copyDepthReflection;
		}
		else {
			useColorTexture = bufferSettings.copyColor && cameraSettings.copyColor;
			useDepthTexture = bufferSettings.copyDepth && cameraSettings.copyDepth;
		}

		…
	}
```

我们现在是否使用<font color="green"> intermediate buffer</font>也取决于是否使用了<font color="green">color texture </font>。而且我们最初也应该把<font color="green">color texture </font>设置为缺失的纹理。在清理的时候也要释放它。

```c#
void Setup () {
		…

		useIntermediateBuffer =
			useColorTexture || useDepthTexture || postFXStack.IsActive;
		…
		buffer.BeginSample(SampleName);
		buffer.SetGlobalTexture(colorTextureId, missingTexture);
		buffer.SetGlobalTexture(depthTextureId, missingTexture);
		ExecuteBuffer();

	}

	void Cleanup () {
		lighting.Cleanup();
		if (useIntermediateBuffer) {
			buffer.ReleaseTemporaryRT(colorAttachmentId);
			buffer.ReleaseTemporaryRT(depthAttachmentId);
			if (useColorTexture) {
				buffer.ReleaseTemporaryRT(colorTextureId);
			}
			if (useDepthTexture) {
				buffer.ReleaseTemporaryRT(depthTextureId);
			}
		}
	}
```

我们现在需要在使用<font color="red">color or a depth texture</font>，或两者都使用时<font color="RoyalBlue"> camera attachments - 复制相机附件</font>。让我们使<font color="green">CopyAttachments</font>的调用依赖于此。

```c#
context.DrawSkybox(camera);
		if (useColorTexture || useDepthTexture) {
			CopyAttachments();
		}
```

然后我们可以让它分别<font color="green"> copy both textures </font>，之后重置<font color="red">render target </font>并执行一次<font color="RoyalBlue">buffer </font>。

```c#
void CopyAttachments () {
		if (useColorTexture) {
			buffer.GetTemporaryRT(
				colorTextureId, camera.pixelWidth, camera.pixelHeight,
				0, FilterMode.Bilinear, useHDR ?
					RenderTextureFormat.DefaultHDR : RenderTextureFormat.Default
			);
			if (copyTextureSupported) {
				buffer.CopyTexture(colorAttachmentId, colorTextureId);
			}
			else {
				Draw(colorAttachmentId, colorTextureId);
			}
		}
		if (useDepthTexture) {
			buffer.GetTemporaryRT(
				depthTextureId, camera.pixelWidth, camera.pixelHeight,
				32, FilterMode.Point, RenderTextureFormat.Depth
			);
			if (copyTextureSupported) {
				buffer.CopyTexture(depthAttachmentId, depthTextureId);
			}
			else {
				Draw(depthAttachmentId, depthTextureId, true);
				//buffer.SetRenderTarget(…);
			}
			//ExecuteBuffer();
		}
		if (!copyTextureSupported) {
			buffer.SetRenderTarget(
				colorAttachmentId,
				RenderBufferLoadAction.Load, RenderBufferStoreAction.Store,
				depthAttachmentId,
				RenderBufferLoadAction.Load, RenderBufferStoreAction.Store
			);
		}
		ExecuteBuffer();
	}
```

#### 4.2、[Sampling the Buffer Color](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#4.2)

为了对摄像机的<font color="green">color texture</font>进行采样，将其添加到<font color="green">Fragment</font>中。我们不会给<font color="green">Fragment</font>添加一个缓冲区的颜色属性，因为我们对其确切位置的颜色不感兴趣。

相反，我们引入一个<font color="green">GetBufferColor</font>函数，以片段和<font color="red"> UV offset </font>为参数，重新调整采样的颜色。

```c#
TEXTURE2D(_CameraColorTexture);
TEXTURE2D(_CameraDepthTexture);

struct Fragment { … };

Fragment GetFragment (float4 positionCS_SS) { … }

float4 GetBufferColor (Fragment fragment, float2 uvOffset = float2(0.0, 0.0)) {
	float2 uv = fragment.screenUV + uvOffset;
	return SAMPLE_TEXTURE2D_LOD(_CameraColorTexture, sampler_linear_clamp, uv);
}
```

为了测试这一点，在<font color="green">UnlitPassFragment</font>中返回缓冲区的颜色，在两个维度上都有一个小的偏移，比如5%。

```c#
	InputConfig config = GetInputConfig(input.positionCS_SS, input.baseUV);
	return GetBufferColor(config.fragment, 0.05);
```

<img src="E:/Typora%20file/SRP/assets/color-buffer-with-offset.png" alt="img" style="zoom:50%;" />

<center>Sampling camera color buffer with offset.</center>

请注意，由于颜色是在不透明阶段之后复制的，透明物体在其中消失了。因此，粒子抹去了所有在它们之前绘制的透明物体，包括彼此。同时深度在这种情况下不起作用，所以比碎片本身更靠近相机平面的碎片的颜色也被复制了。当明确了它的作用后，删除调试的可视化。

```c#
//return GetBufferColor(config.fragment, 0.05);
```

#### 4.3、[Distortion Vectors](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#4.3)

为了创造一个有用的变形效果，我们需要一个平滑过渡的变形矢量图。这里是一个用于单个圆形粒子的简单地图。它是一个法线图，所以把它导入。

![img](E:/Typora%20file/SRP/assets/particles-single-distortion.png)

<center>Particle distortion map.</center>

为<font color="green">UnlitParticles</font>添加一个关键字切换着色器属性，以及扭曲图和强度属性。扭曲将作为屏幕空间的UV偏移来应用，所以需要小的值。让我们使用0-0.2的强度范围，其中0.1为默认值。

```glsl
		[Toggle(_DISTORTION)] _Distortion ("Distortion", Float) = 0
		[NoScaleOffset] _DistortionMap("Distortion Vectors", 2D) = "bumb" {}
		_DistortionStrength("Distortion Strength", Range(0.0, 0.2)) = 0.1
```

<img src="E:/Typora%20file/SRP/assets/distortion-enabled.png" alt="img" style="zoom:50%;" />

<center>Distortion enabled.</center>

添加所需的着色器功能。

```glsl
	#pragma shader_feature _DISTORTION
```

然后将<font color="green">distortion map and strength</font>属性添加到<font color="green">UnlitInput</font>。

```glsl
TEXTURE2D(_BaseMap);
TEXTURE2D(_DistortionMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _SoftParticlesRange)
	UNITY_DEFINE_INSTANCED_PROP(float, _DistortionStrength)
	…
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

引入一个新的<font color="red">GetDistortion</font>函数，返回一个<font color="green">float2</font>向量。让它对失真图进行采样，并像对基本图那样应用翻书式混合，然后解码由<font color="green">distortion strength</font>缩放的法线。我们只需要该向量的XY分量，所以放弃Z。

```glsl
float2 GetDistortion (InputConfig c) {
	float4 rawMap = SAMPLE_TEXTURE2D(_DistortionMap, sampler_BaseMap, c.baseUV);
	if (c.flipbookBlending) {
		rawMap = lerp(
			rawMap, SAMPLE_TEXTURE2D(_DistortionMap, sampler_BaseMap, c.flipbookUVB.xy),
			c.flipbookUVB.z
		);
	}
	return DecodeNormal(rawMap, INPUT_PROP(_DistortionStrength)).xy;
}
```

在<font color="green">UnlitPassFragment</font>中，如果扭曲被启用，则检索它并将其作为偏移量来获得缓冲区的颜色，覆盖基础颜色。在剪裁之后做这个。

```glsl
	float4 base = GetBase(config);
	#if defined(_CLIPPING)
		clip(base.a - GetCutoff(config));
	#endif
	#if defined(_DISTORTION)
		float2 distortion = GetDistortion(config);
		base = GetBufferColor(config.fragment, distortion);
	#endif
```

<img src="E:/Typora%20file/SRP/assets/distorted-color-buffer.png" alt="img" style="zoom:50%;" />

<center>Distorted color buffer.</center>

其结果是，粒子在径向上扭曲了<font color="DarkOrchid ">color texture</font>，除了它们的角落，因为那里的扭曲向量是零。但是，<font color="green">distortion effect</font>应该取决于粒子的视觉强度，这是由原来的基础α控制的。所以要用基础alpha来调制扭曲偏移矢量。

```glsl
	float2 distortion = GetDistortion(config) * base.a;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/particles/distortion/modulated-distortion.png" alt="img" style="zoom:50%;" />

<center>Modulated distortion.</center>

在这一点上，我们仍然得到了硬的边缘，<font color="green">betraying</font>粒子完全相互重叠，并且是矩形的。我们通过保持粒子的原始阿尔法来隐藏这一点。

```glsl
	base.rgb = GetBufferColor(config.fragment, distortion).rgb;
```

<img src="E:/Typora%20file/SRP/assets/faded-distortion.png" alt="img" style="zoom:50%;" />

<center>Faded distortion.</center>

现在，扭曲的颜色纹理样本也消退了，这使得未扭曲的背景和其他粒子又部分可见。结果是一个平滑的混乱，没有物理意义，但足以提供大气折射的幻觉。这可以通过调整扭曲强度来进一步改善，同时通过在粒子的生命周期内调整它们的颜色来平滑地淡入淡出。另外，偏移矢量与屏幕对齐，不受粒子方向的影响。因此，如果粒子在它们的生命周期中被设置为旋转，它们各自的变形模式就会出现扭曲。

#### 4.4、[Distortion Blend](https://catlikecoding.com/unity/tutorials/custom-srp/particles/#4.4)

目前，当扭曲被启用时，我们完全替换了粒子的原始颜色，只保留了它们的alpha。粒子的颜色可以通过各种方式与<font color="green">distorted color buffer</font>相结合。我们将添加一个简单的<font color="RoyalBlue">distortion blend shader</font>属性，在粒子本身的颜色和它引起的扭曲之间进行插值，使用的方法与Unity的粒子着色器相同。

```glsl
	_DistortionStrength("Distortion Strength", Range(0.0, 0.2)) = 0.1
	_DistortionBlend("Distortion Blend", Range(0.0, 1.0)) = 1
```

<img src="E:/Typora%20file/SRP/assets/distortion-blend.png" alt="img" style="zoom:50%;" />

<center>Distortion blend slider.</center>

将该属性添加到<font color="green">UnlitInput</font>中，同时添加一个函数来获取它。

```glsl
	UNITY_DEFINE_INSTANCED_PROP(float, _DistortionStrength)
	UNITY_DEFINE_INSTANCED_PROP(float, _DistortionBlend)

…

float GetDistortionBlend (InputConfig c) {
	return INPUT_PROP(_DistortionBlend);
}
```

我们的想法是，当混合滑块在1的时候，我们只看到<font color="green">distortion</font>。降低它使粒子的颜色出现，但它不会完全隐藏变形。

相反，我们根据粒子的α值减去<font color="green">blend slider</font>的饱和度，从扭曲到粒子的颜色进行插值。因此，当变形被启用时，粒子本身的颜色将总是较弱，与变形被禁用时相比显得较小，除非它是完全不透明的。在<font color="green">UnlitPassFragment</font>中执行插值。

```glsl
	#if defined(_DISTORTION)
		float2 distortion = GetDistortion(config) * base.a;
		base.rgb = lerp(
			GetBufferColor(config.fragment, distortion).rgb, base.rgb,
			saturate(base.a - GetDistortionBlend(config))
		);
	#endif
```

这对更复杂的粒子来说看起来更好，就像我们的翻页书的例子。因此，这里有一个用于翻书的变形纹理。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/particles/distortion/particles-flipbook-distortion.png" alt="img" style="zoom:50%;" />

<center>Distortion map for particle flipbook.</center>

这可以用来创造有趣的失真效果。现实的效果会很微妙，因为当系统处于运动状态时，一点失真就足够了。但为了演示的目的，我把效果做得很强，所以它们在视觉上很明显，甚至在截图中也是如此。

<center><iframe src="https://gfycat.com/ifr/digitalcontentirrawaddydolphin?controls=0" style="border: none; overflow: hidden; width: 350px; height:  350px;"></iframe></center>

<center>Distortion with flipbook and post FX.</center>

#### 4.5、[Fixing Nonstandard Cameras]()

我们目前的方法在只使用单个摄像机时是有效的，但在渲染到中间纹理而不使用后期特效时就会失败。发生这种情况是因为我们正在执行一个常规的复制到<font color="green">camera target</font>，这忽略了<font color="red">viewport and final blend </font>模式。所以<font color="green">CameraRenderer</font>也需要一个<font color="red">FinalPass</font>方法。

它是<font color="green">PostFXStack.FinalPass</font>的副本，只是我们将使用常规的复制传递，所以我们应该在事后将混合模式设置回1-0，以不影响其他复制动作。<font color="green">source texture</font>始终是<font color="RoyalBlue"> color attachment</font>，最终混合模式成为一个参数。

```c#
	void DrawFinal (CameraSettings.FinalBlendMode finalBlendMode) {
		buffer.SetGlobalFloat(srcBlendId, (float)finalBlendMode.source);
		buffer.SetGlobalFloat(dstBlendId, (float)finalBlendMode.destination);
		buffer.SetGlobalTexture(sourceTextureId, colorAttachmentId);
		buffer.SetRenderTarget(
			BuiltinRenderTextureType.CameraTarget,
			finalBlendMode.destination == BlendMode.Zero ?
				RenderBufferLoadAction.DontCare : RenderBufferLoadAction.Load,
			RenderBufferStoreAction.Store
		);
		buffer.SetViewport(camera.pixelRect);
		buffer.DrawProcedural(
			Matrix4x4.identity, material, 0, MeshTopology.Triangles, 3
		);
		buffer.SetGlobalFloat(srcBlendId, 1f);
		buffer.SetGlobalFloat(dstBlendId, 0f);
	}
```

在这种情况下，我们将混合模式着色器属性命名为<font color="red">_CameraSrcBlend和_CameraSrcBlend</font>。

```c#
	sourceTextureId = Shader.PropertyToID("_SourceTexture"),
		srcBlendId = Shader.PropertyToID("_CameraSrcBlend"),
		dstBlendId = Shader.PropertyToID("_CameraDstBlend");
```

调整<font color="green">CameraRenderer</font>的拷贝通道，使之依赖于这些属性。

```glsl
	Pass {
			Name "Copy"

			Blend [_CameraSrcBlend] [_CameraDstBlend]

			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment CopyPassFragment
			ENDHLSL
		}
```

最后，在<font color="green">Render</font>中调用<font color="green">DrawFinal</font>而不是<font color="green">Draw</font>。

```glsl
if (postFXStack.IsActive) {
			postFXStack.Render(colorAttachmentId);
		}
		else if (useIntermediateBuffer) {
			DrawFinal(cameraSettings.finalBlendMode);
			ExecuteBuffer();
		}
```

注意，<font color="red">颜色和深度纹理</font>只包含当前相机渲染的内容。畸变粒子和类似的效果不会从其他相机中获取数据。