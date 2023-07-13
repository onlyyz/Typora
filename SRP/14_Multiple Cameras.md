# 14_Multiple Cameras

用不同的后期特效设置来渲染多个摄像机。
		用自定义混合法对摄像机进行分层。
		支持渲染图层遮罩。
		每个摄像机的遮罩灯。

![img](E:/Typora%20file/SRP/assets/tutorial-image-1678466669969-1.jpg)

<center>Looking at the same scene in different ways.</center>

### 1、Combining Cameras

由于删减、光线处理和阴影渲染是在每个摄像机上进行的，所以每一帧尽可能少地渲染摄像机是个好主意，最好只有一个。但有时我们确实需要同时渲染多个不同的视角。这方面的例子包括多人分屏游戏、后视镜、自上而下的叠加、游戏中的摄像机和3D人物肖像。

#### 1.1、Split Screen

让我们首先考虑一个分屏场景，由两个并排的摄像机组成。左边的摄像机的视口矩形宽度设置为0.5。右边的摄像机的宽度也是0.5，其X位置设置为0.5。如果我们不使用后期特效，那么这就会像预期的那样工作。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/combining-cameras/split-screen-without-post-fx.png" alt="img" style="zoom:67%;" />

<center>Split screen without post FX, showing two different views of the same scene.</center>

但如果我们启用后期特效，就会失败。两台摄像机都以正确的尺寸渲染，但最终覆盖了整个摄像机的目标缓冲区，只有最后一台摄像机可见。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/combining-cameras/split-screen-with-post-fx-incorrect.png" alt="img" style="zoom:50%;" />

<center>Split screen with post FX, incorrect.</center>

这是因为<font color="green">SetRenderTarget</font>的调用也会重置视口以覆盖整个目标。

为了将视口应用到<font color="DarkOrchid ">final post FX pass </font>中，我们必须在设置目标后和绘制前设置<font color="red">viewport</font>。让我们通过复制<font color="RoyalBlue">PostFXStack.Draw</font>，将其重命名为<font color="green">DrawFinal</font>，并在<font color="red">SetRenderTarget</font>之后直接在缓冲区上调用<font color="green">SetViewport</font>，将摄像机的<font color="red">pixelRect</font>作为参数。由于这是最终的绘制，我们可以用硬编码的值替换所有的参数，除了源参数。

```c#
	void DrawFinal (RenderTargetIdentifier from) {
		buffer.SetGlobalTexture(fxSourceId, from);
		buffer.SetRenderTarget(
			BuiltinRenderTextureType.CameraTarget,
			RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
		buffer.SetViewport(camera.pixelRect);
		buffer.DrawProcedural(
			Matrix4x4.identity, settings.Material,
			(int)Pass.Final, MeshTopology.Triangles, 3
		);
	}
```

在<font color="green">DoColorGradingAndToneMapping</font>的结尾处调用新方法而不是常规的Draw。

```c#
	void DoColorGradingAndToneMapping (int sourceId) {
		…
		Draw(…)
		DrawFinal(sourceId);
		buffer.ReleaseTemporaryRT(colorGradingLUTId);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/combining-cameras/split-screen-with-post-fx-correct.png" alt="img" style="zoom:67%;" />

<center>Split screen with post FX, correct.</center>

#### 1.2、[Layering Cameras](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.2)

除了对不同的区域进行渲染，我们还可以使摄像机视口重叠。最简单的例子是使用一个覆盖整个屏幕的常规主摄像机，然后添加第二个摄像机，在稍后的渲染中使用相同的视图，但视口较小。我将第二个视口缩小到一半大小，并将其XY位置设置为0.25，使其居中。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/combining-cameras/two-camera-layers.png)

<center>Two camera layers.</center>

如果我们不使用后期特效，那么我们可以将顶部的相机层设置为仅<font color="DarkOrchid ">partially-transparent - 透明</font>的覆盖，通过设置它只清除深度。

这样就可以去掉它的天幕，露出下面的层。但是，当使用后期特效时，这就不起作用了，因为那时我们会强制它变成<font color="green">CameraClearFlags.Color</font>，所以我们会看到摄像机的背景颜色，而默认的是深蓝色。

<center><img src="E:/Typora%20file/SRP/assets/clear-depth-only-without-post-fx.png" alt="without" style="zoom:50%;" /></center>

 <center><img src="E:/Typora%20file/SRP/assets/clear-depth-only-with-post-fx.png" alt="with" style="zoom:50%;" /></center>

<center>Second camera set to clear depth, without and with post FX.</center>

我们可以做的一件事是改变<font color="green">PostFXStack</font>着色器的最终通道，使其执行alpha混合，而不是默认的<font color="green">One Zero</font>模式，从而使图层透明度与后期特效一起工作。

```glsl
Pass {
			Name "Final"

			Blend SrcAlpha OneMinusSrcAlpha
			
			HLSLPROGRAM
				#pragma target 3.5
				#pragma vertex DefaultPassVertex
				#pragma fragment FinalPassFragment
			ENDHLSL
		}
```

这确实需要我们在<font color="DarkOrchid ">FinalDraw</font>中一直加载目标缓冲区。

```c#
	void DrawFinal (RenderTargetIdentifier from) {
		buffer.SetGlobalTexture(fxSourceId, from);
		buffer.SetRenderTarget(
			BuiltinRenderTextureType.CameraTarget,
			RenderBufferLoadAction.Load, RenderBufferStoreAction.Store
		);
		…
	}
```

现在把<font color="DarkOrchid ">overlay camera's</font>的背景颜色的<font color="red">alpha</font>设置为零。这似乎是可行的，只要我们关闭了bloom。我添加了两个非常明亮的发光物体，以使它明显地显示出绽放是否被激活。

<center><img src="E:/Typora%20file/SRP/assets/bloom-disabled.png" alt="disabled" style="zoom:50%;" /> </center>

<center><img src="E:/Typora%20file/SRP/assets/bloom-enabled.png" alt="enabled" style="zoom:50%;" /></center>

<center>Bloom disabled and enabled.</center>

它不能与bloom一起使用，因为这种效果目前并不保留透明度。我们可以通过调整最终的<font color="green">bloom pass</font>来解决这个问题，使其保持<font color="RoyalBlue">original transparency from the high resolution source texture</font>。我们必须同时调整<font color="red">BloomAddPassFragment和BloomScatterFinalPassFragment</font>，因为两者都可以用于最终绘制。

```glsl
float4 BloomAddPassFragment (Varyings input) : SV_TARGET {
	…
	float4 highRes = GetSource2(input.screenUV);
	return float4(lowRes * _BloomIntensity + highRes.rgb, highRes.a);
}

…

