# 03_Directional Lights



- Use normal vectors to calculate lighting.
- Support up to four directional lights.
- Apply a BRDF.
- Make lit transparent materials.
- Create a custom shader GUI with presets.
- ![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/tutorial-image.jpg)

### 1、Lighting

如果我们想创建一个更真实的场景，那么我们就必须模拟光线与表面的互动。这就需要一个比我们目前拥有的无光的着色器更复杂的着色器。

#### 1.1、Lit Shader

复制<font color="green">UnlitPass.HLSL</font>文件，并将其重命名为<font color="blue">LitPass</font>。调整<font color="green">include guard</font>的定义以及顶点和片段的函数名称，使之相匹配。我们稍后将添加照明计算。

```glsl
#ifndef CUSTOM_LIT_PASS_INCLUDED
#define CUSTOM_LIT_PASS_INCLUDED

…

Varyings LitPassVertex (Attributes input) { … }

float4 LitPassFragment (Varyings input) : SV_TARGET { … }

#endif
```

同时复制<font color="green">Unlit shader</font>并将其重命名为Lit。改变它的菜单名称、它所包含的文件以及它所使用的函数。让我们也把默认颜色改为灰色，因为在一个光线充足的场景中，全白的表面会显得非常明亮。通用管线默认也使用灰色。

```glsl
Shader "Custom RP/Lit" {
	
	Properties {
		_BaseMap("Texture", 2D) = "white" {}
		_BaseColor("Color", Color) = (0.5, 0.5, 0.5, 1.0)
		…
	}

	SubShader {
		Pass {
			…
			#pragma vertex LitPassVertex
			#pragma fragment LitPassFragment
			#include "LitPass.hlsl"
			ENDHLSL
		}
	}
}
```

我们将使用一个自定义的照明方法<font color="red">（ custom lighting ）</font>，我们将通过设置我们的<font color="green">shader</font>的<font color="RoyalBlue">light mode</font>为<font color="RoyalBlue">CustomLit</font>来表示。在<font color="green">Pass</font>中添加一个<font color="blue">Tags</font>块，包含 <font color="red">"LightMode"="CustomLit"。</font>

```c#
	Pass {
			Tags {
				"LightMode" = "CustomLit"
			}

			…
		}
```

<font color="red">为了渲染使用该Pass的对象，我们必须将其包含在CameraRenderer中。首先为它添加一个Shader Tag。</font>

```glsl
	static ShaderTagId
		unlitShaderTagId = new ShaderTagId("SRPDefaultUnlit"),
		litShaderTagId = new ShaderTagId("CustomLit");
```

然后像我们在<font color="green">DrawUnsupportedShaders</font>中所做的那样，将其添加到<font color="red">DrawVisibleGeometry</font>中要渲染的通道中。

```c#
var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		) {
			enableDynamicBatching = useDynamicBatching,
			enableInstancing = useGPUInstancing
		};
	
		drawingSettings.SetShaderPassName(1, litShaderTagId);
```