float4 BloomScatterFinalPassFragment (Varyings input) : SV_TARGET {
	…
	float4 highRes = GetSource2(input.screenUV);
	lowRes += highRes.rgb - ApplyBloomThreshold(highRes.rgb);
	return float4(lerp(highRes.rgb, lowRes, _BloomIntensity), highRes.a);
}
```

<img src="E:/Typora%20file/SRP/assets/layered-with-bloom.png" alt="img" style="zoom:50%;" />

<center>Layered with transparency and bloom.</center>

透明度现在可以和<font color="green">bloom</font>一起使用，但bloom对透明区域的贡献不再可见。我们可以通过将最后一次传递切换为预乘法alpha混合来保留光晕。这确实需要我们将相机的背景颜色设置为纯透明的黑色，因为它将被添加到下面的图层。

```glsl
	Name "Final"

			Blend One OneMinusSrcAlpha
```

<img src="E:/Typora%20file/SRP/assets/bloom-premultiplied.png" alt="img" style="zoom:50%;" />

<center>Bloom affects transparent areas.</center>

#### 1.3、[Layered Alpha](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.3)

我们目前的分层方法只有在我们的Shader产生合理的阿尔法值并与相机图层混合的情况下才能发挥作用。

我们先前并不关心<font color="green"> written alpha </font>值，因为我们从未将它们用于任何用途。但是现在，如果两个alpha值为0.5的物体最终渲染到同一个texel上，这个texel的最终alpha值应该是0.25。而当其中一个alpha值为1时，结果应该总是1。

而当第二个alpha值为0时，应该保留原始alpha。所有这些情况都可以通过在混合alpha时使用OneMinusSrcAlpha来解决。我们可以将alpha通道的着色器混合模式与颜色分开配置，方法是在颜色混合模式后面加上一个逗号和alpha的模式。对我们的<font color="green">Lit和Unlit着色器</font>的常规通道都要这样做。

```c#
	Blend [_SrcBlend] [_DstBlend], One OneMinusSrcAlpha
```

只要使用适当的阿尔法值，这就可以了，这通常意味着写入深度的对象也应该总是产生1的阿尔法。这对不透明材质来说似乎很简单，但如果它们最终使用了一个也包含不同α值的基础贴图，就会出问题。对于剪辑材质来说，它也会出错，因为它们依靠alpha阈值来丢弃片段。如果一个片段被剪掉了，它就会正常，但如果它没有被剪掉，它的alpha就应该变成1。

![img](E:/Typora%20file/SRP/assets/cubes-alpha-zero.png)

<center>Opaque cubes with alpha zero add to the base layer instead of replacing it.</center>

确保alpha对我们的着色器表现正确的最快方法是在UnityPerMaterial缓冲区中添加<font color="red">_ZWrite</font>，包括<font color="green">LitInput和UnlitInput</font>。

```glsl
UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
	UNITY_DEFINE_INSTANCED_PROP(float, _ZWrite)
```

然后在两个输入文件中添加一个带阿尔法参数的<font color="red">GetFinalAlpha</font>函数。如果<font color="DarkOrchid ">_ZWrite</font>被设置为1，它就返回1，否则就返回所提供的值。

```c#
float GetFinalAlpha (float alpha) {
	return INPUT_PROP(_ZWrite) ? 1.0 : alpha;
}
```

通过<font color="green">LitPassFragment</font>中的这个函数过滤表面的阿尔法，在最后得到正确的阿尔法值。

```glsl
float4 LitPassFragment (Varyings input) : SV_TARGET {
	…
	return float4(color, GetFinalAlpha(surface.alpha));
}
```

并对<font color="green">UnlitPassFragment</font>中的基本alpha做同样的处理。

```glsl
float4 UnlitPassFragment (Varyings input) : SV_TARGET {
	…
	return float4(base.rgb, GetFinalAlpha(base.a));
}
```

#### 1.4、[Custom Blending](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.4)

与前一个摄像机层的混合只对叠加摄像机有意义。底层摄像机将与摄像机目标的初始内容相混合，这些内容要么是随机的，要么是以前的帧的累积，除非编辑器提供一个清除的目标。

所以第一台摄像机应该使用一零模式进行混合。为了支持替换、叠加和更多奇特的分层选项，我们将为摄像机添加一个可配置的最终混合模式，在启用后期特效时使用。我们将为这些设置创建一个新的可序列化的配置<font color="red">CameraSettings</font>类，就像我们为阴影做的那样。为了方便起见，将源和目的混合模式都包裹在一个内部的<font color="green">FinalBlendMode</font>结构中，然后将其默认设置为<font color="RoyalBlue">One Zero</font>混合模式。

```c#
using System;
using UnityEngine.Rendering;

[Serializable]
public class CameraSettings {
	
	[Serializable]
	public struct FinalBlendMode {

		public BlendMode source, destination;
	}

	public FinalBlendMode finalBlendMode = new FinalBlendMode {
		source = BlendMode.One,
		destination = BlendMode.Zero
	};
}
```

我们不能直接将这些设置添加到摄像机组件中，所以我们将创建一个补充的<font color="green">ustomRenderPipelineCamera</font>组件。它只能被添加到作为摄像机的游戏对象中一次，而且只有一次。给它一个<font color="red">CameraSettings</font>配置字段，并附带有<font color="green">getter</font>属性。

因为设置是一个类，该属性必须确保一个类的存在，所以如果需要的话，创建一个新的设置对象实例。如果这个组件还没有被编辑器序列化，或者在运行时给摄像机添加了一个组件，就会出现这种情况。

```c#
using UnityEngine;

[DisallowMultipleComponent, RequireComponent(typeof(Camera))]
public class CustomRenderPipelineCamera : MonoBehaviour {

	[SerializeField]
	CameraSettings settings = default;

	public CameraSettings Settings => settings ?? (settings = new CameraSettings());
}
```

现在我们可以在<font color="green">CameraRenderer.Render</font>的开始处获得相机的<font color="red">CustomRenderPipelineCamera</font>组件。为了支持没有自定义设置的相机，我们将检查我们的组件是否存在。

如果存在，我们就使用它的设置，否则我们就使用一个默认的设置对象，我们创建一次，并在一个静态字段中存储一个引用。然后我们在设置<font color="DarkOrchid ">stack.</font>的时候传递最终的<font color="green"> final blend</font>。

```c#
	static CameraSettings defaultCameraSettings = new CameraSettings();

	…

	public void Render (…) {
		this.context = context;
		this.camera = camera;

		var crpCamera = camera.GetComponent<CustomRenderPipelineCamera>();
		CameraSettings cameraSettings =
			crpCamera ? crpCamera.Settings : defaultCameraSettings;
		
		…
			postFXStack.Setup(
			context, camera, postFXSettings, useHDR, colorLUTResolution,
			cameraSettings.finalBlendMode
		);
		…
	}
```

<font color="green">PostFXStack</font>现在必须跟踪摄像机的最终混合模式。

```c#
CameraSettings.FinalBlendMode finalBlendMode;

	…
	
	public void Setup (
		ScriptableRenderContext context, Camera camera, PostFXSettings settings,
		bool useHDR, int colorLUTResolution, CameraSettings.FinalBlendMode finalBlendMode
	) {
		this.finalBlendMode = finalBlendMode;
		…
	}
```

所以它可以在<font color="green">DrawFinal</font>开始时设置新的<font color="green">_FinalSrcBlend和_FinalDstBlend</font>浮点着色器属性。另外，我们只需要关心在目标混合模式不为零的情况下加载<font color="DarkOrchid ">目标缓冲区</font>的问题。

```c#
int
		finalSrcBlendId = Shader.PropertyToID("_FinalSrcBlend"),
		finalDstBlendId = Shader.PropertyToID("_FinalDstBlend");
	
	…
	
	void DrawFinal (RenderTargetIdentifier from) {
		buffer.SetGlobalFloat(finalSrcBlendId, (float)finalBlendMode.source);
		buffer.SetGlobalFloat(finalDstBlendId, (float)finalBlendMode.destination);
		buffer.SetGlobalTexture(fxSourceId, from);
		buffer.SetRenderTarget(
			BuiltinRenderTextureType.CameraTarget,
			finalBlendMode.destination == BlendMode.Zero ?
				RenderBufferLoadAction.DontCare : RenderBufferLoadAction.Load,
			RenderBufferStoreAction.Store
		);
		…
	}
```

最后，在<font color="red">final pass </font>使用新的属性而不是硬编码的混合模式。

```
		Name "Final"

			Blend [_FinalSrcBlend] [_FinalDstBlend]