>   [DrawingSettings](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings.html) DrawingSettings 描述了如何对可见对象进行排序 ( [sortingSettings](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings-sortingSettings.html) ) 以及使用哪个着色器传递 ( [shaderPassName](https://docs.unity3d.com/ScriptReference/Rendering.DrawingSettings-shaderPassName.html) )。
>
>   [SetShaderPassName](https://docs.unity3d.com/ScriptReference/30_search.html?q=SetShaderPassName) 设置此绘制调用可以渲染的shader passes 

现在我们可以创建一个新的 <font color="RoyalBlue">opaque</font>材质，尽管此时它会产生与unlit材质相同的结果。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/opaque-material.png" alt="img" style="zoom:50%;" />

​																默认不透明材质。

#### [1.2、Normal Vectors](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#1.2)

一个物体被照亮的程度取决于多种因素，包括<font color="green">light and surface</font>之间的相对角度。要知道表面的方向，我们需要访问表面法线，这是一个单位长度的矢量。

这个向量是顶点数据的一部分，就像 <font color="green">position</font>一样定义在 <font color="green">object space </font>。所以在**LitPass**的<font color="blue">Attributes</font>中添加它。

```C#
struct Attributes {
	float3 positionOS : POSITION;
	float3 normalOS : NORMAL;
	float2 baseUV : TEXCOORD0;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

光照是按<font color="RoyalBlue">片断fragment</font>计算的，所以我们必须把法线向量也加到**Varyings**中。我们将在世界空间中进行计算，所以将其命名为<font color="RoyalBlue">normalWS。</font>

```C#
struct Varyings {
	float4 positionCS : SV_POSITION;
	float3 normalWS : VAR_NORMAL;
	float2 baseUV : VAR_BASE_UV;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

我们可以使用<font color="green">SpaceTransforms</font>中的<font color="green">TransformObjectToWorldNormal</font>来将<font color="blue">LitPassVertex</font>中的法线转换成<font color="DarkOrchid "> world space</font>。

```c#
	output.positionWS = TransformObjectToWorld(input.positionOS);
	output.positionCS = TransformWorldToHClip(positionWS);
	output.normalWS = TransformObjectToWorldNormal(input.normalOS);
```

为了验证我们是否获得了正确的法线向量，<font color="blue">LitPassFragment</font>我们可以将其用作颜色。

```glsl
	base.rgb = input.normalWS;
	return base;
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/world-normals.png)

<center>World-space normal vectors.</center>

负值无法可视化，因此它们被固定为零。

#### [1.3、Interpolated Normals](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#1.3)

尽管<font color="green"> normal vectors </font>在顶点程序中是单位长度的，但跨三角形的线性插值会影响其长度。我们可以通过渲染1和向量的长度之差，放大10倍以使其更加明显，来直观地看到这个误差。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/interpolated-normal-error.png" alt="img" style="zoom:50%;" />

​											*Interpolated normal error, exaggerated.*

我们可以<font color="red">通过对<font color="blue">LitPassFragment</font>中的法线向量进行归一化来平滑插值的失真</font>。当只看法线向量时，这种差别其实并不明显，但用于照明时就比较明显。

```glsl
	base.rgb = normalize(input.normalWS);
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/normalization-after-interpolation.png)

​													Normalization after interpolation.

#### [1.4、Surface Properties](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#1.4)

<font color="green">Shader</font>中的照明是为了模拟<font color="green">light与surface</font>的相互作用，这意味着我们必须跟踪<font color="green">表面的属性surface's properties</font>。现在我们有一个法线矢量和一个基础颜色。

我们可以将后者一分为二：RGB颜色和alpha值。

我们将在一些地方使用这些数据，所以让我们定义一个方便的<font color="green">Surface</font>结构来包含所有相关数据。

把它放在<font color="green">**ShaderLibrary**</font>文件夹中一个单独的<font color="green">**Surface.HLSL**</font>文件中。

```glsl
#ifndef CUSTOM_SURFACE_INCLUDED
#define CUSTOM_SURFACE_INCLUDED

struct Surface {
	float3 normal;
	float3 color;
	float alpha;
};

#endif
```

>   我们不应该把正常定义为normalWS吗?
>   我们可以，但是曲面并不关心法线是在什么空间中定义的。光照计算可以在任何合适的3D空间中进行。所以我们让空间没有定义。当填充数据时，我们只需要在所有地方使用相同的空间。我们将使用世界空间，但稍后我们可以切换到另一个空间，所有内容仍然相同。

<font color="blue">**Include it in   LitPass**</font>中，在<font color="green">Common</font>之后。这样我们就可以保持<font color="green">LitPass</font>的简短。从现在开始，我们将把专门的代码放在自己的HLSL文件中，以便更容易找到相关功能。

```glsl
#include "../ShaderLibrary/Common.hlsl"
#include "../ShaderLibrary/Surface.hlsl"
```

定义一个<font color="green">surface</font>变量在<font color="blue">LitPassFragment</font>中并填满它。然后最终结果变成表面的颜色和 alpha

```glsl
Surface surface;
	surface.normal = normalize(input.normalWS);
	surface.color = base.rgb;
	surface.alpha = base.a;

	return float4(surface.color, surface.alpha);
```

#### [1.5、Calculating Lighting](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#1.5)

为了计算实际光照，我们将创建一个<font color="DarkOrchid ">GetLighting</font>具有<font color="green">Surface</font>参数的函数。最初让它返回表面法线的 Y 分量。

由于这是照明功能，我们将把它放在一个单独的<font color="green">***Lighting.HLSL** </font>文件。

```c#
#ifndef CUSTOM_LIGHTING_INCLUDED
#define CUSTOM_LIGHTING_INCLUDED

float3 GetLighting (Surface surface) {
	return surface.normal.y;
}

#endif
```

把它<font color="green">include</font>在<font color="RoyalBlue">LitPass</font>中，在取得<font color="DarkOrchid ">Surface</font>之后让<font color="RoyalBlue">Lighting</font>依赖于它。

```c#
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```

现在我们可以获取光照<font color="red">**LitPassFragment**</font>并将其用于<font color="RoyalBlue">fragment</font>的 RGB 部分。

```c#
	float3 color = GetLighting(surface);
	return float4(color, surface.alpha);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/diffuse-lighting-from-above.png" alt="img" style="zoom:50%;" />

<center>	Diffuse lighting from above.</center>

在这一点上，结果是<font color="blue">surface normal</font>的Y分量，所以它在球体的顶部是1，在球体的侧面下降到0。在这之下，结果变成了负数，在底部达到-1，但我们看不到负值。

它与法线和向上矢量之间的角度的cos相匹配。忽略负的部分，这在视觉上与直射向下的定向光的漫射照明相匹配。

最后的润色是将表面颜色纳入<font color="green">GetLighting</font>的结果中，将其解释为<font color="DarkOrchid ">surface albedo</font>（表面反照率）。

```glsl
float3 GetLighting (Surface surface) {
	return surface.normal.y * surface.color;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lighting/albedo.png" alt="img" style="zoom:50%;" />

<center>Albedo applied.</center>

>   <font color="red">albedo </font>反照率是什么意思?
>   反照率在拉丁语中是白色的意思。这是一种测量有多少光被表面漫反射的方法。如果反照率不是完全白色，那么部分光能就会被吸收而不是反射。

### [2、Lights](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2)

为了进行适当的照明，我们还需要了解灯光的属性。在本教程中，我们将只限于方向性灯光。一个方向性的光代表一个很远的光源，它的位置并不重要，重要的是它的方向。这是一种简化，但它足以模拟地球上的太阳光和其他传入光线或多或少是单向的情况。

#### [2.1、Light Structure](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.1)

我们将使用一个<font color="green">struct</font>来存储灯光数据。现在我们只需要一个颜色和一个方向就够了。把它放在一个单独的<font color="green">Light HLSL</font>文件中。

同时定义一个<font color="red">**GetDirectionalLight**</font>函数，返回一个配置好的方向性灯光。最初使用一个白色的颜色和向上的矢量，与我们目前使用的灯光数据相匹配。

注意，<font color="red">灯光的方向因此被定义为灯光的来源方向，而不是它的去向</font>。

```c#
#ifndef CUSTOM_LIGHT_INCLUDED
#define CUSTOM_LIGHT_INCLUDED

struct Light {
	float3 color;
	float3 direction;
};

Light GetDirectionalLight () {3
	Light light;
	light.color = 1.0;
	light.direction = float3(0.0, 1.0, 0.0);
	return light;
}

#endif
```

Include the file in ***LitPass*** before *Lighting*.

```c#
#include "../ShaderLibrary/Light.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```

#### [2.2、Lighting Functions](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.2)

在<font color="green">**Lighting**</font>中添加一个<font color="red">**IncomingLight**</font>函数，计算在一个给定的表面和光线下有多少入射光。

对于一个任意的光线方向，我们必须取<font color="red">表面法线<font color="RoyalBlue">(Surface Normal)</font>和方向<font color="RoyalBlue">(Light.Direction)</font>的点积</font>。我们可以使用dot函数来实现。结果应该是由光的颜色来调制的。

```c#
float3 IncomingLight (Surface surface, Light light) {
	return dot(surface.normal, light.direction) * light.color;
}
```

但这只有在表面朝向光线的情况下才是正确的。当点积为负数时，我们必须将其钳制为零，这一点我们可以通过 <font color="green">saturate </font>函数来实现。

```c#
float3 IncomingLight (Surface surface, Light light) {
	return saturate(dot(surface.normal, light.direction)) * light.color;
}
```

添加另一个<font color="red">**GetLighting**</font>函数，它返回<font color="red">**表面<font color="RoyalBlue">(Surface)</font>和光<font color="RoyalBlue">(Light)</font>的最终照明**</font>。现在它是入射光线乘以表面颜色。把它定义在另一个函数的上面。

```c#
float3 GetLighting (Surface surface, Light light) {
	return IncomingLight(surface, light) * surface.color;
}
```

最后，调整只有一个表面参数的<font color="red">**GetLighting**</font>函数，使其调用另一个，使用<font color="red">**GetDirectionalLight**</font>来提供<font color="green">**光线数据**。</font>

```c#
float3 GetLighting (Surface surface) {
	return GetLighting(surface, GetDirectionalLight());
}
```

#### [2.3、Sending Light Data to the GPU](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.3)

我们不应该总是使用来自上方的白光，而应该使用当前场景的光线。默认场景有一个代表太阳的定向光，颜色略微偏黄--FFF4D6十六进制，并且围绕X轴旋转50°，围绕Y轴旋转-30°。如果这样的灯不存在，就创建一个。

为了使灯光的数据能够在着色器中被访问，我们必须为它创建统一的值，就像着色器的属性一样。在这种情况下，我们将定义两个float3向量。

<font color="orange">DirectionalLightColor</font>和<font color="orange">DirectionalLightDirection</font>。把它们放在Light的顶部定义的<font color="blue">**_CustomLight**</font>缓冲区中。

```glsl
CBUFFER_START(_CustomLight)
	float3 _DirectionalLightColor;
	float3 _DirectionalLightDirection;
CBUFFER_END
```

Use these values instead of constants in <font color="blue">GetDirectionalLight</font>.

```c#
Light GetDirectionalLight () {
	Light light;
	light.color = _DirectionalLightColor;
	light.direction = _DirectionalLightDirection;
	return light;
}
```

现在我们的RP必须将灯光数据发送到GPU。我们将为此创建一个新的照明类。

它的工作原理与<font color="green">CameraRenderer</font>类似，但用于灯光。给它一个带有<font color="red">context </font>参数的公共<font color="blue">Setup</font>方法，在这个方法中它会调用一个单独的<font color="green">SetupDirectionalLight</font>方法。

虽然不是严格意义上的需要，但我们也给它一个<font color="RoyalBlue">**command buffer**</font>，在完成后执行，这对调试很方便。另一种方法是添加一个<font color="green">buffer parameter</font>。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class Lighting {

	const string bufferName = "Lighting";

	CommandBuffer buffer = new CommandBuffer {
		name = bufferName
	};
	
	public void Setup (ScriptableRenderContext context) {
		buffer.BeginSample(bufferName);
		SetupDirectionalLight();
		buffer.EndSample(bufferName);
		context.ExecuteCommandBuffer(buffer);
		buffer.Clear();
	}
	
	void SetupDirectionalLight () {}
}
```

追踪两个<font color="green">shader properties</font>。

```c#
	static int
		dirLightColorId = Shader.PropertyToID("_DirectionalLightColor"),
		dirLightDirectionId = Shader.PropertyToID("_DirectionalLightDirection");
```

我们可以通过<font color="red">   **RenderSettings.sun**</font>  访问场景的主灯<font color="orange">most important directional light</font> 。这可以让我们得到默认的最重要的方向性灯光，它也可以通过<font color="RoyalBlue">Window / Rendering / Lighting Settings</font>明确地配置。

使用<font color="RoyalBlue">**CommandBuffer.SetGlobalVector**</font>来发送灯光数据到GPU。

颜色是灯光在线性空间中的颜色，而方向是灯光变换的正向矢量被否定。

```c#
	void SetupDirectionalLight () {
		Light light = RenderSettings.sun;
		buffer.SetGlobalVector(dirLightColorId, light.color.linear);
		buffer.SetGlobalVector(dirLightDirectionId, -				             			light.transform.forward);
	}
```

>   [SetGlobalVector ](https://docs.unity3d.com/ScriptReference/Rendering.CommandBuffer.SetGlobalVector.html)添加“设置全局着色器矢量属性”命令。

灯的颜色属性是其配置的颜色，但灯也有一个单独的强度系数。最终的颜色是两者相乘的。

```c#
	buffer.SetGlobalVector(
			dirLightColorId, light.color.linear * light.intensity
		);
```

给<font color="green">CameraRenderer</font>	一个<font color="green">Lighting实例</font>	，在绘制可见几何体之前用它来设置照明。

```c#
	Lighting lighting = new Lighting();

	public void Render (
		ScriptableRenderContext context, Camera camera,
		bool useDynamicBatching, bool useGPUInstancing
	) {
		…

		Setup();
		lighting.Setup(context);
		DrawVisibleGeometry(useDynamicBatching, useGPUInstancing);
		DrawUnsupportedShaders();
		DrawGizmos();
		Submit();
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lights/lit-by-sun.png" alt="img" style="zoom:50%;" />

<center>Lit by the sun.</center>

#### [2.4、Visible Lights](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.4)

在<font color="green">culling</font> 时，Unity也会计算出哪些灯光会影响摄像机可见的空间<font color="blue">( space visible to the camera)</font>	。

我们可以依靠这些信息而不是全局的太阳。要做到这一点，<font color="orange">**Lighting**</font>需要访问 <font color="red">**culling results,**</font>     

所以在Setup中添加一个<font color="green">parameter</font>	 ，并将其存储在一个字段中，以方便使用。然后我们可以支持一个以上的灯光，所以用一个新的<font color="green">SetupLights</font>方法取代<font color="green">SetupDirectionalLight</font>	的调用。

```c#
	CullingResults cullingResults;

	public void Setup (
		ScriptableRenderContext context, CullingResults cullingResults
	) {
		this.cullingResults = cullingResults;
		buffer.BeginSample(bufferName);
		//SetupDirectionalLight();
		SetupLights();
		…
	}
	
	void SetupLights () {}
```

在<font color="red">CameraRenderer.Render</font>     中调用Setup时，将<font color="red">culling results</font>     作为一个参数加入。

```c#
	lighting.Setup(context, cullingResults);
```

现在<font color="green">Lighting.SetupLights</font>可以通过<font color="green">culling</font>结果的<font color="red">visibleLights</font>属性来检索所需的数据。它作为<font color="DarkOrchid">Unity.Collections</font>.<font color="green">NativeArray</font>可用于<font color="green">VisibleLight</font>元素类型。

```c#
using Unity.Collections;
using UnityEngine;
using UnityEngine.Rendering;
public class Lighting {
	…

	void SetupLights () {
		NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
	}

	…
}
```

>   **什么是[NativeArray？](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)**
>
>   这是一个类似于数组的结构，但提供了一个与本地内存缓冲区(native memory buffe)的连接。它使管理的C#代码和本地Unity引擎代码之间有效地共享数据成为可能。
>
>   [CullingResults](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.html).visibleLights  、Array of visible lights.<font color="red"> 可见光阵列</font>
>
>   [**VisibleLight**](https://docs.unity3d.com/ScriptReference/Rendering.VisibleLight.html)  [CullingResults.visibleLights](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults-visibleLights.html)将包含一组可见的灯光。可见光结构包含最常用的[Light](https://docs.unity3d.com/ScriptReference/Light.html)变量的打包信息，以及对 Light 组件本身的[VisibleLight.light](https://docs.unity3d.com/ScriptReference/Rendering.VisibleLight-light.html)引用。

#### [2.5、Multiple Directional Lights](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.5)

使用<font color="red"> visible light data </font>使得支持多个方向的灯光成为可能，但我们必须将所有这些灯光的数据发送到GPU上。

所以我们将使用两个Vector4数组来代替一对向量，再加上一个整数来表示灯光数量。我们还将定义一个定向灯的最大数量，我们可以用它来初始化两个数组字段来缓冲数据。让我们把最大数量设置为四个，这对大多数场景来说应该是足够的。

```c#
	const int maxDirLightCount = 4;

	static int
		//dirLightColorId = Shader.PropertyToID("_DirectionalLightColor"),
		//dirLightDirectionId = Shader.PropertyToID("_DirectionalLightDirection");
		dirLightCountId = Shader.PropertyToID("_DirectionalLightCount"),
		dirLightColorsId = Shader.PropertyToID("_DirectionalLightColors"),
		dirLightDirectionsId = Shader.PropertyToID("_DirectionalLightDirections");

	static Vector4[]
		dirLightColors = new Vector4[maxDirLightCount],
		dirLightDirections = new Vector4[maxDirLightCount];
```

>   Why not use structured buffers?
>   因为shader对structured buffers的支持还不够好。要么根本不支持，要么只在fragment programs中支持，要么性能比普通数组差。好消息是，数据如何在CPU和GPU之间传递的具体细节只在少数地方有影响，所以很容易改变。这就是使用Light struct的另一个好处。

给<font color="green">SetupDirectionalLight</font>添加一个<font color="red">Index</font>和一个<font color="red">VisibleLight</font>参数。让它用提供的<font color="green">Index</font>来设置<font color="red">Color</font>和<font color="red">direction elements</font>。

在这种情况下，最终的颜色是通过<font color="blue">VisibleLight.finalColor</font>属性提供的。

<font color="DarkOrchid ">forward vector</font>可以通过<font color="red">VisibleLight.localToWorldMatrix</font>   property找到。它是矩阵的第三列，而且必须再次被取反。

```c#
void SetupDirectionalLight (int index, VisibleLight visibleLight) {
		dirLightColors[index] = visibleLight.finalColor;
		dirLightDirections[index] = -visibleLight.localToWorldMatrix.GetColumn(2);
	}
```

最后的颜色已经应用了 <font color="DarkOrchid ">light's intensity</font>，但在默认情况下，Unity不会将其转换为线性空间。

我们必须将<font color="green">GraphicsSettings</font>.<font color="red">lightsUseLinearIntensity</font>设置为<font color="blue">true</font>，我们可以在<font color="green">CustomRenderPipeline</font>的构造函数中执行一次。

```c#
public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher
	) {
		this.useDynamicBatching = useDynamicBatching;
		this.useGPUInstancing = useGPUInstancing;
		GraphicsSettings.useScriptableRenderPipelineBatching = useSRPBatcher;
		GraphicsSettings.lightsUseLinearIntensity = true;
	}
```

接下来，遍历所有可见光<font color="red">**Lighting.SetupLights**</font>并为每个元素调用<font color="green">**SetupDirectionalLight**</font>。然后调用缓冲区上的<font color="RoyalBlue">SetGlobalInt和SetGlobalVectorArray</font>将数据发送到 GPU。

```c#
	NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
		for (int i = 0; i < visibleLights.Length; i++) {
			VisibleLight visibleLight = visibleLights[i];
			SetupDirectionalLight(i, visibleLight);
		}

		buffer.SetGlobalInt(dirLightCountId, visibleLights.Length);
		buffer.SetGlobalVectorArray(dirLightColorsId, dirLightColors);
		buffer.SetGlobalVectorArray(dirLightDirectionsId, dirLightDirections);
```

但是我们最多只支持四个方向灯，所以我们应该在达到最大值时中止循环。让我们<font color="red">**跟踪与循环迭代器分开的定向光索引**</font>  。

```c#
	int dirLightCount = 0;
		for (int i = 0; i < visibleLights.Length; i++) {
			VisibleLight visibleLight = visibleLights[i];
			SetupDirectionalLight(dirLightCount++, visibleLight);
			if (dirLightCount >= maxDirLightCount) {
				break;
			}
		}
		buffer.SetGlobalInt(dirLightCountId, dirLightCount);
```

因为我们只支持定向光，所以我们应该忽略其他光类型。我们可以通过检查<font color="blue">lightType</font>可见光的属性是否。等于来做到这一点<font color="blue">LightType.Directional</font>。

```c#
	VisibleLight visibleLight = visibleLights[i];
	//checking  lightType property
			if (visibleLight.lightType == LightType.Directional) {
				SetupDirectionalLight(dirLightCount++, visibleLight);
				if (dirLightCount >= maxDirLightCount) {
					break;
				}
			}
```

这可行，<font color="red">但<font color="green">VisibleLight</font>结构相当大。理想情况下，我们只从native array中检索它一次</font>，并且不将它作为常规参数传递给<font color="green">SetupDirectionalLight</font>，因为那样会复制它。

我们可以使用 Unity 相同的trick 用于该<font color="blue">ScriptableRenderContext.DrawRenderers</font>方法 ，即通过<font color="red">**引用传递参数**</font>。

```c#
	SetupDirectionalLight(dirLightCount++, ref visibleLight);
```

这要求我们也将参数定义为引用。

```c#
	void SetupDirectionalLight (int index, ref VisibleLight visibleLight) { … }
```

#### [2.6、Shader Loop](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#2.6)

调整Light中的<font color="RoyalBlue">_CustomLight缓冲区</font>，使其符合我们新的数据格式。在这种情况下，我们将明确地使用<font color="green">float4</font>作为数组类型。在Shader中，数组有一个固定的大小，它们不能被调整大小。

请确保使用我们在<font color="green">Lighting</font>中定义的最大尺寸。

```glsl
#define MAX_DIRECTIONAL_LIGHT_COUNT 4

CBUFFER_START(_CustomLight)
	//float4 _DirectionalLightColor;
	//float4 _DirectionalLightDirection;
	int _DirectionalLightCount;
	float4 _DirectionalLightColors[MAX_DIRECTIONAL_LIGHT_COUNT];
	float4 _DirectionalLightDirections[MAX_DIRECTIONAL_LIGHT_COUNT];
CBUFFER_END
```

添加一个函数来获取定向光计数<font color="green">(directional light count )</font>并进行调整<font color="green">GetDirectionalLight</font>，以便它检索特定光指数的数据。

```c#
int GetDirectionalLightCount () {
	return _DirectionalLightCount;
}

Light GetDirectionalLight (int index) {
	Light light;
	light.color = _DirectionalLightColors[index].rgb;
	light.direction = _DirectionalLightDirections[index].xyz;
	return light;
}
```

然后<font color="green">GetLight</font>针对表面进行调整，使其使用<font color="red">**for循环来累积所有定向光的贡献**</font>。

```c#
float3 GetLighting (Surface surface) {
	float3 color = 0.0;
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		color += GetLighting(surface, GetDirectionalLight(i));
	}
	return color;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/lights/four-directional-lights.png" alt="img" style="zoom:50%;" />

<center>Four directional lights.</center>

现在我们的着色器最多支持四个方向灯。通常只需要一个定向光来表示太阳或月亮，但也许在一个有多个太阳的行星上有一个场景。方向灯也可用于模拟多个大型灯光装置，例如大型体育场的灯光装置。

如果你的游戏总是有一个单一的定向光，那么你可以摆脱循环，或者制作多个着色器变体。但对于本教程，我们将保持简单并坚持使用一个通用循环。最好的性能总是通过去除所有你不需要的东西来实现的，尽管它并不总是产生显着的差异。

#### 2.7、Shader Target Level

具有可变长度的循环曾经是着色器的问题，但现代 GPU 可以毫无问题地处理它们，尤其是当绘制调用的所有片段以相同的方式迭代相同的数据时。但是，默认情况下，<font color="green">OpenGL ES 2.0 和 WebGL 1.0</font> 图形 API 无法处理此类循环。

我们可以通过合并硬编码的最大值来使其工作，例如通过<font color="green">GetDirectionalLight</font>返回<font color="RoyalBlue">min(_DirectionalLightCount, MAX_DIRECTIONAL_LIGHT_COUNT).</font> 这使得展开循环成为可能，将其变成一系列条件代码块

在非常老式的硬件上，所有的代码块都会被执行，它们的贡献通过条件赋值来控制。虽然我们可以使其正常工作，但它使代码更加复杂，因为我们还必须做出其他调整。

所以我选择忽略这些限制，为了简单起见，在构建中关闭<font color="red">WebGL 1.0和OpenGL ES 2.0</font>支持。反正它们不支持线性照明。我们还可以通过#pragma target 3.5指令，将着色器通道的目标级别提高到3.5，从而避免为它们编译OpenGL ES 2.0着色器变体。让我们保持一致，对两个着色器都这样做。

```glsl
		HLSLPROGRAM
			#pragma target 3.5
			…
			ENDHLSL
```

### [3、BRDF](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3)

我们目前使用的是一个非常简单的 <font color="RoyalBlue">lighting model</font>，只适合于完全漫反射表面(<font color="green">diffuse surfaces</font> )。我们可以通过应用 <font color="red">bidirectional reflectance distribution function</font>（双向反射率分布函数简称BRDF）来实现更多的变化和真实的照明。有很多这样的函数。我们将使用<font color="RoyalBlue">Universal RP</font>使用的那个函数，它用一些真实性来换取性能。

#### 3.1、Incoming Light

当一束光正面击中<font color="RoyalBlue">surface fragment</font>时，它的所有能量都会影响这个<font color="green">surface fragment</font>。为了简单起见，我们将假设光束的宽度与fragment的一致。

当光线方向的 L和表面N对齐，所以N⋅L= 1

当它们不对齐时，至少有一部分光束会错过surface fragment，所以影响fragment的能量较少。影响fragment的能量部分是<font color="red">N⋅L</font>的结果，意味着表面越远离光线，就越不会受到光线的影响。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/incoming-light.png" alt="img" style="zoom:50%;" />

<center>Incoming light portion.</center>

#### 3.2、Outgoing Light

我们不能直接看到到达表面的光线。我们只看到从表面反弹到相机或我们眼睛的部分。如果表面是一面完全平坦的镜子，那么光就会从它身上反射出来，其出射角等于入射角。我们只有在相机对准它的情况下才会看到这种光。这就是所谓的镜面反射。这是对光的相互作用的简化，但对我们的目的来说已经足够好了。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/specular-reflection.png)

<center>Perfect specular reflection.</center>

但是，如果表面不是完全平坦的，那么光就会被散射，因为fragments 实际上是由许多更小的fragments 组成的，这些fragments 有不同的方向。这就把光束分割成不同方向的小光束，这就有效地模糊了镜面反射。我们最终可能会看到一些散射的光，即使没有对准完美的反射方向。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/scattered-reflection.png)

<center>Scattered specular reflection.</center>

除此之外，光线还会穿透表面，四处反弹，以不同的角度射出，还有其他我们不需要考虑的事情。走到极端，我们最终会得到一个完美的漫反射表面，它能将光线均匀地散射到所有可能的方向。这就是我们目前在着色器中计算的照明。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/diffuse-reflection.png)

<center>Perfect diffuse reflection.</center>

无论相机在哪里，从表面（surface ）收到的散射光（diffused light）量都是一样的。但这意味着，我们观察到的光能远远小于到达表面fragment.的量。

我们应该用某个系数来缩放入射光线。然而，由于这个因素总是相同的，我们可以把它烘托在灯光的颜色和强度中。

我们使用的最终光色代表了从正面照亮的完全白色的漫反射表面碎片中观察到的数量。这只是实际发出的光总量的极小部分。还有其他配置灯光的方法，例如通过指定 lumen or lux,，这使得配置真实的光源更容易，但我们将坚持使用目前的方法。

#### [3.3、Surface Properties](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.3)

表面可以是完美的漫反射，完美的镜子，或介于两者之间的任何东西。我们有多种方法来控制这一点。我们将使用金属的工作流程，这需要我们在Lit shader中添加两个表面属性。

第一个属性是一个表面是<font color="red">**metallic or nonmetalic**</font>(金属的还是非金属的)，也被称为电介质。因为一个表面可以包含两者的混合，我们将为它添加一个范围为0-1的滑块，1表示它是完全金属的。默认的是全电介质。

第二个属性控制表面的<font color="red">**smooth(光滑程度)**</font>。我们也将使用一个范围为0-1的滑块，0表示完全粗糙，1表示完全光滑。我们将使用0.5作为默认值。

```glsl
		_Metallic ("Metallic", Range(0, 1)) = 0
		_Smoothness ("Smoothness", Range(0, 1)) = 0.5
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/metallic-smoothness.png" alt="img" style="zoom:50%;" />

<center>Material with metallic and smoothness sliders.</center>

Add the properties to the <font color="blue">UnityPerMaterial buffer</font>	.

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
	UNITY_DEFINE_INSTANCED_PROP(float, _Metallic)
	UNITY_DEFINE_INSTANCED_PROP(float, _Smoothness)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

And also to <font color="blue">the Surface struct.</font>

```glsl
struct Surface {
	float3 normal;
	float3 color;
	float alpha;
	float metallic;
	float smoothness;
};
```

Copy them to the surface in <font color="blue">LitPassFragment.</font>	

```glsl
	Surface surface;
	surface.normal = normalize(input.normalWS);
	surface.color = base.rgb;
	surface.alpha = base.a;
	surface.metallic = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Metallic);
	surface.smoothness =
		UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Smoothness);
```

And also add support for them to <font color="green">PerObjectMaterialProperties.</font>

```glsl
	static int
		baseColorId = Shader.PropertyToID("_BaseColor"),
		cutoffId = Shader.PropertyToID("_Cutoff"),
		metallicId = Shader.PropertyToID("_Metallic"),
		smoothnessId = Shader.PropertyToID("_Smoothness");

	…

	[SerializeField, Range(0f, 1f)]
	float alphaCutoff = 0.5f, metallic = 0f, smoothness = 0.5f;

	…

	void OnValidate () {
		…
		block.SetFloat(metallicId, metallic);
		block.SetFloat(smoothnessId, smoothness);
		GetComponent<Renderer>().SetPropertyBlock(block);
	}
}
```

#### [3.4、BRDF Properties](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.4)

我们将使用表面属性来计算<font color="red">**BRDF Function**</font>。

它告诉我们，我们最终会看到多少光从一个表面反射出来，这是漫反射和镜面反射的组合。我们需要将表面的颜色分成漫反射和镜面反射两部分，我们还需要知道表面的粗糙程度。

让我们在一个BRDF结构中跟踪这三个值，放在一个单独的<font color="green">BRDF.HLSL</font>文件中。

```glsl
#ifndef CUSTOM_BRDF_INCLUDED
#define CUSTOM_BRDF_INCLUDED

struct BRDF {
	float3 diffuse;
	float3 specular;
	float roughness;
};

#endif
```

添加一个函数来获取一个给定<font color="blue">Surface BRDF数据</font>。从一个完美的漫反射表面开始，所以漫反射部分等于表面颜色，而<font color="red">specular</font>为0、<font color="red">roughness </font>为1。

```c#
BRDF GetBRDF (Surface surface) {
	BRDF brdf;
	brdf.diffuse = surface.color;
	brdf.specular = 0.0;
	brdf.roughness = 1.0;
	return brdf;
}
```

Include *BRDF* after *Light* and before *Lighting*.

```glsl
#include "../ShaderLibrary/Common.hlsl"
#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Light.hlsl"
#include "../ShaderLibrary/BRDF.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```

在<font color="green">GetLighting</font>的两个函数中添加一个<font color="blue">BRDF parameter</font>，然后用<font color="green">diffuse</font>部分而不是整个<font color="green">surface color</font>乘以入射光线。

```c#
float3 GetLighting (Surface surface, BRDF brdf, Light light) {
	return IncomingLight(surface, light) * brdf.diffuse;
}

float3 GetLighting (Surface surface, BRDF brdf) {
	float3 color = 0.0;
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		color += GetLighting(surface, brdf, GetDirectionalLight(i));
	}
	return color;
}
```

最后，在<font color="green">LitPassFragment</font>中获得<font color="blue">BRDF Data</font>并将其传递给<font color="green">GetLighting</font>。

```c#
	BRDF brdf = GetBRDF(surface);
	float3 color = GetLighting(surface, brdf);
```

#### [3.5、Reflectivity](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.5)

How reflective a surface is varies, but in general<font color="red"> metals reflect all light via specular reflection and have zero diffuse reflection</font>。

所以我们要声明反射率等于金属表面的属性。被反射的光线不会被漫反射， so we should scale the diffuse color by one minus the reflectivity in `GetBRDF`.

```c#
	float oneMinusReflectivity = 1.0 - surface.metallic;

	brdf.diffuse = surface.color * oneMinusReflectivity;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/reflectivity.png" alt="img" style="zoom:50%;" />

<center>White spheres with metallic 0, 0.25, 0.5, 0.75, and 1.</center>

在现实中，一些光线也会从电介质表面反弹，这使它们有了亮点。非金属的反射率各不相同，但平均来说大约是<font color="red">**0.04**</font>。让我们将其定义为最小反射率，并添加一个<font color="green">OneMinusReflectivity</font>函数，将范围从0-1调整到0-0.96。这个范围的调整与<font color="red">**Universal RP**</font>的方法相匹配。

```c#
#define MIN_REFLECTIVITY 0.04

float OneMinusReflectivity (float metallic) {
	float range = 1.0 - MIN_REFLECTIVITY;
	return range - metallic * range;
}
```

在<font color="green">GetBRDF</font>中使用该函数来强制执行最小值。

在只渲染漫反射的情况下，这种差别几乎是不明显的，但是当我们加入镜面反射的时候，这种差别就很重要了。没有它，<font color="red">非金属就不会有镜面高光</font>。

```glsl
	float oneMinusReflectivity = OneMinusReflectivity(surface.metallic);
```

#### [3.6、Specular Color](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.6)

被反射到一个方向的光不能同时被反射到另一个方向。这就是所谓的 <font color="blue">energy conservation</font>，也就是说，<font color="orange">outgoing light</font>不能超过<font color="orange">incoming light</font>.。这表明<font color="orange">specular color</font> 应该等于<font color="orange">surface color</font>减去<font color="orange">diffuse color</font> 。

```glsl
brdf.diffuse = surface.color * oneMinusReflectivity;
brdf.specular = surface.color - brdf.diffuse;
```

然而忽略了一个事实，<font color="red">metals affect the color of specular reflections</font>，而非金属则不会。

(介质)<font color="blue">dielectric surfaces</font>的镜面颜色应该是白色的，我们可以通过使用金属属性在最小反射率和表面颜色之间进行插值来代替实现。

```glsl
	brdf.specular = lerp(MIN_REFLECTIVITY, surface.color, surface.metallic);
```

#### [3.7、Roughness](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.7)

<font color="RoyalBlue">Roughness </font>是 <font color="RoyalBlue">smoothness</font>的反面，所以我们可以简单地<font color="green">one minus smoothness</font>。<font color="red">*Core*RP</font>库有一个函数可以做到这一点，名为<font color="green">**PerceptualSmoothnessToPerceptualRoughness**</font>。

我们将使用这个函数，以明确<font color="blue">smoothness</font>，因此也包括<font color="blue">Roughness</font> 都被定义为感知的。我们可以通过<font color="green">**PerceptualRoughnessToRoughness**</font>函数转换为<font color="red">实际的粗糙度值</font> ，将感知值平方化。

这与<font color="red">**Disney lighting model**</font> 相匹配。之所以这样做，是因为在编辑材质时，调整感知版本更直观。

```glsl
	float perceptualRoughness =
		PerceptualSmoothnessToPerceptualRoughness(surface.smoothness);
	brdf.roughness = PerceptualRoughnessToRoughness(perceptualRoughness);
```

这些函数被定义在<font color="blue">Core RP Libary</font>的<font color="green">CommonMaterial HLSL</font>文件中。在包括<font color="green"> core's *Common*</font>.，将其包含在我们的**Common hlsl**中。

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/CommonMaterial.hlsl"
#include "UnityInput.hlsl"
```

#### [3.8、View Direction](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#3.8)

为了确定摄像机与完美反射方向的对齐程度，我们需要知道摄像机的位置。<font color="green">Unity</font>通过<font color="RoyalBlue">float3 </font><font color="red">_WorldSpaceCameraPos</font>提供了这个数据，所以把它添加到**UnityInput**中。

```glsl
float3 _WorldSpaceCameraPos;
```

为了在<font color="green">LitPassFragment</font>中获得 view 方向--从表面到摄像机的方向，我们需要在<font color="green">Varyings</font>中加入世界空间的表面位置。

```c#
struct Varyings {
	float4 positionCS : SV_POSITION;
	float3 positionWS : VAR_POSITION;
	…
};

Varyings LitPassVertex (Attributes input) {
	…
	output.positionWS = TransformObjectToWorld(input.positionOS);
	output.positionCS = TransformWorldToHClip(output.positionWS);
	…
}
```

我们会认为视图方向是 <font color="green">surface </font>的一部分，所以把它添加到Surface中。

```c#
struct Surface {
	float3 normal;
	float3 viewDirection;
	float3 color;
	float alpha;
	float metallic;
	float smoothness;
};
```

在**LitPassFragment**中指定它。它等于摄像机的位置减去片段的位置，经过<font color="RoyalBlue">normalized</font>。

```glsl
	surface.normal = normalize(input.normalWS);
	surface.viewDirection = normalize(_WorldSpaceCameraPos - input.positionWS);
```

#### [3.9、Specular Strength]()

我们观察到的镜面反射的强度取决于我们的观察方向与完美反射方向的匹配程度。我们将使用 <font color="green">Universal RP </font>中使用的相同公式，它是<font color="red">Minimalist CookTorrance BRDF</font> 的一个变体。这个公式包含一些模块，所以我们先给**Common**添加一个方便的Square函数。

```glsl
float Square (float v) {
	return v * v;
}
```

给<font color="green">BRDF</font>添加一个<font color="red">SpecularStrength</font>函数， 参数是surface、BRDF数据和 light。这些参数用于计算

![img](E:\Typora file\SRP\自定义渲染管线_files\clip_image002.png)

其中r是粗糙度，而且所有的点积都是saturated的

<img src="E:\Typora file\SRP\自定义渲染管线_files\2weima.com.png" alt="2weima.com" style="zoom: 33%;" />

N是表面法线，L是光方向，H = L+V 半角方向<font color="red">normalized</font>。这是在 light 、 view方向之间的中途矢量。使用<font color="green">SafeNormalize</font>函数对该向量进行归一化处理，以避免在向量相反的情况下被除以零。

<font color="RoyalBlue">n = 4r + 2</font> 也需要归一化

```c#
float SpecularStrength (Surface surface, BRDF brdf, Light light) {
	float3 h = SafeNormalize(light.direction + surface.viewDirection);
	float nh2 = Square(saturate(dot(surface.normal, h)));
	float lh2 = Square(saturate(dot(light.direction, h)));
	float r2 = Square(brdf.roughness);
	float d2 = Square(nh2 * (r2 - 1.0) + 1.00001);
	float normalization = brdf.roughness * 4.0 + 2.0;
	return r2 / (d2 * max(0.1, lh2) * normalization);
}
```

接下来，添加一个<font color="green">DirectBRDF</font>，通过给定一个<font color="blue">表面、BRDF和灯光(Surface NRDF、Light)</font>，返回<font color="red">**通过直接照明获得的颜色**</font>。其结果是由 <font color="orange">specular strength</font>调制的<font color="orange">specular color</font>，加上<font color="orange">diffuse color.</font>

```c#
float3 DirectBRDF (Surface surface, BRDF brdf, Light light) {
	return SpecularStrength(surface, brdf, light) * brdf.specular + brdf.diffuse;
}
```

然后<font color="green">**GetLighting**</font>必须用该函数的结果乘以传入的光线。

```c#
float3 GetLighting (Surface surface, BRDF brdf, Light light) {
	return IncomingLight(surface, light) * DirectBRDF(surface, brdf, light);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/smoothness.png" alt="img" style="zoom:50%;" />

<center>Smoothness top to bottom 0, 0.25, 0.5, 0.75, and 0.95.</center>

我们现在得到了<font color="green">specular reflections</font>，它为我们的表面增加了高光。对于完全粗糙的表面，高光是模仿漫反射。

比较光滑的表面会有一个更集中的高光。一个完全光滑的表面会有一个无限小的亮点，我们看不到。需要一些散射来使其可见。

由于能量守恒，对于<font color="green"> smooth surfaces</font>，高光部分可以变得非常明亮，因为大部分到达表面碎片的光线都被聚焦了。因此，我们最终看到的光线要比高光处的漫反射所能看到的多得多。你可以通过将最终渲染的颜色放大来验证这一点。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/colors-001.png" alt="img" style="zoom:50%;" />

<center>Final color divided by 100.</center>

你也可以通过使用白色以外的底色来验证，金属会影响镜面反射的颜色，而非金属则不会。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/blue.png" alt="img" style="zoom:50%;" />

<center>Blue base color.</center>

我们现在<font color="orange">functional direct lighting</font> 是可信的，尽管目前的结果是太暗了--特别是对于金属--因为我们还不支持<font color="orange">environmental reflections </font>。

在这一点上，一个统一的黑色环境会比默认的天空盒更真实，但这使我们的物体更难看。添加更多的灯光也是如此。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/four-lights.png" alt="img" style="zoom:50%;" />

<center>Four lights.</center>

#### 3.10、Mesh Ball

让我们也为**MeshBal**l添加对不同<font color="RoyalBlue">metallic 和 smoothness </font>属性的支持。这需要添加两个浮动数组。

```c#
static int
		baseColorId = Shader.PropertyToID("_BaseColor"),
		metallicId = Shader.PropertyToID("_Metallic"),
		smoothnessId = Shader.PropertyToID("_Smoothness");

	…
	float[]
		metallic = new float[1023],
		smoothness = new float[1023];

	…

	void Update () {
		if (block == null) {
			block = new MaterialPropertyBlock();
			block.SetVectorArray(baseColorId, baseColors);
			block.SetFloatArray(metallicId, metallic);
			block.SetFloatArray(smoothnessId, smoothness);
		}
		Graphics.DrawMeshInstanced(mesh, 0, material, matrices, 1023, block);
	}
```

让我们把25%的 instances 变成 metallic，并把Awake中的 smoothness 从0.05变化到0.95。

```c#
	baseColors[i] =
				new Vector4(
					Random.value, Random.value, Random.value,
					Random.Range(0.5f, 1f)
				);
			metallic[i] = Random.value < 0.25f ? 1f : 0f;
			smoothness[i] = Random.Range(0.05f, 0.95f);
```

Then make the mesh ball use a lit material.

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/brdf/mesh-ball.png" alt="img" style="zoom:50%;" />

<center>	Lit mesh ball.</center>

### [4、Transparency](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#4)

让我们再次考虑透明度。对象仍然根据其 alpha 值l fade，但现在淡化的是reflected light。这对于 <font color="orange">diffuse reflections</font>是有意义的，因为只有一部分光被反射，而其余光穿过表面。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/transparency/fading.png" alt="img" style="zoom:50%;" />

<center>Fading sphere.</center>

然而，<font color="orange">**specular reflections**</font>也会逐渐消失。在一个完全透明的玻璃中，光线要么通过，要么被反射。<font color="orange">specular reflections</font>不会褪色。我们不能用我们目前的方法来表示这一点。

#### 4.1、Premultiplied Alpha

解决方案是只淡化漫射光，同时保持镜面反射在充分的强度。由于源混合模式适用于所有我们不能使用它的东西，所以让我们将其设置为1，同时仍然使用目标混合模式的<font color="green">1 -source-alpha</font>。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/transparency/src-blend-one-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/transparency/src-blend-one-scene.png" alt="scene" style="zoom:50%;" />

<center>Source blend mode set to one.</center>	

这就恢复了<font color="RoyalBlue">specular reflections</font>，但<font color="RoyalBlue">diffuse reflections</font>不再消退了。

我们通过将<font color="RoyalBlue">surface alpha</font> 计入 <font color="RoyalBlue">diffuse color</font>来解决这个问题。因此，我们将<font color="RoyalBlue">diffuse reflections</font>预先乘以alpha，而不是依靠GPU后期的混合。

<font color="DarkOrchid ">这种方法被称为预乘法alpha混合。</font>

在<font color="red">**GetBRDF**</font>中这样做。

```glsl
	brdf.diffuse = surface.color * oneMinusReflectivity;
	brdf.diffuse *= surface.alpha;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/transparency/premultiplied-diffuse.png" alt="img" style="zoom:50%;" />

<center>Premultiplied diffuse.</center>

#### 4.2、[Premultiplication Toggle](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#4.2)

用diffuse预乘alpha可以有效地将物体变成玻璃，而普通的<font color="green">alpha blend</font>则使物体只有效地部分存在。

让我们支持这两种方法，在<font color="red">GetBRDF中添加一个布尔参数来控制我们是否对alpha进行预乘</font>，默认设置为false。

```c#
BRDF GetBRDF (inout Surface surface, bool applyAlphaToDiffuse = false) {
	…
	if (applyAlphaToDiffuse) {
		brdf.diffuse *= surface.alpha;
	}

	…
}
```

我们可以使用<font color="red">**_PREMULTIPLY_ALPHA**</font>关键字来决定在<font color="green">LitPassFragment</font>中使用哪种方法，类似于我们控制alpha剪切的方式。

```glsl
	#if defined(_PREMULTIPLY_ALPHA)
		BRDF brdf = GetBRDF(surface, true);
	#else
		BRDF brdf = GetBRDF(surface);
	#endif
	float3 color = GetLighting(surface, brdf);
	return float4(color, surface.alpha);
```

Add a shader feature for the keyword to the <font color="green">Pass</font> of Lit.

```glsl
	#pragma shader_feature _CLIPPING
	#pragma shader_feature _PREMULTIPLY_ALPHA
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/transparency/premultiply-alpha-toggle.png" alt="img" style="zoom:50%;" />

<center>预乘 alpha 切换。</center>

### [5、Shader GUI](https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/#5)

我们现在支持多种渲染模式，每种模式都需要特定的设置。为了方便在不同的模式之间进行切换，让我们在材质检查器中添加一些按钮来应用预设的配置。

在**Lit shader**的主块中添加一个**CustomEditor "CustomShaderGUI** "语句。

```c#
Shader "Custom RP/Lit" {
	…

	CustomEditor "CustomShaderGUI"
}
```

这将指示Unity编辑器使用CustomShaderGUI类的一个实例来绘制使用Lit Shader's inspector。为该类创建一个脚本资产，并把它放在一个new *Custom RP / Editor* folder文件夹中。

我们需要使用UnityEditor、UnityEngine和UnityEngine.Rendering命名空间。该类必须扩展ShaderGUI并覆盖公共的OnGUI方法，该方法有一个MaterialEditor和一个MaterialProperty数组参数。让它调用基方法，所以我们最终使用默认的检查器。

```c#
using UnityEditor;
using UnityEngine;
using UnityEngine.Rendering;

public class CustomShaderGUI : ShaderGUI {

	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		base.OnGUI(materialEditor, properties);
	}
}
```

#### 5.1、Setting Properties and Keywords

为了完成我们的工作，我们需要访问三样东西，我们将把它们存储在fields中。

首先是材料编辑器，它是负责显示和编辑材料的底层编辑器对象。

第二是对正在编辑的材料的引用，我们可以通过编辑器的target属性来检索。它被定义为一个`Object`数组，因为target是通用 `Editor` class的一个属性。

第三是可以被编辑的属性数组。

```c#
MaterialEditor editor;
	Object[] materials;
	MaterialProperty[] properties;

	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		base.OnGUI(materialEditor, properties);
		editor = materialEditor;
		materials = materialEditor.targets;
		this.properties = properties;
	}
```

要设置一个property，我们首先要在数组中找到它，为此我们可以使用ShaderGUI.FindPropery方法，传给它一个名称和属性数组。然后我们可以调整它的值，通过赋值给它的floatValue属性。将其封装在一个方便的SetProperty方法中，该方法有一个名称和一个值参数。

```c#
void SetProperty (string name, float value) {
		FindProperty(name, properties).floatValue = value;
	}
```

设置关键字有点复杂。我们将`SetKeyword`为此创建一个方法，使用一个名称和一个布尔参数来指示是否应启用或禁用该关键字。我们必须调用其中一个`EnableKeyword`或`DisableKeyword`所有材料，将关键字名称传递给它们。

```c#
void SetKeyword (string keyword, bool enabled) {
		if (enabled) {
			foreach (Material m in materials) {
				m.EnableKeyword(keyword);
			}
		}
		else {
			foreach (Material m in materials) {
				m.DisableKeyword(keyword);
			}
		}
	}
```

[DisableKeyword](https://docs.unity3d.com/ScriptReference/Material.DisableKeyword.html)

让我们也创建一个SetProperty的变体，来切换一个 property–keyword 组合。

```c#
void SetProperty (string name, string keyword, bool value) {
		SetProperty(name, value ? 1f : 0f);
		SetKeyword(keyword, value);
	}
```

现在我们可以定义简单`Clipping`的 , `PremultiplyAlpha`, `SrcBlend`, `DstBlend`, 和`ZWrite`setter 属性。

```c#
	bool Clipping {
		set => SetProperty("_Clipping", "_CLIPPING", value);
	}

	bool PremultiplyAlpha {
		set => SetProperty("_PremulAlpha", "_PREMULTIPLY_ALPHA", value);
	}

	BlendMode SrcBlend {
		set => SetProperty("_SrcBlend", (float)value);
	}

	BlendMode DstBlend {
		set => SetProperty("_DstBlend", (float)value);
	}

	bool ZWrite {
		set => SetProperty("_ZWrite", value ? 1f : 0f);
	}
```

`RenderQueue`最后，通过分配给所有材质的属性来设置渲染队列。我们可以`RenderQueue`为此使用枚举。

```c#
	RenderQueue RenderQueue {
		set {
			foreach (Material m in materials) {
				m.renderQueue = (int)value;
			}
		}
	}
```

[RegisterPropertyChangeUndo](https://docs.unity3d.com/ScriptReference/MaterialEditor.RegisterPropertyChangeUndo.html)

#### 5.2、Preset Buttons

一个按钮可以通过GUILayout.Button方法被创建，传递给它一个label，这将是一个预设的名字。如果该方法返回true，那么它就被按下了。在应用预设之前，我们应该在编辑器中注册一个撤销步骤，这可以通过调用RegisterPropertyChangeUndo对其进行命名来完成。

因为这段代码对所有的预设都是一样的，所以把它放在一个PresetButton方法中，该方法返回预设是否应该被应用。

```c#
	bool PresetButton (string name) {
		if (GUILayout.Button(name)) {
			editor.RegisterPropertyChangeUndo(name);
			return true;
		}
		return false;
	}
```

我们将为每个预设创建一个单独的方法，从默认的不透明模式开始。让它在激活时适当地设置属性。

```c#
	void OpaquePreset () {
		if (PresetButton("Opaque")) {
			Clipping = false;
			PremultiplyAlpha = false;
			SrcBlend = BlendMode.One;
			DstBlend = BlendMode.Zero;
			ZWrite = true;
			RenderQueue = RenderQueue.Geometry;
		}
	}
```

第二个预设是Clip，它是Opaque的copy ，开启了剪切功能，队列设置为AlphaTest。

```c#
	void ClipPreset () {
		if (PresetButton("Clip")) {
			Clipping = true;
			PremultiplyAlpha = false;
			SrcBlend = BlendMode.One;
			DstBlend = BlendMode.Zero;
			ZWrite = true;
			RenderQueue = RenderQueue.AlphaTest;
		}
	}
```

第三个预设是用于standard transparency，它淡化了物体，所以我们将它命名为Fade。它是copy of *Opaque*，有调整过的混合模式和队列，另外没有写深度。

```c#
//standard transparency
   void FadePreset () {
		if (PresetButton("Fade")) {
			Clipping = false;
			PremultiplyAlpha = false;
			SrcBlend = BlendMode.SrcAlpha;
			DstBlend = BlendMode.OneMinusSrcAlpha;
			ZWrite = false;
			RenderQueue = RenderQueue.Transparent;
		}
	}
```

第四个预设是Fade的一个变体，它应用了预乘的alpha混合。我们将它命名为 "*Transparent*"，因为它适用于具有semitransparent surfaces with correct lighting.。

```c#
	void TransparentPreset () {
		if (PresetButton("Transparent")) {
			Clipping = false;
			PremultiplyAlpha = true;
			SrcBlend = BlendMode.One;
			DstBlend = BlendMode.OneMinusSrcAlpha;
			ZWrite = false;
			RenderQueue = RenderQueue.Transparent;
		}
	}
```

在OnGUI的结尾处调用预设方法，这样它们就会显示在默认检查器的下面。

```c#
	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		…

		OpaquePreset();
		ClipPreset();
		FadePreset();
		TransparentPreset();
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/shader-gui/preset-buttons.png" alt="img" style="zoom:50%;" />

​																					Preset buttons.

预设按钮不会被经常使用，所以让我们把它们放在一个默认为折叠的折页中。这是通过调用EditorGUILayout.Foldout与当前的折叠状态、标签和表示点击它应该切换其状态的true来实现的。它返回新的折叠状态，我们应该将其存储在一个字段中。只有在折线图打开时才会画出按钮。

```c#
	bool showPresets;

	…

	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		…

		EditorGUILayout.Space();
		showPresets = EditorGUILayout.Foldout(showPresets, "Presets", true);
		if (showPresets) {
			OpaquePreset();
			ClipPreset();
			FadePreset();
			TransparentPreset();
		}
	}
```

#### 5.3、Presets for Unlit

We can also use the custom shader GUI for our *Unlit* shader.

```
Shader "Custom RP/Unlit" {
	…

	CustomEditor "CustomShaderGUI"
}
```

然而，激活一个预设将导致一个错误，因为我们试图设置一个Shader没有的属性。我们可以通过调整SetProperty来防止这种情况。

让它在调用FindProperty时使用false作为附加参数，表明如果没有找到该属性，它不应该记录错误。然后，结果将是null，所以只有在这种情况下才设置值。同时返回该属性是否存在。

```c#
	bool SetProperty (string name, float value) {
		MaterialProperty property = FindProperty(name, properties, false);
		if (property != null) {
			property.floatValue = value;
			return true;
		}
		return false;
	}
```

然后调整SetProperty的关键字版本，以便它只在相关属性存在的情况下设置关键字。

```c#
	void SetProperty (string name, string keyword, bool value) {
		if (SetProperty(name, value ? 1f : 0f)) {
			SetKeyword(keyword, value);
		}
	}
```

#### 5.4、No Transparency

现在，预设也适用于使用*Unlit* shader，尽管透明模式在这种情况下没有什么意义，因为相关的属性并不存在。让我们在不相关的时候隐藏这个预设。

首先，添加一个HasProperty方法，返回一个属性是否存在。

```
bool HasProperty (string name) =>
		FindProperty(name, properties, false) != null;
```

第二，创建一个方便的属性来检查_PremultiplyAlpha是否存在。

```
bool HasPremultiplyAlpha => HasProperty("_PremulAlpha");
```

最后，通过在TransparentPreset中首先检查它，使透明预设的一切都以该属性为条件。

```
	if (HasPremultiplyAlpha && PresetButton("Transparent")) { … }
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/directional-lights/shader-gui/no-transparent-preset.png" alt="img" style="zoom:50%;" />

​												*Unlit materials lack* *Transparent* *preset.*

### 7、strcut

<font color="orange">lit.Shader.BRDF brdf = GetBRDF(surface, true);</font> 

<font color="orange">lit.Shader.float3 color = GetLighting(surface, brdf);</font> =><font color="red">**Lighting.GetLighting（）**</font>
{
	=>GetLighting(surface,  brdf, light)		 返回表面和光的最终照明
	=>GetDirectionalLight()				 			提供光线数据
} 

GetLighting (surface,  brdf, light)

{

​	IncomingLight(surface, light)          IncomingLight()		计算光照结果

​    DirectBRDF(surface, brdf, light) 										<font color="red">**通过直接照明获得的颜色**</font>

}

<font color="red">Render（）</font>  

{

<font color="red">	=>light.Setup	</font>     						设置灯光实例

}