```

从现在开始，没有我们设置的摄像机将覆盖目标缓冲区的内容，因为默认的最终混合模式是一零。覆盖相机必须被赋予不同的最终混合模式，通常是<font color="green">One OneMinusSrcAlpha</font>。

<img src="E:/Typora%20file/SRP/assets/camera-settings.png" alt="img" style="zoom:50%;" />

<center>Component with settings for overlay camera.</center>

#### 1.5、[Render Textures](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.5)

除了创建分屏显示或直接分层摄像机外，使用摄像机在游戏中显示或作为GUI的一部分也很常见。在这些情况下，摄像机的目标必须是一个渲染纹理，要么是资产，要么是在运行时创建的。作为一个例子，我通过Assets / Create / Render Texture创建了一个200×100的渲染纹理。我<font color="red">没有给它深度缓冲</font>，因为我给它渲染了一个带有后期特效的摄像机，它创建了自己的带有深度缓冲的中间渲染纹理。

<img src="E:/Typora%20file/SRP/assets/render-texture.png" alt="img" style="zoom:50%;" />

<center>Render texture asset.</center>

然后我创建了一个<font color="red">Render Texture Camera</font>相机，通过与相机的目标纹理属性挂钩，将场景渲染到这个纹理上。

<img src="E:/Typora%20file/SRP/assets/target-texture.png" alt="img" style="zoom:50%;" />

<center>Camera target texture set.</center>

与常规渲染一样，底部摄像机必须使用<font color="green">One Zero</font>作为其最终混合模式。编辑器最初会呈现一个清晰的黑色纹理，但之后的渲染纹理会包含最后渲染到它的任何内容。多个摄像机可以在任何视口下向同一个渲染纹理进行渲染，就像正常情况一样。唯一不同的是，Unity会自动渲染具有渲染<font color="DarkOrchid ">texture targets</font>的摄像机，然后再渲染到显示器上。先渲染有<font color="green">texture targets</font>的摄像机，然后再渲染没有目标纹理的摄像机，深度依次增加。

#### 1.6、[Unity UI](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.6)

渲染纹理可以像任何常规纹理一样使用。为了通过Unity的用户界面显示它，我们必须使用一个带有原始图像组件的游戏对象，通过GameObject / UI / Raw Image创建。

<img src="E:/Typora%20file/SRP/assets/raw-image-inspector-1678467291993-37.png" alt="inspector" style="zoom:50%;" />

<img src="E:/Typora%20file/SRP/assets/raw-image-game.png" alt="game" style="zoom:50%;" />

<center>UI raw image, partially overlapping a button.</center>

原始图像使用的是默认的UI材质，它执行标准的<font color="green">SrcAlpha OneMinusSrcAlpha</font>混合。因此，透明度是可行的，但是bloom不是加法的，除非纹理显示为像素级的完美<font color="green"> bilinear filtering</font>，否则会使相机的黑色背景颜色在透明的边缘呈现出暗色的轮廓。

为了支持其他混合模式，我们必须创建一个<font color="green">custom UI shader</font>。我们将简单地复制<font color="red"> *Default-UI* shader</font>，通过<font color="green">_SrcBlend和_DstBlend</font>着色器属性添加对可配置混合的支持。我还调整了着色器的代码，以更好地配合本系列教程的风格。

```glsl
Shader "Custom RP/UI Custom Blending" {
	Properties {
		[PerRendererData] _MainTex ("Sprite Texture", 2D) = "white" {}
		_Color ("Tint", Color) = (1,1,1,1)
		_StencilComp ("Stencil Comparison", Float) = 8
		_Stencil ("Stencil ID", Float) = 0
		_StencilOp ("Stencil Operation", Float) = 0
		_StencilWriteMask ("Stencil Write Mask", Float) = 255
		_StencilReadMask ("Stencil Read Mask", Float) = 255
		_ColorMask ("Color Mask", Float) = 15
		[Toggle(UNITY_UI_ALPHACLIP)] _UseUIAlphaClip ("Use Alpha Clip", Float) = 0
		[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend ("Src Blend", Float) = 1
		[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend ("Dst Blend", Float) = 0
	}

	SubShader {
		Tags {
			"Queue" = "Transparent"
			"IgnoreProjector" = "True"
			"RenderType" = "Transparent"
			"PreviewType" = "Plane"
			"CanUseSpriteAtlas" = "True"
		}

		Stencil {
			Ref [_Stencil]
			Comp [_StencilComp]
			Pass [_StencilOp]
			ReadMask [_StencilReadMask]
			WriteMask [_StencilWriteMask]
		}

		Blend [_SrcBlend] [_DstBlend]
		ColorMask [_ColorMask]
		Cull Off
		ZWrite Off
		ZTest [unity_GUIZTestMode]

		Pass { … }
	}
}
```

这里是通行证，除风格外未作修改。

```glsl
	Pass {
			Name "Default"
			
			CGPROGRAM
			#pragma vertex UIPassVertex
			#pragma fragment UIPassFragment
			#pragma target 2.0

			#include "UnityCG.cginc"
			#include "UnityUI.cginc"

			#pragma multi_compile_local _ UNITY_UI_CLIP_RECT
			#pragma multi_compile_local _ UNITY_UI_ALPHACLIP

			struct Attributes {
				float4 positionOS : POSITION;
				float4 color : COLOR;
				float2 baseUV : TEXCOORD0;
				UNITY_VERTEX_INPUT_INSTANCE_ID
			};

			struct Varyings {
				float4 positionCS : SV_POSITION;
				float2 positionUI : VAR_POSITION;
				float2 baseUV : VAR_BASE_UV;
				float4 color : COLOR;
				UNITY_VERTEX_OUTPUT_STEREO
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			float4 _Color;
			float4 _TextureSampleAdd;
			float4 _ClipRect;

			Varyings UIPassVertex (Attributes input) {
				Varyings output;
				UNITY_SETUP_INSTANCE_ID(input);
				UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(output);
				output.positionCS = UnityObjectToClipPos(input.positionOS);
				output.positionUI = input.positionOS.xy;
				output.baseUV = TRANSFORM_TEX(input.baseUV, _MainTex);
				output.color = input.color * _Color;
				return output;
			}

			float4 UIPassFragment (Varyings input) : SV_Target {
				float4 color =
					(tex2D(_MainTex, input.baseUV) + _TextureSampleAdd) * input.color;
				#if defined(UNITY_UI_CLIP_RECT)
					color.a *= UnityGet2DClipping(input.positionUI, _ClipRect);
				#endif
				#if defined(UNITY_UI_ALPHACLIP)
					clip (color.a - 0.001);
				#endif
				return color;
			}
			ENDCG
		}
```

<img src="E:/Typora%20file/SRP/assets/raw-image-premultiplied.png" alt="img" style="zoom:50%;" />

<center>Raw UI image using custom UI shader with premultiplied alpha blending.</center>

#### 1.7、[Post FX Settings Per Camera](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#1.7)

当使用多个相机时，应该可以在每个相机上使用不同的后期特效，所以让我们增加对它的支持。给一个切换<font color="green">CameraSettings</font>按钮，控制它是否覆盖全局的后期特效设置，以及它自己的<font color="red">PostFXSettings</font>字段。

```c#
	public bool overridePostFX = false;

	public PostFXSettings postFXSettings = default;
```

<img src="E:/Typora%20file/SRP/assets/override-post-fx-settings.png" alt="img" style="zoom:50%;" />

<center>Camera post FX override settings.</center>

让<font color="green">CameraRenderer.Render</font>检查相机是否覆盖了后期的特效设置。如果是，就用相机的设置替换渲染管道提供的设置。

```c#
	var crpCamera = camera.GetComponent<CustomRenderPipelineCamera>();
		CameraSettings cameraSettings =
			crpCamera ? crpCamera.Settings : defaultCameraSettings;

		if (cameraSettings.overridePostFX) {
			postFXSettings = cameraSettings.postFXSettings;
		}
```

现在每个摄像机都可以使用默认的或自定义的后期特效。例如，我让<font color="green">bottom camera</font>使用默认的，关闭了<font color="red">overlay camera</font>的后期特效，并给<font color="green">render texture camera</font>提供了不同的后期特效，包括冷温度转变和中性色调映射。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/combining-cameras/different-post-fx-settings.png" alt="img" style="zoom:50%;" />

<center>Different post FX settings per camera.</center>

### 2、[Rendering Layers]()

当同时显示多个摄像机视图时，我们并不总是希望为所有摄像机渲染相同的场景。例如，我们可以渲染主视图和一个人物肖像。Unity一次只支持一个全局场景，所以我们必须用一种方法来限制每个摄像机看到的东西。

#### 2.1、[Culling Masks](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#2.1)

每个游戏对象都属于一个单独的层。场景窗口可以通过编辑器右上方的层下拉菜单来过滤它所显示的层。同样地，每个摄像机都有一个Culling Mask属性，可以用来限制它以同样的方式显示的内容。这个遮罩是在渲染的剔除步骤中应用的。

每个物体正好属于一个图层，而<font color="green">culling masks</font>可以包括多个图层。例如，你可以有两台摄像机都渲染默认层，而其中一台还渲染忽略射线，而另一台则渲染水。这样一来，有些物体在两台相机上都能看到，而有些物体只对其中一台可见，还有一些物体可能根本就没有被渲染。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/rendering-layers/different-culling-masks.png" alt="img" style="zoom:50%;" />

<center>Split screen with different culling masks per camera.</center>


灯光也有<font color="green"> culling masks</font>。这个想法是，一个被遮蔽的物体在灯光下的表现就像那个灯光不存在一样。该物体不被灯光照亮，也不会为它投下阴影。但是，如果我们用一个定向光来试一下，只有它的阴影会受到影响。

<img src="E:/Typora%20file/SRP/assets/culling-mask-directional-light.png" alt="img" style="zoom:50%;" />

<center>Culling mask applied to directional light only affects shadows.</center>

如果我们尝试用另一种灯光类型，如果我们的RP的 "<font color="green">*Lights Per Object* </font> "选项被禁用，也会发生同样的情况。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/rendering-layers/culling-mask-point-light.png" alt="img" style="zoom:50%;" />

<center>Same culling mask applied to bright point light.</center>

如果启用了 "每个物体使用灯光"，那么灯光剔除就能正常工作，但只适用于点光源和聚光灯。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/rendering-layers/culling-mask-point-light-per-object.png" alt="img" style="zoom:50%;" />

<center>Point light with lights-per-object enabled.</center>

我们得到这些结果是因为Unity在向GPU发送每个物体的光照指数时，会应用光照的<font color="red">culling mask</font>。因此，如果我们不使用这些遮蔽，就无法工作。而且它对方向性的灯光也不起作用，因为我们总是对所有的东西都应用这些。阴影总是被正确地剔除，因为当从灯光的角度渲染阴影投射者时，灯光的<font color="green"> culling mask </font>就像照相机的<font color="red">Mask</font>一样被使用。

在我们目前的方法中，我们不能完全支持灯光的遮挡。这个限制并不是什么大问题，HDRP也不支持灯光的遮蔽。Unity提供了渲染层作为SRP的替代品。使用<font color="red">rendering layers</font>而不是游戏对象层有两个好处。首先，渲染器并不局限于单一的层，这使得它们更加灵活。第二，渲染层不用于其他任何东西，不像默认层那样也用于物理学。

在我们继续讨论<font color="DarkOrchid ">rendering layers</font>之前，让我们在灯光的检查器中显示一个警告，当它的<font color="green"> culling mask</font>被设置为<font color="red">Everything</font>以外的东西时。灯光的遮罩是通过它的<font color="green">cullingMask</font>整数属性提供的，-1代表所有层。

如果<font color="green">CustomLightEditor</font>的目标将其遮罩设置为其他任何东西，在<font color="red">OnInspectorGUI</font>的最后调用<font color="green">EditorGUILayout.HelpBox</font>，用一个字符串表示遮罩只影响阴影，用<font color="red">MessageType.Warning</font>来显示一个警告图标。

```c#
public override void OnInspectorGUI() {
		…

		var light = target as Light;
		if (light.cullingMask != -1) {
			EditorGUILayout.HelpBox(
				"Culling Mask only affects shadows.",
				MessageType.Warning
			);
		}
	}
```

<img src="E:/Typora%20file/SRP/assets/culling-mask-warning.png" alt="img" style="zoom:50%;" />

<center>Culling mask warning for lights.</center>

我们可以说得更具体一点，提到<font color="green">*Lights Per Object* setting </font>对那些没有方向性的灯光有区分。

```c#
		EditorGUILayout.HelpBox(
				light.type == LightType.Directional ?
					"Culling Mask only affects shadows." :
					"Culling Mask only affects shadow unless Use Lights Per Objects is on.",
				MessageType.Warning
			);
```

#### 2.2、[Adjusting the Rendering Layer Mask]()

当使用SRP时，<font color="DarkOrchid ">灯光和MeshRenderer</font>组件的检查器会暴露一个<font color="RoyalBlue">*Rendering Layer Mask*</font>，而当使用默认RP时，该属性是隐藏的。

<img src="E:/Typora%20file/SRP/assets/mesh-renderer-rendering-layer-mask.png" alt="img" style="zoom:50%;" />

<center>Rendering layer mask for `MeshRenderer`.</center>

默认情况下，下拉菜单显示32个图层，命名为Layer1、Layer2，等等。这些层的名称可以通过覆盖<font color="green">RenderPipelineAsset.renderingLayerMaskNames getter</font>属性来配置每个RP。因为这对下拉菜单来说纯粹是外观上的，我们只需要为<font color="red">Unity Editor</font>做这个。所以把<font color="green">CustomRenderPipelineAsset</font>变成一个局部类。

```
public partial class CustomRenderPipelineAsset : RenderPipelineAsset { … }
```

然后为它创建一个<font color="red"> editor-only</font>的脚本资产，重写该属性。它返回一个字符串数组，我们可以在一个静态构造方法中创建它。我们先用与默认值相同的名字，只是在层字和数字之间加一个空格。

```c#
partial class CustomRenderPipelineAsset {

#if UNITY_EDITOR

	static string[] renderingLayerNames;

	static CustomRenderPipelineAsset () {
		renderingLayerNames = new string[32];
		for (int i = 0; i < renderingLayerNames.Length; i++) {
			renderingLayerNames[i] = "Layer " + (i + 1);
		}
	}

	public override string[] renderingLayerMaskNames => renderingLayerNames;

#endif
}
```

这稍微改变了渲染层的标签。对于<font color="red">MeshRenderer</font>组件来说，它的效果很好，但不幸的是，灯光的属性并不响应变化。

渲染层的下拉菜单显示出来了，但调整并没有被应用。我们不能直接解决这个问题，但可以添加我们自己的版本的属性，它确实可以工作。首先在<font color="red">CustomLightEditor</font>中为它创建一个<font color="green">GUIContent</font>，用同样的标签和一个工具提示表明这是上面的属性的功能版本。

```c#
	static GUIContent renderingLayerMaskLabel =
		new GUIContent("Rendering Layer Mask", "Functional version of above property.");
```

然后创建一个<font color="green">DrawRenderingLayerMask</font>方法，它是<font color="red">LightEditor.DrawRenderingLayerMask</font>的替代方法，确实将改变的值分配回属性。为了使下拉菜单使用RP的层名，我们不能简单地依赖<font color="green">EditorGUILayout.PropertyField</font>。我们必须从设置中抓取相关的属性，确保处理多选的混合值，抓取<font color="RoyalBlue">mask </font>作为一个整数，显示它，并将改变的值赋回给属性。这是在默认的<font color="DarkOrchid ">light inspector</font>的版本中缺少的最后一步。

显示下拉菜单是通过调用<font color="green">EditorGUILayout.MaskField</font>来完成的，参数为标签、遮罩和<font color="red">GraphicsSettings.currentRenderPipeline.renderingLayerMaskNames</font>。

```c#
void DrawRenderingLayerMask () {
		SerializedProperty property = settings.renderingLayerMask;
		EditorGUI.showMixedValue = property.hasMultipleDifferentValues;
		EditorGUI.BeginChangeCheck();
		int mask = property.intValue;
		mask = EditorGUILayout.MaskField(
			renderingLayerMaskLabel, mask,
			GraphicsSettings.currentRenderPipeline.renderingLayerMaskNames
		);
		if (EditorGUI.EndChangeCheck()) {
			property.intValue = mask;
		}
		EditorGUI.showMixedValue = false;
	}
```

在调用base.<font color="green">OnInspectorGUI</font>后直接调用新方法，所以额外的渲染层遮罩属性会直接显示在无功能的属性下面。另外，我们现在必须始终调用<font color="red">ApplyModifiedProperties</font>，以确保渲染层遮罩的变化被应用到灯光上。

```c#
	public override void OnInspectorGUI() {
		base.OnInspectorGUI();
		DrawRenderingLayerMask();
		
		if (
			!settings.lightType.hasMultipleDifferentValues &&
			(LightType)settings.lightType.enumValueIndex == LightType.Spot
		)
		{
			settings.DrawInnerAndOuterSpotAngle();
			//settings.ApplyModifiedProperties();
		}

		settings.ApplyModifiedProperties();

		…
	}
```

<img src="E:/Typora%20file/SRP/assets/rendering-layer-mask-light.png" alt="img" style="zoom:50%;" />

<center>Extra rendering layer mask property for light.</center>

我们版本的属性确实应用了变化，只是选择Everything或Layer 32选项产生的结果与选择Nothing相同。这是因为灯光的渲染层遮罩在内部被存储为无符号整数，即<font color="green">uint</font>。这是有道理的，因为它是作为一个<font color="DarkOrchid ">bit mask</font>使用的，但是<font color="red">SerializedProperty</font>只支持获取和设置有<font color="red">符号的整数值</font>。

<font color="green">Everything</font>选项用-1表示，该属性将其夹紧为零。而第32层对应的是最高位，表示比<font color="green">int.MaxValue大1</font>的数字，该属性也将其替换为0。

我们可以通过简单地移除最后一层来解决第二个问题，即把渲染层名称的数量减少到31个。这还是有很多层的。HDRP只支持8个。

```c#
renderingLayerNames = new string[31];
```

通过移除一层，<font color="green">Everything</font>选项现在被表示为一个除了最高位以外都被设置的值，这与<font color="red">int.MaxValue</font>相匹配。因此，我们可以通过显示-1而存储int.MaxValue来解决第一个问题。默认的属性不会这样做，这就是为什么它在适当的时候显示混合...而不是万物。HDRP也有这样的问题。

```c#
	int mask = property.intValue;
		if (mask == int.MaxValue) {
			mask = -1;
		}
		mask = EditorGUILayout.MaskField(
			renderingLayerMaskLabel, mask,
			GraphicsSettings.currentRenderPipeline.renderingLayerMaskNames
		);
		if (EditorGUI.EndChangeCheck()) {
			property.intValue = mask == -1 ? int.MaxValue : mask;
		}
```

<img src="E:/Typora%20file/SRP/assets/functional-rendering-layer-mask-property.png" alt="img" style="zoom:50%;" />

<center>Functional rendering layer mask property</center>

我们终于可以正确调整灯光的<font color="green">rendering layers mask</font>了。但是默认情况下并没有使用遮罩，所以没有什么变化。我们可以通过启用<font color="red">Shadows</font>中<font color="green">ShadowDrawingSettings</font>的<font color="red">useRenderingLayerMaskTest</font>来将其应用于阴影。对所有的灯光都这样做，所以在<font color="DarkOrchid ">RenderDirectionalShadows、RenderSpotShadows和RenderPointShadows</font>。现在我们可以通过配置物体和灯光的渲染层掩码来消除阴影。

```c#
	var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex) {
				useRenderingLayerMaskTest = true
			};
```

#### 2.3、[Sending a Mask to the GPU](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#2.3)

为了将渲染层的遮罩应用于<font color="green">Lit shader</font>的照明计算，物体和灯光的遮罩必须在GPU端可用。为了访问对象的遮罩，我们必须在<font color="green">UnityInput</font>的<font color="DarkOrchid ">UnityPerDraw</font>结构中添加一个float4的<font color="RoyalBlue">unity_RenderingLayer</font>字段，直接在<font color="red">unity_WorldTransformParams</font>下面。遮罩被存储在其<font color="DarkOrchid ">第一个组件x</font>中。

```glsl
	real4 unity_WorldTransformParams;

	float4 unity_RenderingLayer;
```

我们将把掩码添加到我们的<font color="red">Surface</font>结构中，因为它是一个<font color="green">bit mask.</font>，所以是uint。

```glsl
struct Surface {
	…
	uint renderingLayerMask;
};
```

在<font color="red">LitPassFragment</font>中<font color="RoyalBlue">surface's mask</font>时，我们必须使用<font color="green">asuint</font>的内在函数。这可以使用原始数据，而不需要进行数字类型的转换，即从<font color="red">float到uint</font>，这将改变<font color="RoyalBlue">bit pattern.</font>

```glsl
surface.dither = InterleavedGradientNoise(input.positionCS.xy, 0);
	surface.renderingLayerMask = asuint(unity_RenderingLayer.x);
```

我们必须对<font color="green">Light</font>结构做同样的处理，所以也要给它的<font color="red">rendering layer mask </font>一个<font color="RoyalBlue">uint</font>字段。

```c#
struct Light {
	…
	uint renderingLayerMask;
};
```

我们要负责将遮罩发送到GPU上。让我们通过将它存储在<font color="red">_DirectionalLightDirections和_OtherLightDirections</font>数组中未使用的第四个组件中来实现。为了清楚起见，在它们的名字后面加上<font color="green">AndMasks</font>的后缀。

```glsl
CBUFFER_START(_CustomLight)
	…
	float4 _DirectionalLightDirectionsAndMasks[MAX_DIRECTIONAL_LIGHT_COUNT];
	…
	float4 _OtherLightDirectionsAndMasks[MAX_OTHER_LIGHT_COUNT];
	…
CBUFFER_END
```

复制<font color="green">GetDirectionalLight</font>中的Mask。

```c#
	light.direction = _DirectionalLightDirectionsAndMasks[index].xyz;
	light.renderingLayerMask = asuint(_DirectionalLightDirectionsAndMasks[index].w);
```

And in <font color="red">GetOtherLight</font>.

```c#
	float3 spotDirection = _OtherLightDirectionsAndMasks[index].xyz;
	light.renderingLayerMask = asuint(_OtherLightDirectionsAndMasks[index].w);
```

在<font color="green">CPU</font>方面，调整我们<font color="red">Lighting</font>类中的<font color="RoyalBlue"> identifier and array names  - 标识符和数组名称</font>，使之相匹配。然后还要复制<font color="green">rendering layer mask </font>。我们从<font color="red">SetupDirectionalLight</font>开始，现在它也需要直接访问灯光对象。让我们把它作为一个参数加入。

```c#
	void SetupDirectionalLight (
		int index, int visibleIndex, ref VisibleLight visibleLight, Light light
	) {
		dirLightColors[index] = visibleLight.finalColor;
		Vector4 dirAndMask = -visibleLight.localToWorldMatrix.GetColumn(2);
		dirAndMask.w = light.renderingLayerMask;
		dirLightDirectionsAndMasks[index] = dirAndMask;
		dirLightShadowData[index] =
			shadows.ReserveDirectionalShadows(light, visibleIndex);
	}
```

对<font color="red">SetupSpotLight</font>做同样的修改，同时增加一个<font color="red">Light</font>参数以保持一致。

```c#
void SetupSpotLight (
		int index, int visibleIndex, ref VisibleLight visibleLight, Light light
	) {
		…
		Vector4 dirAndMask = -visibleLight.localToWorldMatrix.GetColumn(2);
		dirAndMask.w = light.renderingLayerMask;
		otherLightDirectionsAndMasks[index] = dirAndMask;

		//Light light = visibleLight.light;
		…
		}
```

然后为<font color="green">SetupPointLight</font>做这个，现在它也要改变其他<font color="red">LightDirectionsAndMasks</font>。因为它不使用方向，它可以被设置为零。

```c#
	void SetupPointLight (
		int index, int visibleIndex, ref VisibleLight visibleLight, Light light
	) {
		…
		Vector4 dirAndmask = Vector4.zero;
		dirAndmask.w = light.renderingLayerMask;
		otherLightDirectionsAndMasks[index] = dirAndmask;
		//Light light = visibleLight.light;
		otherLightShadowData[index] =
			shadows.ReserveOtherShadows(light, visibleIndex);
	}
```

现在我们必须在<font color="green">SetupLights</font>中抓取一次<font color="DarkOrchid ">Light</font>对象，并将其传递给所有的设置方法。我们还将在这里对灯光做一些其他的事情。

```c#
VisibleLight visibleLight = visibleLights[i];
			Light light = visibleLight.light;
			switch (visibleLight.lightType) {
				case LightType.Directional:
					if (dirLightCount < maxDirLightCount) {
						SetupDirectionalLight(
							dirLightCount++, i, ref visibleLight, light
						);
					}
					break;
				case LightType.Point:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupPointLight(otherLightCount++, i, ref visibleLight, light);
					}
					break;
				case LightType.Spot:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupSpotLight(otherLightCount++, i, ref visibleLight, light);
					}
					break;
			}
```

回到GPU方面，为<font color="DarkOrchid ">Lighting</font>增加一个<font color="green">RenderingLayersOverlap</font>函数，返回<font color="green">masks of a surface and light </font>是否<font color="RoyalBlue"> overlap</font>。这是通过检查<font color="green"> bit masks</font>的<font color="green">bitwise-AND</font>是否为非零来实现的。

按位AND运算符(&)将第一个操作数的每一位与第二个操作数的对应位进行比较。如果两个比特都为1，则结果位设置为1。否则，对应的结果位设置为0。位AND操作符的两个操作数都必须为整型

```c#
bool RenderingLayersOverlap (Surface surface, Light light) {
	return (surface.renderingLayerMask & light.renderingLayerMask) != 0;
}
```

现在我们可以用这个方法来检查<font color="green">GetLighting</font>的三个循环里面是否需要添加照明。

```c#
for (int i = 0; i < GetDirectionalLightCount(); i++) {
		Light light = GetDirectionalLight(i, surfaceWS, shadowData);
		if (RenderingLayersOverlap(surfaceWS, light)) {
			color += GetLighting(surfaceWS, brdf, light);
		}
	}
	
	#if defined(_LIGHTS_PER_OBJECT)
		for (int j = 0; j < min(unity_LightData.y, 8); j++) {
			int lightIndex = unity_LightIndices[j / 4][j % 4];
			Light light = GetOtherLight(lightIndex, surfaceWS, shadowData);
			if (RenderingLayersOverlap(surfaceWS, light)) {
				color += GetLighting(surfaceWS, brdf, light);
			}
		}
	#else
		for (int j = 0; j < GetOtherLightCount(); j++) {
			Light light = GetOtherLight(j, surfaceWS, shadowData);
			if (RenderingLayersOverlap(surfaceWS, light)) {
				color += GetLighting(surfaceWS, brdf, light);
			}
		}
	#endif
```

#### 2.4、[Reinterpreting an Int as a Float](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#2.4)

尽管<font color="green">rendering masks</font>在此时会影响光照，但它并没有正确地这样做。<font color="red">Light.renderingLayerMask</font>属性将其位掩码暴露为一个<font color="green">int</font>，在<font color="red">light setup</font>方法中转换为<font color="green">float</font>时，它被搅乱了。

没有办法直接向GPU发送一个整数数组，所以我们必须以某种方式将int重新解释为float，而不进行转换，但C#中没有直接对应的<font color="green">asuint</font>。

我们不能像<font color="red">HLSL</font>那样简单地重新解释C#中的数据，因为C#是强类型的。我们能做的是通过使用跟<font color="green">union</font>结构来别名数据类型。我们将通过为<font color="red">int</font>添加一个<font color="green">ReinterpretAsFloat</font>扩展方法来隐藏这种方法。为这个方法创建一个静态的<font color="green">ReinterpretExtensions</font>类，它最初只是执行一个普通的类型转换。

```c#
public static class ReinterpretExtensions {

	public static float ReinterpretAsFloat (this int value) {
		return value;
	}
}
```

在三个灯光设置方法中使用<font color="green">ReinterpretAsFloat</font>，而不是依赖隐含的转换。

```c#
dirAndMask.w = light.renderingLayerMask.ReinterpretAsFloat();
```

然后在<font color="green">ReinterpretExtensions</font>中定义一个结构类型，它有一个int和一个float字段。在<font color="green">ReinterpretAsFloat</font>中初始化这个类型的默认变量，设置其整数值，然后返回其浮点值。

```c#
	struct IntFloat {

		public int intValue;

		public float floatValue;
	}

	public static float ReinterpretAsFloat (this int value) {
		IntFloat converter = default;
		converter.intValue = value;
		return converter.floatValue;
	}
}
```

为了将其转化为重新解释，我们必须使结构的两个字段重叠，以便它们共享<font color="green">same data</font>。

这是有可能的，因为两种类型的大小都是四字节。我们通过将<font color="red">StructLayout</font>属性附加到类型上，设置为<font color="green">LayoutKind.Explicit</font>，使结构的布局显性化。然后，我们必须为其字段添加<font color="red">FieldOffset</font>属性，以指示字段的数据应该放在哪里。将两个偏移量都设置为零，这样它们就会重叠。这些属性来自<font color="red">System.Runtime.InteropServices</font>命名空间。

```c#
using System.Runtime.InteropServices;

public static class ReinterpretExtensions {

	[StructLayout(LayoutKind.Explicit)]
	struct IntFloat {

		[FieldOffset(0)]
		public int intValue;

		[FieldOffset(0)]
		public float floatValue;
	}

	…
}
```

现在，结构的int和float字段代表相同的数据，但解释方式不同。这使得位掩码保持不变，渲染层掩码现在可以正常工作。

<img src="E:/Typora%20file/SRP/assets/directional-renderling-layer-mask.png" alt="img" style="zoom:50%;" />

<center>Directional light ignores half the objects.</center>

#### 2.5、Camera Rendering Layer Mask

我们还可以使用<font color="green">rendering masks </font>来限制相机的渲染内容，除了使用他们现有的剔除蒙版之外。相机没有渲染层遮罩属性，但我们可以将其添加到<font color="red">CameraSettings</font>中。我们将使它成为一个<font color="green">int</font>，因为<font color="green">light's mask</font>也是以<font color="green">int</font>的形式暴露的。默认情况下将其设置为-1，代表所有层。

```c#
public int renderingLayerMask = -1;
```

<img src="E:/Typora%20file/SRP/assets/camera-rendering-layer-mask-int.png" alt="img" style="zoom:50%;" />

<center>Camera rendering layer mask, exposed as integer.</center>

为了将遮罩暴露为一个下拉菜单，我们必须为它创建一个自定义GUI。但与其为整个<font color="green">CameraSettings</font>类创建一个自定义编辑器，不如只为<font color="green">rendering layer masks</font>创建一个。

首先，为了表明一个字段代表一个<font color="red">rendering layer mask</font>，创建一个<font color="RoyalBlue">RenderingLayerMaskFieldAttribute</font>类，它扩展了<font color="red">PropertyAttribute</font>。这只是一个标记属性，不需要做其他任何事情。注意，这不是一个编辑器类型，所以不应该放在<font color="green">*Editor*</font>文件夹中。

```c#
using UnityEngine;

public class RenderingLayerMaskFieldAttribute : PropertyAttribute {}
```

把这个属性附加到我们的<font color="green">rendering layer mask</font>字段。

```c#
[RenderingLayerMaskField]
	public int renderingLayerMask = -1;
```

现在创建一个自定义属性抽屉编辑器类，它扩展了<font color="green">PropertyDrawer</font>，为我们的属性类型添加了<font color="red">CustomPropertyDrawer</font>属性。把<font color="green">CustomLightEditor.DrawRenderingLayerMask</font>复制到其中，重命名为<font color="red">Draw</font>，并使其成为公共静态。

然后给它三个参数：<font color="green">a position Rect, a serialized property, and a GUIContent label 一个位置<font color="RoyalBlue"> Rect</font>，一个<font color="RoyalBlue">serialized</font> 的属性，和一个<font color="RoyalBlue">GUIContent</font>标签</font>。使用这些来调用<font color="DarkOrchid ">EditorGUI.MaskField</font>而不是<font color="green">EditorGUILayout.MaskField</font>。

```c#
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;

[CustomPropertyDrawer(typeof(RenderingLayerMaskFieldAttribute))]
public class RenderingLayerMaskDrawer : PropertyDrawer {

	public static void Draw (
		Rect position, SerializedProperty property, GUIContent label
	) {
		//SerializedProperty property = settings.renderingLayerMask;
		EditorGUI.showMixedValue = property.hasMultipleDifferentValues;
		EditorGUI.BeginChangeCheck();
		int mask = property.intValue;
		if (mask == int.MaxValue) {
			mask = -1;
		}
		mask = EditorGUI.MaskField(
			position, label, mask,
			GraphicsSettings.currentRenderPipeline.renderingLayerMaskNames
		);
		if (EditorGUI.EndChangeCheck()) {
			property.intValue = mask == -1 ? int.MaxValue : mask;
		}
		EditorGUI.showMixedValue = false;
	}
}
```

只有当属性的基础类型是<font color="green">uint</font>时，我们才需要单独处理-1。如果它的类型属性等于 <font color="red">"uint"</font>，就属于这种情况。

```c#
		int mask = property.intValue;
		bool isUint = property.type == "uint";
		if (isUint && mask == int.MaxValue) {
			mask = -1;
		}
		…
		if (EditorGUI.EndChangeCheck()) {
			property.intValue = isUint && mask == -1 ? int.MaxValue : mask;
		}
```

然后重写<font color="green">OnGUI</font>方法，简单地将其调用转发给<font color="green">Draw</font>。

```c#
	public override void OnGUI (
		Rect position, SerializedProperty property, GUIContent label
	) {
		Draw(position, property, label);
	}
```

<img src="E:/Typora%20file/SRP/assets/camera-rendering-layer-mask.png" alt="img" style="zoom:50%;" />

<center>Rendering layer mask dropdown menu.</center>

为了使<font color="green">Draw</font>更容易使用，添加一个没有<font color="green">Rect</font>参数的版本。调用<font color="red">EditorGUILayout.GetControlRect</font>，从<font color="red"> layout engine</font>获得一个单行位置矩形。

```c#
	public static void Draw (SerializedProperty property, GUIContent label) {
		Draw(EditorGUILayout.GetControlRect(), property, label);
	}
```

现在我们可以从<font color="green">CustomLightEditor</font>中移除<font color="red">DrawRenderingLayerMask</font>方法，并调用<font color="green">RenderingLayerMaskDrawer</font>.Draw来代替。

```c#
	public override void OnInspectorGUI() {
		base.OnInspectorGUI();
		//DrawRenderingLayerMask();
		RenderingLayerMaskDrawer.Draw(
			settings.renderingLayerMask, renderingLayerMaskLabel
		);
		
		…
	}

	//void DrawRenderingLayerMask () { … }
```

为了应用<font color="DarkOrchid "> camera's rendering layer mask </font>，在<font color="green">CameraRenderer.DrawVisibleGeometry</font>中为其添加一个参数，并将其作为一个名为<font color="green">renderingLayerMask</font>的参数传递给<font color="red">FilteringSettings</font>构造方法，并铸为一个<font color="red">uint</font>。

[FilteringSettings](https://docs.unity3d.com/ScriptReference/Rendering.FilteringSettings.html)

```c#
	void DrawVisibleGeometry (
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		int renderingLayerMask
	) {
		…

		var filteringSettings = new FilteringSettings(
			RenderQueueRange.opaque, renderingLayerMask: (uint)renderingLayerMask
		);

		…
	}
```

然后在<font color="green">Render</font>中调用<font color="red">DrawVisibleGeometry</font>时，将渲染层的遮罩传递给它。

```c#
	DrawVisibleGeometry(
			useDynamicBatching, useGPUInstancing, useLightsPerObject,
			cameraSettings.renderingLayerMask
		);
```

现在可以使用更灵活的渲染层遮罩来控制摄像机渲染的内容。例如，我们可以让一些物体投射阴影，即使摄像机没有看到它们，而不需要专门的只投射阴影的物体。

<img src="E:/Typora%20file/SRP/assets/rendering-objects-not-affected-by-light.png" alt="img" style="zoom:50%;" />

<center>Only rendering objects not affected by the light, plus the ground.</center>

有一点需要注意的是，只有<font color="green">culling mask</font>是用于剔除的，所以如果你排除了很多对象，<font color="red">regular culling mask </font>会表现得更好。

#### 2.6、[Masking Lights Per Camera](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/#2.6)

虽然Unity的RP没有这样做，但是除了几何图形之外，也可以在每个摄像机上遮挡灯光。我们将再次使用渲染层来实现这一点，但由于这是一个非标准行为，让我们通过在<font color="green">CameraSettings</font>中添加一个切换按钮来使其成为可选项。

```c#
	public bool maskLights = false;
```

<img src="E:/Typora%20file/SRP/assets/mask-lights.png" alt="img" style="zoom:50%;" />

<center>Camera set to mask lights.</center>

我们需要做的就是在<font color="green">Lighting.SetupLights</font>中跳过<font color="green"> masked lights</font>。在这个方法中添加一个渲染层遮罩参数，然后检查<font color="red"> light's rendering layer mask</font>是否与提供的遮罩重叠。如果是的话，就进行开关语句来设置灯光，否则就跳过它。

```c#
	void SetupLights (bool useLightsPerObject, int renderingLayerMask) {
		…
		for (i = 0; i < visibleLights.Length; i++) {
			int newIndex = -1;
			VisibleLight visibleLight = visibleLights[i];
			Light light = visibleLight.light;
			if ((light.renderingLayerMask & renderingLayerMask) != 0) {
				switch (visibleLight.lightType) {
					…
				}
			}
			if (useLightsPerObject) {
				indexMap[i] = newIndex;
			}
		}
		
		…
	}
```

<font color="red">Lighting.Setup</font>必须将渲染图层的遮罩传递过来。

```c#
public void Setup (
		ScriptableRenderContext context, CullingResults cullingResults,
		ShadowSettings shadowSettings, bool useLightsPerObject, int renderingLayerMask
	) {
		…
		SetupLights(useLightsPerObject, renderingLayerMask);
		…
	}
```

而我们必须在<font color="green">CameraRenderer.Render</font>中提供相机的遮罩，但只有在它适用于灯光的情况下，否则使用-1。

```c#
	lighting.Setup(
			context, cullingResults, shadowSettings, useLightsPerObject,
			cameraSettings.maskLights ? cameraSettings.renderingLayerMask : -1
		);
```

现在我们可以做一些事情，比如让两个摄像机渲染同一个场景，但使用不同的灯光，而不需要在两者之间调整灯光。这也使得我们可以很容易地渲染一个单独的场景，比如在世界原点的人物肖像，而不用让主场景的灯光影响它。注意，这只适用于实时照明，完全烘烤的灯光和混合灯光的烘烤间接贡献不能被掩盖。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/multiple-cameras/rendering-layers/two-camera-different-lighting.png)

<center>Two cameras seeing the same scene in a different light.</center>