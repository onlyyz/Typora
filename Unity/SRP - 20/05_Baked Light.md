# 05_**Baked Light**

-   Bake static global illumination.
-   Sample light maps, probes, and LPPVs.
-   Create a meta pass.
-   Support emissive surfaces.

![img](E:\Typora file\SRP\assets\tutorial-image.jpg)

*Scene illuminated by a single mixed-mode light, plus a little emission.*

### 1、Baking Static Light

到目前为止，我们已经在渲染时计算了所有的光照，但这并不是唯一的选择。

照明也可以提前计算并存储在<font color="RoyalBlue"> light map</font>和探针中。这样做的主要原因有两个：减少实时计算量和增加无法在运行时计算的间接照明。后者是统称为全局照明的一部分：光线不是直接来自光源，而是通过反射、来自环境或来自发光表面的间接照明。

<font color="red">baked lighting</font>的缺点是它是静态的，所以不能在运行时改变。它还需要被存储，这就增加了构建尺寸和内存的使用。

>   实时全局光照怎么办？
>   Unity使用Enlighten系统进行实时全局光照，但这已经被废弃了，所以我们不会使用它。除此之外，反射探针可以在运行时渲染，以创建镜面环境反射，但我们不会在本教程中涉及它们。

#### [1.1Scene Lighting Settings](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#1.1)

全局照明是通过 "<font color="green">*Lighting* window "</font>窗口的 "场景 "选项卡对每个场景进行配置。烘烤照明是通过混合照明下的烘烤全局照明切换来启用的。还有一个照明模式选项，我们将其设置为<font color="red">Baked indirect lighting.</font>烘烤间接照明，这意味着我们烘烤所有的<font color="red"> static indirect lighting.</font>

如果你的项目是在Unity 2019.2或更早的版本中创建的，那么你还会看到一个启用实时照明的选项，它应该是禁用的。如果你的项目是在Unity 2019.3或更高版本中创建的，那么这个选项将不会显示。

<img src="E:\Typora file\SRP\assets\baked-indirect.png" alt="img" style="zoom:50%;" />

<center>Baked indirect lighting only.</center>

再往下是<font color="red">Lightmapping Settings</font>部分，可以用来控制<font color="green">lightmapping</font>过程，这是由Unity编辑器完成的。我将使用默认的设置，除了将<font color="green">LightMap</font>分辨率降低到20，禁用压缩光影，并将方向性模式设置为非方向性。我还使用了渐进式CPU光映射。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/lightmapping-settings.png" alt="img" style="zoom:50%;" />

<center>Lightmapping settings.</center>

#### [1.2、Static Objects](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#1.2)

为了演示烘烤照明，我创建了一个场景，用一个绿色的平面作为地面，几个盒子和球体，以及中心的一个结构，这个结构只有一个开放的侧面，所以它的内部是完全被阴影覆盖的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/scene-with-ceiling.png" alt="img" style="zoom:50%;" />

<center>内部黑暗的场景。</center>

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/scene-without-ceiling.png" alt="img" style="zoom:50%;" />

<center>没有天花板的相同场景。</center>

这个场景有一个单一的定向光，它的<font color="green">*Mode*</font>设置<font color="green">*Mixed*</font>. 这告诉 Unity 它应该为这个光烘<font color="red"> indirect lighting </font>。除此之外，灯光仍然像普通的实时灯光一样工作。

<img src="E:\Typora file\SRP\assets\mixed-mode-light.png" alt="img" style="zoom:50%;" />

<center>Mixed-mode light.</center>

我还将地平面和所有立方体--包括那些形成结构的立方体--纳入烘烤过程。

它们将成为光线反弹的对象，从而成为<font color="red">间接光线</font>。这可以通过启用<font color="blue">MeshRenderer</font>组件的<font color="red">Contribute Global Illumination</font>（贡献全局照明）开关来实现。

启用这个功能也会自动将<font color="RoyalBlue">automatically switches their *Receive Global Illumination* mode to *Lightmaps* _ 它们的接收全局照明模式切换为光照图</font>，这意味着到达它们表面的间接光会被烤成<font color="red">light  map_ 光照图</font>。你也可以通过在对象的静态下拉列表中启用贡献全局照明<font color="red">（Contribute GI）</font>，或者使其完全静态化来启用这个模式。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/contribute-gi.png" alt="img" style="zoom:50%;" />

<center>Contribute global illumination enabled.</center>

一旦启用，场景的光照将再次被烘焙，前提是在<font color="red">Lighting</font>窗口中启用了<font color="red">Auto Generate（自动生成）</font>，否则你就必须按下<font color="RoyalBlue">Generate Lighting（生成光照）</font>按钮。<font color="RoyalBlue">Lightmapping settings _ 光照映射设置</font>也会显示在<font color="blue">MeshRenderer</font>组件中，包括包含物体的光照映射的视图。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/baked-indirect-map.png" alt="img" style="zoom:50%;" />

<center>Map of baked received indirect light.</center>

球体没有显示在光照图中，因为它们对<font color="red"> global illumination _ 全局照明</font>没有贡献，因此被认为是<font color="red">dynamic动态的</font>。

它们将不得不依赖于<font color="red"> light probes _ 光探针</font>，这一点我们将在后面介绍。静态物体也可以通过将它们的接收<font color="green">*Receive Global Illumination*</font>切换回光探针而从<font color="RoyalBlue"> map</font>中排除。它们仍然会影响烘烤的结果，但不会占用<font color="green"> light map.</font>的空间。

#### [1.3、Fully-Baked Light](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#1.3)

<font color="red"> baked lighting</font> 的照明大部分是蓝色的，因为它被天空盒所支配，代表了来自环境天空的间接照明。中心的建筑物周围较亮的区域是由光源从地面和墙壁上反弹的间接照明造成的。

我们还可以把所有的照明都烘烤到<font color="red">map</font>中，包括直接和间接照明。这可以通过将灯光的模式设置为烘烤来完成。然后它就不再提供实时照明了。

<img src="E:\Typora file\SRP\assets\baked-mode-inspector-1677166478545-11.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\baked-mode-scene.png" alt="scene" style="zoom:50%;" />

<center>No realtime lighting.</center>

<font color="red">Baked</font>的直射光也被当作间接光，从而最终进入地图，使其变得更加明亮。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baking-light/fully-baked-map.png" alt="img" style="zoom:50%;" />

<center>Map of fully baked light.</center>

### 2、Sampling Baked Light

目前所有的东西都被渲染成了纯黑色，因为没有实时光线，而且我们的着色器还不知道<font color="RoyalBlue"> global illumination </font>。我们必须对<font color="green"> light map</font>进行采样以使其发挥作用。

#### [2.1、Global Illumination](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#2.1)

创建一个新的<font color="red">ShaderLibrary/GI.hlsl</font>文件，包含所有与<font color="RoyalBlue">global illumination</font>相关的代码。

在这个文件中，定义一个<font color="green">GI Strut</font>和一个<font color="green">GetGI Funtion</font>来检索它，给定一些<font color="red">light map </font>的UV坐标。间接光来自各个方向，因此只能用于漫反射照明，不能用于镜面。所以给<font color="red">GI struct a diffuse color field</font>。为了调试的目的，最初用光照图的UV填充它。

```glsl
#ifndef CUSTOM_GI_INCLUDED
#define CUSTOM_GI_INCLUDED

struct GI {
	float3 diffuse;
};

GI GetGI (float2 lightMapUV) {
	GI gi;
	gi.diffuse = float3(lightMapUV, 0.0);
	return gi;
}

#endif
```

>   那么，<font color="red">specular global illumination</font>？
>   镜面环境反射通常是通过反射探头提供的，我们将在未来的教程中介绍。屏幕空间的反射是另一种选择。

在<font color="red">GetLighting</font>中添加一个<font color="red">GI参数</font>，在积累实时光照之前，用它来初始化颜色值。在这一点上我们不与表面的漫反射率相乘，所以我们可以看到未经修改的接收光。

```c#
float3 GetLighting (Surface surfaceWS, BRDF brdf, GI gi) {
	ShadowData shadowData = GetShadowData(surfaceWS);
	float3 color = gi.diffuse;
	…
	return color;
}
```

Include *GI* before *Lighting* in *LitPass*.

```c#
#include "../ShaderLibrary/GI.hlsl"
#include "../ShaderLibrary/Lighting.hlsl"
```

在<font color="red">LitPassFragment</font>中获取<font color="green">global illumination data</font>，最初的UV坐标为零，并将其传递给<font color="blue">GetLighting</font>。

```c#
	GI gi = GetGI(0.0);
	float3 color = GetLighting(surface, brdf, gi);
```

#### [2.2、Light Map Coordinates](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#2.2)

为了得到<font color="red">light map UV coordinates</font>Unity必须将它们发送到Shader。我们必须指示流水线为<font color="DarkOrchid ">每个被光照的物体</font>做这件事。这可以通过在<font color="red">CameraRenderer.DrawVisibleGeometry</font>中把绘图设置的每个对象数据属性设置为<font color="green">PerObjectData.Lightmaps</font>来完成。

[PerObjectData](https://docs.unity3d.com/ScriptReference/Rendering.PerObjectData.html) 	在渲染过程中要设置什么样的每个对象数据。

```c++
	var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		) {
			enableDynamicBatching = useDynamicBatching,
			enableInstancing = useGPUInstancing,
			perObjectData = PerObjectData.Lightmaps
		};
```

Unity现在会用具有<font color="red">LIGHTMAP_ON</font>关键字的Shader variant来渲染<font color="green">lightmapped objects</font>。在我们的Lit着色器的<font color="blue">CustomLit</font>通道中添加一个多编译指令。

```glsl
	#pragma multi_compile _ LIGHTMAP_ON
	#pragma multi_compile_instancing
```

<font color="green"> light map</font>的UV坐标是<font color="green"> **Attributes** </font>顶点数据的一部分。我们必须把它们转移到<font color="green">Varyings</font>中，这样我们才能在<font color="RoyalBlue">LitPassFragment</font>中使用它们。

但我们应该只在需要时才这样做。我们可以使用类似于转移<font color="blue">instancing identifier _ 实例标识符</font>的方法，并依靠<font color="red">GI_ATTRIBUTE_DATA、GI_VARYINGS_DATA和TRANSFER_GI_DATA</font>宏。

```glsl
struct Attributes {
	…
	GI_ATTRIBUTE_DATA
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct Varyings {
	…
	GI_VARYINGS_DATA
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings LitPassVertex (Attributes input) {
	Varyings output;
	UNITY_SETUP_INSTANCE_ID(input);
	UNITY_TRANSFER_INSTANCE_ID(input, output);
	TRANSFER_GI_DATA(input, output);
	…
}
```

加上另一个<font color="red">GI_FRAGMENT_DATA</font>宏，以检索<font color="green">GetGI</font>的必要参数。

```glsl
	GI gi = GetGI(GI_FRAGMENT_DATA(input));
```

我们必须自己定义这些宏，在GI中。除了<font color="red">GI_FRAGMENT_DATA</font>之外，最初将它们定义为无，因为<font color="green">GI_FRAGMENT_DATA</font>将直接为零。

宏的参数列表与函数的参数列表一样，只是没有类型，而且在宏的名称和参数列表之间不允许有空格，否则列表会被解释为宏定义的东西。

```glsl
#define GI_ATTRIBUTE_DATA
#define GI_VARYINGS_DATA
#define TRANSFER_GI_DATA(input, output)
#define GI_FRAGMENT_DATA(input) 0.0
```

当定义<font color="red">LIGHTMAP_ON</font>时，宏应该定义代码，将另一个<font color="red">UV</font>集添加到结构中，复制它，并检索它。光照图的UV是通过第二个纹理坐标通道提供的，所以我们需要在属性中使用<font color="RoyalBlue">TEXCOORD1</font>语义。

```glsl
#if defined(LIGHTMAP_ON)
	#define GI_ATTRIBUTE_DATA float2 lightMapUV : TEXCOORD1;
	#define GI_VARYINGS_DATA float2 lightMapUV : VAR_LIGHT_MAP_UV;
	#define TRANSFER_GI_DATA(input, output) output.lightMapUV = input.lightMapUV;
	#define GI_FRAGMENT_DATA(input) input.lightMapUV
#else
	#define GI_ATTRIBUTE_DATA
	#define GI_VARYINGS_DATA
	#define TRANSFER_GI_DATA(input, output)
	#define GI_FRAGMENT_DATA(input) 0.0
#endif
```

<img src="E:\Typora file\SRP\assets\light-map-uv.png" alt="img" style="zoom:50%;" />

<center>Light map coordinates.</center>

所有静态烘烤的物体现在都显示它们的UV，而所有动态物体都保持黑色。

#### [2.3、Transformed Light Map Coordinates](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#2.3)

<font color="green">light map</font>坐标通常是由Unity自动生成的，或者是导入的网格数据的一部分。他们定义了一个纹理解包，将网格压平，使其映射到纹理坐标。在<font color="green">light map</font>中，每个物体的解包都被缩放和定位，所以每个实例都有自己的空间。这就像应用于基础UV的缩放和平移一样。我们也要把这个应用到<font color="green">light map</font>的UV上。

<font color="green">light map</font>的UV变换作为<font color="red">UnityPerDraw Buffer</font>的一部分被传递给GPU，所以要把它加到那里。它被称为<font color="blue">unity_LightmapST</font>。尽管它已被废弃，但也要在它之后添加<font color="RoyalBlue">unityDynamicLightmapST</font>，否则<font color="red">SRP批处理程序的兼容性会被破坏</font>。

```glsl
CBUFFER_START(UnityPerDraw)
	float4x4 unity_ObjectToWorld;
	float4x4 unity_WorldToObject;
	float4 unity_LODFade;
	real4 unity_WorldTransformParams;

	float4 unity_LightmapST;
	float4 unity_DynamicLightmapST;
CBUFFER_END
```

然后调整<font color="red">TRANSFER_GI_DATA</font>宏，使其应用转换。宏定义可以分成多行，如果每行的末尾都用反斜杠标记，但最后一行除外。

```
	#define TRANSFER_GI_DATA(input, output) \
		output.lightMapUV = input.lightMapUV * \
		unity_LightmapST.xy + unity_LightmapST.zw;
```

<img src="E:\Typora file\SRP\assets\transformed-light-map-uv.png" alt="img" style="zoom: 50%;" />

<center>Transformed light map coordinates.</center>

#### [2.4、Sampling the Light Map](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#2.4)

对<font color="green">light map</font>的采样是<font color="red">GI</font>的责任。<font color="red"> light map texture </font>被称为<font color="red">unity_Lightmap</font>，并伴随着采样器状态。也包括<font color="green">*Core RP Library*, </font>中的<font color="blue">EntityLighting.hlsl</font>，因为我们将用它来检索光照数据。

Code in GI.hlsl

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/EntityLighting.hlsl"

TEXTURE2D(unity_Lightmap);
SAMPLER(samplerunity_Lightmap);
```

创建一个<font color="blue">SampleLightMap</font>函数，当有<font color="green">light map</font>时调用<font color="RoyalBlue">SampleSingleLightmap</font>，否则返回0。在<font color="red">GetGI</font>中使用它来设置漫反射光。

```glsl
float3 SampleLightMap (float2 lightMapUV) {
	#if defined(LIGHTMAP_ON)
		return SampleSingleLightmap(lightMapUV);
	#else
		return 0.0;
	#endif
}

GI GetGI (float2 lightMapUV) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV);
	return gi;
}
```

<font color="red">SampleSingleLightmap</font>函数还需要一些参数。首先，我们必须把纹理和采样器状态作为前两个参数传给它，为此我们可以使用<font color="red">TEXTURE2D_ARGS</font>宏。

```glsl
	return SampleSingleLightmap(
			TEXTURE2D_ARGS(unity_Lightmap, samplerunity_Lightmap), lightMapUV
		);
```

之后是要应用的缩放和平移。因为我们之前已经做过了，所以在这里我们将使用一个身份转换。

```glsl
return SampleSingleLightmap(
			TEXTURE2D_ARGS(unity_Lightmap, samplerunity_Lightmap), lightMapUV,
			float4(1.0, 1.0, 0.0, 0.0)
		);
```

然后是一个布尔值，表示<font color="green"> light map</font>是否被压缩，当<font color="blue">UNITY_LIGHTMAP_FULL_HDR</font>没有被定义时，就是这种情况。

最后一个参数是一个包含解码指令的<font color="green">float4</font>。它的第一个组件使用<font color="red">LIGHTMAP_HDR_MULTIPLIER</font>，第二个组件使用<font color="red">LIGHTMAP_HDR_EXPONENT</font>。它的其他组件没有被使用。

```glsl
	return SampleSingleLightmap(
			TEXTURE2D_ARGS(unity_Lightmap, samplerunity_Lightmap), lightMapUV,
			float4(1.0, 1.0, 0.0, 0.0),
			#if defined(UNITY_LIGHTMAP_FULL_HDR)
				false,
			#else
				true,
			#endif
			float4(LIGHTMAP_HDR_MULTIPLIER, LIGHTMAP_HDR_EXPONENT, 0.0, 0.0)
		);
```

<img src="E:\Typora file\SRP\assets\sampled-baked-light.png" alt="img" style="zoom:50%;" />

<center>Sampled baked light.</center>

#### [2.5、Disabling Environment Lighting](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#2.5)

烘烤的光线是相当明亮的，因为它还包括来自天空的间接照明。我们可以通过将其强度乘数减为零来禁用它。这样我们就可以把注意力集中在单一方向的光线上。

<img src="E:\Typora file\SRP\assets\environment-intensity-inspector-1677234429071-11.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\environment-intensity-scene.png" alt="scene" style="zoom:50%;" />

<center>Environment intensity set to zero.</center>

请注意，现在结构的内部是间接照明，主要是通过地面。

### [3、Light Probes](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#3)

<font color="red">动态物体不影响烘焙的全局光照</font>，但它们可以通过<font color="red">light probes _ 光探针</font>受其影响。

光线探针是场景中的一个点，它通过用三阶多项式，特别是L2球面谐波，来烘烤所有进入的光线。光探针被放置在场景周围，Unity在每个物体之间进行插值，得出它们位置的最终照明近似值。

#### 3.1、Light Probe Group

通过<font color="green">GameObject / Light / Light Probe Group</font>（游戏对象/灯光/灯光探针组）创建一个灯光探针组，将灯光探针添加到场景中。这将创建一个具有<font color="red">LightProbeGroup</font>组件的游戏对象，该组件默认包含六个立方体形状的探头。

启用 "编辑光探针 "后，你可以移动、复制和删除单个探针，就像它们是游戏对象一样。

<img src="E:\Typora file\SRP\assets\light-probes-editing-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/light-probes/light-probes-editing-scene.png" alt="scene" style="zoom:50%;" />

<center>Editing light probe group inside structure.</center>

一个场景中可以有多个探测组。Unity将所有的探针组合在一起，然后创建一个连接它们的四面体网格。每个动态物体最终都在一个四面体内。在其顶点的四个探针被插值，以达到应用于物体的最终照明。如果一个物体最终出现在探针所覆盖的区域之外，就会使用最近的三角形来代替，所以照明可能会显得很奇怪。

默认情况下，当一个动态物体被选中时，会使用小工具来显示影响该物体的探针，以及其位置的内插结果。你可以通过调整照明窗口的调试设置下的光探针可视化来改变这一点。

<img src="E:\Typora file\SRP\assets\light-probes-selected-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\light-probes-selected-scene-1677237244524-22.png" alt="scene" style="zoom:50%;" />

 <center>Light probes used by selected objects.</center>

你在哪里放置探照灯取决于场景。

首先，它们只需要放在动态物体的地方。第二，把它们放在有照明变化的地方。每个探针都是插值的终点，所以把它们放在光照转换的地方。第三，不要把它们放在烘烤的几何体里面，因为它们最终会变成黑色。最后，插值会穿过物体，所以如果一堵墙的两边光照不同，就把探针放在靠近墙的两边。这样，就不会有物体在两边插值了。除此之外，你必须进行试验。

<img src="E:\Typora file\SRP\assets\light-probes-all.png" alt="img" style="zoom:50%;" />

<center>Showing all light probes.</center>

## [3.2、Sampling Probes](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#3.2)

插值的<font color="red">light probe data </font>必须被传递给每个对象的GPU。

我们必须告诉Unity这样做，这次是通过<font color="green">PerObjectData.LightProbe</font>而不是<font color="blue">PerObjectData.Lightmaps</font>。我们需要启用这两个特征标志，所以用布尔OR运算符将它们结合起来。

<font color="RoyalBlue">Scripts in camerarender</font>

```c#
	perObjectData = PerObjectData.Lightmaps | PerObjectData.LightProbe
```

所需的UnityPerDraw数据由七个<font color="green">float4</font>向量组成，代表红、绿、蓝光的多项式分量。它们被命名为<font color="red">unity_SH</font>*，*代表<font color="green">A、B或C</font>。前两个有三个版本，分别是<font color="green">r、g和b</font>的后缀。

<font color="RoyalBlue">Scripts in UnityInput</font>

```glsl
CBUFFER_START(UnityPerDraw)
	…

	float4 unity_SHAr;
	float4 unity_SHAg;
	float4 unity_SHAb;
	float4 unity_SHBr;
	float4 unity_SHBg;
	float4 unity_SHBb;
	float4 unity_SHC;
CBUFFER_END
```

我们通过一个新的<font color="red">SampleLightProbe</font>函数对<font color="red">GI</font>中的光探针进行采样。我们需要一个方向来做这个，所以给它一个世界空间的表面参数。

如果这个对象正在使用<font color="green">light maps</font>，那么返回0。否则返回0和<font color="red">SampleSH9</font>之中的最大值。该函数需要<font color="blue">probe data</font>和<font color="blue">normal vector</font>作为参数。<font color="blue">probe data</font>必须以<font color="red">array of coefficients _ 系数数组</font>的形式提供。

```glsl
float3 SampleLightProbe (Surface surfaceWS) {
	#if defined(LIGHTMAP_ON)
		return 0.0;
	#else
		float4 coefficients[7];
		coefficients[0] = unity_SHAr;
		coefficients[1] = unity_SHAg;
		coefficients[2] = unity_SHAb;
		coefficients[3] = unity_SHBr;
		coefficients[4] = unity_SHBg;
		coefficients[5] = unity_SHBb;
		coefficients[6] = unity_SHC;
		return max(0.0, SampleSH9(coefficients, surfaceWS.normal));
	#endif
}
```

在<font color="red">GetGI</font>中添加一个表面参数，并使其将<font color="red">light probe sample</font>添加到<font color="green"> diffuse light.</font>中。

```glsl
GI GetGI (float2 lightMapUV, Surface surfaceWS) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	return gi;
}
```

最后，在<font color="green">LitPassFragment</font>中把表面传递给它。

```glsl
	GI gi = GetGI(GI_FRAGMENT_DATA(input), surface);
```

<img src="E:\Typora file\SRP\assets\sampling-probes.png" alt="img" style="zoom:67%;" />

<center>Sampling light probes.</center>

#### [3.3、Light Probe Proxy Volumes](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#3.3)

光线探针对相当小的动态物体有效，但由于灯光是基于单点的，所以对较大的物体来说效果并不理想。作为一个例子，我在场景中添加了两个被拉伸的立方体。因为它们的位置位于黑暗区域内，所以立方体是均匀的黑暗，尽管这显然不符合照明的要求。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/light-probes/large-objects-single-probe.png" alt="img" style="zoom:50%;" />

<center>Large objects sampling from one position.</center>

我们可以通过使用<font color="red">light probe proxy volume</font>，简称<font color="red">LPPV</font>来解决这个限制。最简单的做法是在每个立方体上添加一个<font color="green">LightProbeProxyVolume</font>组件，然后将它们的光探针模式设置为使用<font color="red"> Proxy Volume.</font>。

<font color="red">volumes </font>可以用多种方式配置。在这个案例中，我使用了一个<font color="red">custom resolution</font>模式，将<font color="blue">sub-probes</font>沿着立方体的边缘放置，所以它们是可见的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/light-probes/lppv-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\lppv-scene.png" alt="scene" style="zoom:50%;" />

<center>Using LPPVs.</center>

#### [3.4、Sampling LPPVs](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#3.4)

<font color="green">LPPVs</font>也需要向每个物体的GPU发送数据。在这种情况下，我们必须启用<font color="blue">PerObjectData.LightProbeProxyVolume</font>。

```c#
	perObjectData =
				PerObjectData.Lightmaps | PerObjectData.LightProbe |
				PerObjectData.LightProbeProxyVolume
```

四个额外的值必须被添加到<font color="red">UnityPerDraw</font>中：<font color="green">unity_ProbeVolumeParams, unity_ProbeVolumeWorldToObject, unity_ProbeVolumeSizeInv,</font> 和<font color="green">unity_ProbeVolumeMin</font>。第二个是一个矩阵，而其他的是4D向量。

```glsl
CBUFFER_START(UnityPerDraw)
	…

	float4 unity_ProbeVolumeParams;
	float4x4 unity_ProbeVolumeWorldToObject;
	float4 unity_ProbeVolumeSizeInv;
	float4 unity_ProbeVolumeMin;
CBUFFER_END
```

<font color="green"> volume data</font>被存储在一个<font color="green">3D float texture _ 3D浮点纹理</font>中，称为<font color="red">unity_ProbeVolumeSH</font>。通过<font color="RoyalBlue">TEXTURE3D_FLOAT</font>宏把它和它的采样器状态一起添加到GI中。

```glsl
TEXTURE3D_FLOAT(unity_ProbeVolumeSH);
SAMPLER(samplerunity_ProbeVolumeSH);
```

是否使用<font color="red">LPPV or interpolated light probe</font>是通过<font color="red">unity_ProbeVolumeParams</font>的第一个组件来传达的。

如果它被设置了，那么我们必须通过<font color="red">SampleProbeVolumeSH4</font>函数对<font color="green">volume</font>进行采样。我们必须把<font color="green">Texture、Sample</font>传给它，然后是<font color="green">World_position and Normal</font>。然后是<font color="green">矩阵，unity_ProbeVolumeParams.YZ</font>，接着是最小值和大小值的XYZ部分的数据。

```glsl
		if (unity_ProbeVolumeParams.x) {
			return SampleProbeVolumeSH4(
				TEXTURE3D_ARGS(unity_ProbeVolumeSH, samplerunity_ProbeVolumeSH),
				surfaceWS.position, surfaceWS.normal,
				unity_ProbeVolumeWorldToObject,
				unity_ProbeVolumeParams.y, unity_ProbeVolumeParams.z,
				unity_ProbeVolumeMin.xyz, unity_ProbeVolumeSizeInv.xyz
			);
		}
		else {
			float4 coefficients[7];
			coefficients[0] = unity_SHAr;
			coefficients[1] = unity_SHAg;
			coefficients[2] = unity_SHAb;
			coefficients[3] = unity_SHBr;
			coefficients[4] = unity_SHBg;
			coefficients[5] = unity_SHBb;
			coefficients[6] = unity_SHC;
			return max(0.0, SampleSH9(coefficients, surfaceWS.normal));
		}
```

<img src="E:\Typora file\SRP\assets\sampling-lppvs.png" alt="img" style="zoom:50%;" />

<center>Sampling LPPVs.</center>

对<font color="red">LPPV</font>进行采样需要对<font color="green"> volume's space</font>进行转换，同时还要进行一些其他的计算，<font color="green">volume texture</font>采样，以及应用<font color="red"> spherical harmonics _球面谐波</font>。在这种情况下，只有L1球面谐波被应用，所以结果不太精确，但在单个物体的表面会有变化。

### [4、Meta Pass](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4)

因为间接漫射光从表面反弹，它应该受到这些表面的<font color="red">diffuse reflectivity </font>的影响。目前这种情况并没有发生。Unity将我们的表面视为均匀的白色。

Unity使用一个特殊的<font color="red">meta pass</font>来确定烘烤时的反射光。由于我们没有定义这样的传递，Unity使用了默认的传递，最后是白色的。

#### [4.1、Unified Input](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4.1)

添加另一个<font color="green">Pass</font>意味着我们必须再次定义**shader**属性。

让我们从<font color="green">LitPass</font>中提取基础纹理和<font color="blue">UnityPerMaterial</font>缓冲区，并把它放在一个新的<font color="red">Shaders/LitInput.hlsl</font>文件中。

我们还将通过引入<font color="green">TransformBaseUV、GetBase、GetCutoff、GetMetallic和GetSmoothness</font>函数来隐藏实例化代码。给它们都提供一个基础UV参数，即使它是未使用的。一个值是否从地图中获取，也是这样隐藏的。

```glsl
#ifndef CUSTOM_LIT_INPUT_INCLUDED
#define CUSTOM_LIT_INPUT_INCLUDED

TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
	UNITY_DEFINE_INSTANCED_PROP(float, _Metallic)
	UNITY_DEFINE_INSTANCED_PROP(float, _Smoothness)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

float2 TransformBaseUV (float2 baseUV) {
	float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseMap_ST);
	return baseUV * baseST.xy + baseST.zw;
}

float4 GetBase (float2 baseUV) {
	float4 map = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, baseUV);
	float4 color = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
	return map * color;
}

float GetCutoff (float2 baseUV) {
	return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff);
}

float GetMetallic (float2 baseUV) {
	return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Metallic);
}

float GetSmoothness (float2 baseUV) {
	return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Smoothness);
}

#endif
```

为了在<font color="red">Lit</font>的所有通道中包含这个文件，在其<font color="red">SubShader</font>块的顶部，在通道之前添加一个<font color="red">HLSLINCLUDE</font>块。将<font color="green">Common</font>包含在其中，然后是<font color="green">LitInput</font>。这段代码将被插入到所有通道的开头。

```glsl
	SubShader {
		HLSLINCLUDE
		#include "../ShaderLibrary/Common.hlsl"
		#include "LitInput.hlsl"
		ENDHLSL
		
		…
	}
```

从<font color="red">LitPass</font>中删除现在重复的include语句和声明。

```glsl
//#include "../ShaderLibrary/Common.hlsl"
…

//TEXTURE2D(_BaseMap);
//SAMPLER(sampler_BaseMap);

//UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	//…
//UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

在<font color="red">LitPassVertex</font>中使用<font color="green">TransformBaseUV</font>。

```glsl
	//float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseMap_ST);
	output.baseUV = TransformBaseUV(input.baseUV);
```

以及在<font color="red">LitPassFragment</font>中检索着色器属性的相关函数。

```glsl
	//float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
	//float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
	float4 base = GetBase(input.baseUV);
	#if defined(_CLIPPING)
		clip(base.a - GetCutoff(input.baseUV));
	#endif
	
	…
	surface.metallic = GetMetallic(input.baseUV);
	surface.smoothness = GetSmoothness(input.baseUV);
```

给<font color="red">ShadowCasterPass</font>以同样的待遇。

#### [4.2、Unlit](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4.2)

让我们也为<font color="green">Unlit shader</font>做这件事。复制<font color="green">LitInput.hlsl</font>并将其重命名为<font color="RoyalBlue">UnlitInput.hlsl</font>。然后从其<font color="green">UnityPerMaterial</font>版本中删除<font color="blue">_Metallic和_Smoothness</font>。保留<font color="RoyalBlue">GetMetallic和GetSmoothness</font>函数，让它们返回0.0，代表一个非常暗淡的漫反射表面。之后，也给着色器一个<font color="red">HLSLINCLUDE</font>块。

```
		HLSLINCLUDE
		#include "../ShaderLibrary/Common.hlsl"
		#include "UnlitInput.hlsl"
		ENDHLSL
```

就像我们对<font color="red">LitPass</font>所做的那样转换<font color="green">UnlitPass</font>。请注意，<font color="blue">ShadowCasterPass</font>对这两种着色器都能正常工作，尽管它最终会有不同的输入定义。

#### [4.3、Meta Light Mode](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4.3)

在<font color="red">Lit和Unlit</font>着色器中都添加一个新的<font color="green">Pass</font>，<font color="blue">LightMode</font>设置为<font color="red">Meta</font>。这个通道要求<font color="red">Clip</font>总是关闭的，这可以通过添加<font color="red">Cull Off</font>选项进行配置。它将使用<font color="green">MetaPassVertex和MetaPassFragment</font>函数，定义在一个新的<font color="red">MetaPass.hlsl</font>文件中。它不需要多编译指令。

```glsl
		Pass {
			Tags {
				"LightMode" = "Meta"
			}

			Cull Off

			HLSLPROGRAM
			#pragma target 3.5
			#pragma vertex MetaPassVertex
			#pragma fragment MetaPassFragment
			#include "MetaPass.hlsl"
			ENDHLSL
		}
```

我们需要知道<font color="red"> surface's diffuse </font>，所以我们必须在<font color="green">MetaPassFragment</font>中得到它的<font color="red">BRDF数据</font>。因此，我们必须包括<font color="red">*BRDF*, plus *Surface*, *Shadows* and Light</font>，因为它取决于它们。

我们只需要知道<font color="green">object-space position and base UV</font>，最初将<font color="blue">clip-space position to zero</font>。表面可以通过<font color="RoyalBlue">ZERO_INITIALIZE(Surface, surface)</font>初始化为零，之后我们只需要设置<font color="red">color, metallic, and smoothness values</font>。这就足以获得<font color="green">BRDF数据</font>了，但我们首先要返回零。

```glsl
#ifndef CUSTOM_META_PASS_INCLUDED
#define CUSTOM_META_PASS_INCLUDED

#include "../ShaderLibrary/Surface.hlsl"
#include "../ShaderLibrary/Shadows.hlsl"
#include "../ShaderLibrary/Light.hlsl"
#include "../ShaderLibrary/BRDF.hlsl"

struct Attributes {
	float3 positionOS : POSITION;
	float2 baseUV : TEXCOORD0;
};

struct Varyings {
	float4 positionCS : SV_POSITION;
	float2 baseUV : VAR_BASE_UV;
};

Varyings MetaPassVertex (Attributes input) {
	Varyings output;
	output.positionCS = 0.0;
	output.baseUV = TransformBaseUV(input.baseUV);
	return output;
}

float4 MetaPassFragment (Varyings input) : SV_TARGET {
	float4 base = GetBase(input.baseUV);
	Surface surface;
	ZERO_INITIALIZE(Surface, surface);
	surface.color = base.rgb;
	surface.metallic = GetMetallic(input.baseUV);
	surface.smoothness = GetSmoothness(input.baseUV);
	BRDF brdf = GetBRDF(surface);
	float4 meta = 0.0;
	return meta;
}

#endif
```

一旦Unity用我们自己的<font color="red">meta pass</font>再次烘烤场景，所有的<font color="red"> indirect lighting</font>就会消失，因为黑色的表面不会反射什么。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/meta-pass/no-indirect-light.png" alt="img" style="zoom:50%;" />

<center>No more indirect light.</center>

#### [4.4、Light Map Coordinates](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4.4)

就像对<font color="green"> light map </font>进行采样时，我们需要使用<font color="green"> light map UV coordinates</font>。

不同的是，这一次我们朝相反的方向走，用它们来表示<font color="red">XY object-space position.</font>。之后，我们必须将其送入<font color="blue">TransformWorldToHClip</font>，尽管在这种情况下，该函数执行的是一种不同于其名称的变换。

```glsl
struct Attributes {
	float3 positionOS : POSITION;
	float2 baseUV : TEXCOORD0;
	float2 lightMapUV : TEXCOORD1;
};

…

Varyings MetaPassVertex (Attributes input) {
	Varyings output;
	input.positionOS.xy =
		input.lightMapUV * unity_LightmapST.xy + unity_LightmapST.zw;
	output.positionCS = TransformWorldToHClip(input.positionOS);
	output.baseUV = TransformBaseUV(input.baseUV);
	return output;
}
```

我们仍然需要<font color="green"> object-space vertex attribute </font>作为输入，因为Shader希望它存在。事实上，似乎OpenGL不明确使用Z坐标就不会工作。我们将使用Unity自己的<font color="blue">meta pass</font>使用的相同的假赋值，即<font color="red">input.positionOS.z > 0.0 ?FLT_MIN : 0.0。</font>

```glsl
	input.positionOS.xy =
		input.lightMapUV * unity_LightmapST.xy + unity_LightmapST.zw;
	input.positionOS.z = input.positionOS.z > 0.0 ? FLT_MIN : 0.0;
```

#### [4.5、Diffuse Reflectivity](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#4.5)

<font color="red">meta pass</font>可以用来生成不同的数据。所要求的东西是通过一个<font color="green">bool4 </font><font color="red">unity_MetaFragmentControl</font>标志向量来传达的。

```c#
bool4 unity_MetaFragmentControl;
```

如果<font color="red">X分量</font>被设置了，那么就需要<font color="blue">diffuse reflectivity</font>，所以要把它变成RGB结果。A通道应该被设置为1。

```glsl
	float4 meta = 0.0;
	if (unity_MetaFragmentControl.x) {
		meta = float4(brdf.diffuse, 1.0);
	}
	return meta;
```

这足以给<font color="red">reflected light</font>上色，但<font color="green">Unity</font>的<font color="red">meta pass</font>也能提高一些效果，通过添加按<font color="red">half the specular reflectivity scaled by roughness _粗糙度缩放的一半镜面反射率</font>。

这背后的想法是，<font color="red">高镜面但粗糙的材料也会传递一些间接光</font>。

```
meta = float4(brdf.diffuse, 1.0);
		meta.rgb += brdf.specular * brdf.roughness * 0.5;
```

之后，<font color="red">meta</font>结果被提高，通过<font color="red">unity_OneOverOutputBoost与PositivePow</font>的幂方法，然后将其限制在<font color="RoyalBlue">unity_MaxOutputValue</font>。

```glsl
	meta.rgb += brdf.specular * brdf.roughness * 0.5;
		meta.rgb = min(
			PositivePow(meta.rgb, unity_OneOverOutputBoost), unity_MaxOutputValue
		);
```

这些值都是以浮点数提供的。

```glsl
float unity_OneOverOutputBoost;
float unity_MaxOutputValue;
```

<img src="E:\Typora file\SRP\assets\colored-indirect-light.png" alt="img" style="zoom:50%;" />

<center>Colored indirect light, mostly green from the ground.</center>

现在我们得到了正确颜色的<font color="red"> indirect lighting</font>，在<font color="red">GetLighting</font>中也将接收面的<font color="green"> diffuse reflectivity </font>应用于此。

```c#
	float3 color = gi.diffuse * brdf.diffuse;
```

<img src="E:\Typora file\SRP\assets\proper-lighting.png" alt="img" style="zoom:50%;" />

<center>Properly colored baked lighting.</center>

让我们也把<font color="red"> environment lighting </font>再次打开，把它的强度设置为1。

<img src="E:\Typora file\SRP\assets\environment-lighting.png" alt="img" style="zoom:50%;" />

<center>With environment lighting.</center>

最后，将灯光的模式设置为<font color="red">*Mixed*</font>模式。这使得它再次成为一个实时的灯光，所有的<font color="red">indirect diffuse lighting baked.</font>都被烘烘焙出来。

<img src="E:\Typora file\SRP\assets\mixed-lighting.png" alt="img" style="zoom:50%;" />

<center>Mixed lighting.</center>

### 5、Emissive Surfaces

有些表面会自己发光，因此即使在没有其他照明的情况下也能看到。

这可以通过简单地在<font color="green">LitPassFragment</font>的末尾添加一些颜色来实现。这不是一个真正的光源，所以它不会影响其他表面。然而，这种效果可以为烘烤照明做出贡献。

#### [5.1、Emitted Light](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#5.1)

为Lit shader添加两个新的属性：<font color="green"> emission map and color</font>，就像<font color="green">base map and color</font>一样。

然而，我们将为两者使用相同的坐标变换，所以我们不需要为<font color="green"> emission map</font>显示单独的控件。它们可以通过赋予它<font color="red">NoScaleOffset</font>属性而被隐藏。为了支持非常明亮的发射，给颜色添加<font color="red">HDR</font>属性。这使得通过检查器配置亮度大于1的颜色成为可能，显示一个<font color="red">HRD</font>颜色的弹出窗口，而不是常规的窗口。

作为一个例子，做了一个<font color="red"> opaque emissive material </font>，它使用了<font color="green">Default-Particle</font>纹理，其中包含一个圆形的渐变，从而产生了一个明亮的点。

```
		[NoScaleOffset] _EmissionMap("Emission", 2D) = "white" {}
		[HDR] _EmissionColor("Emission", Color) = (0.0, 0.0, 0.0, 0.0)
```

<img src="E:\Typora file\SRP\assets\emissive-material.png" alt="img" style="zoom:50%;" />

<center>Material with emission set to white dots.</center>

将贴图添加到<font color="red">LitInput</font>，将发射颜色添加到<font color="red">UnityPerMaterial</font>。然后添加一个<font color="green">GetEmission</font>函数，它的工作原理和<font color="green">GetBase</font>一样，只是它使用了其他的纹理和颜色。

```glsl
TEXTURE2D(_BaseMap);
TEXTURE2D(_EmissionMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float4, _EmissionColor)
	…
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float3 GetEmission (float2 baseUV) {
	float4 map = SAMPLE_TEXTURE2D(_EmissionMap, sampler_BaseMap, baseUV);
	float4 color = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _EmissionColor);
	return map.rgb * color.rgb;
}
```

在<font color="green">LitPassFragment</font>的结尾处将<font color="red">emission</font>添加到最终颜色。

```c#
	float3 color = GetLighting(surface, brdf, gi);
	color += GetEmission(input.baseUV);
	return float4(color, surface.alpha);
```

同时为<font color="red">UnlitInput</font>添加一个<font color="red">GetEmission</font>函数。在这种情况下，我们只是让它成为<font color="green">GetBase</font>的一个代替。因此，如果你烘烤一个不亮的物体，它最终会发出它的全部颜色。

```c#
float3 GetEmission (float2 baseUV) {
	return GetBase(baseUV).rgb;
}
```

为了使无光材料能够发出非常明亮的光，我们可以在无光的基础颜色属性中添加HDR属性。

```glsl
	[HDR] _BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
```

最后，让我们把<font color="green">emission colo</font>添加到<font color="green">PerObjectMaterialProperties</font>中。在这种情况下，我们可以通过给配置字段加上<font color="red">ColorUsage</font>属性来允许HDR输入。我们必须向它传递两个布尔值。第一个表示是否要显示alpha通道，我们不需要。第二个表示是否允许HDR值。

```glsl
	static int
		baseColorId = Shader.PropertyToID("_BaseColor"),
		cutoffId = Shader.PropertyToID("_Cutoff"),
		metallicId = Shader.PropertyToID("_Metallic"),
		smoothnessId = Shader.PropertyToID("_Smoothness"),
		emissionColorId = Shader.PropertyToID("_EmissionColor");

	…

	[SerializeField, ColorUsage(false, true)]
	Color emissionColor = Color.black;

	…

	void OnValidate () {
		…
		block.SetColor(emissionColorId, emissionColor);
		GetComponent<Renderer>().SetPropertyBlock(block);
	}
```

<img src="E:\Typora file\SRP\assets\per-object-emission.png" alt="img" style="zoom:50%;" />

<center>Per-object emission set to HDR yellow.</center>

我在场景中添加了几个小的发光方块。我让它们为<font color="red">global illumination</font>做贡献，并把它们在<font color="red"> *Lightmap*</font>中的比例增加了一倍，以避免出现UV坐标重叠的警告。当顶点在<font color="red">light map</font>中靠得太近时就会发生这种情况，所以它们必须共享同一个文本。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/emissive-surfaces/emissive-objects.png" alt="img" style="zoom:50%;" />

<center>Emissive cubes; no environment lighting.</center>

#### [5.2、Baked Emission](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#5.2)

<font color="red">Emissive light</font>是通过一个单独的通道进行烘烤的。当<font color="red">unity_MetaFragmentControl</font>的Y标志被设置时，<font color="red">MetaPassFragment</font>应该返回<font color="red">emitted light</font>，再次将A分量设置为1。

```
	if (unity_MetaFragmentControl.x) {
		…
	}
	else if (unity_MetaFragmentControl.y) {
		meta = float4(GetEmission(input.baseUV), 1.0);
	}
```

但这并不会自动发生。我们必须启用每个材料的<font color="red">emission </font>烘烤。我们可以通过在<font color="green">PerObjectMaterialProperties.OnGUI</font>中调用编辑器上的<font color="red">LightmapEmissionProperty</font>来显示这个配置选项。

```
	public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		EditorGUI.BeginChangeCheck();
		base.OnGUI(materialEditor, properties);
		editor = materialEditor;
		materials = materialEditor.targets;
		this.properties = properties;

		BakedEmission();

		…
	}

	void BakedEmission () {
		editor.LightmapEmissionProperty();
	}
```

这使得一个<font color="blue"> *Global Illumination*</font>的下拉菜单显示出来，它最初被设置为无。尽管它的名字叫 "<font color="red">全局照明"</font>，但它只影响<font color="green"> baked emission.</font>。把它改成烘烤的时候，<font color="red">lightmapper</font>就会为<font color="green">emitted light.</font>运行一个单独的通道。还有一个<font color="blue">Realtime</font>选项，但它已经被废弃了。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/emissive-surfaces/emission-set-to-baked.png" alt="img" style="zoom:50%;" />

<center>Emission set to baked.</center>

这还是不行，因为Unity在烘烤时试图避免单独<font color="red">emission pass</font>。如果<font color="blue">material's emission is set to zero </font>，那么它就被忽略了。

然而，这并没有考虑到每个物体的材料属性。我们可以通过在改变<font color="red">emission mode</font>时禁用所有选定材质的<font color="red">globalIlluminationFlags</font>属性中默认的<font color="green">MaterialGlobalIlluminationFlags.EmissiveIsBlack</font>标志来覆盖这一行为。这意味着你应该只在需要时启用<font color="green">*Baked* </font>选项。

[MaterialGlobalIlluminationFlags](https://docs.unity3d.com/ScriptReference/MaterialGlobalIlluminationFlags.html)  材质如何与光照贴图和光照探针交互。

```
	void BakedEmission () {
		EditorGUI.BeginChangeCheck();
		editor.LightmapEmissionProperty();
		if (EditorGUI.EndChangeCheck()) {
			foreach (Material m in editor.targets) {
				m.globalIlluminationFlags &=
					~MaterialGlobalIlluminationFlags.EmissiveIsBlack;
			}
		}
	}
```

<img src="E:\Typora file\SRP\assets\baked-emission-with-light.png" alt="with" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\baked-emission-without-light-1677323826428-13.png" alt="without" style="zoom:50%;" />

<center>Baked emission, with and without directional light.</center>

### 6、Baked Transparency

<font color="green">bake transparent objects</font>也是可能的，但这需要一点额外的努力。

<img src="E:\Typora file\SRP\assets\semitransparent-ceiling.png" alt="img" style="zoom:50%;" />

<center>Semitransparent ceiling treated as opaque.</center>

#### [6.1、Hard-Coded Properties](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#6.1)

硬编码属性

不幸的是，Unity的<font color="green">lightmapper</font>对透明度有一个硬编码的方法。它通过查看材质的队列来确定它是不透明的、剪切的还是透明的。然后，它通过乘以<font color="red">**_MainTex** and **_Color** property, using the *_Cutoff* property for alpha clipping</font>，使用_Cutoff属性进行alpha剪切。

我们的着色器有第三个，但缺乏前两个。目前唯一的方法是将预期的属性添加到我们的**Shader**中，赋予它们<font color="red">HideInInspector</font>属性，这样它们就不会在检查器中显示出来。<font color="green">Unity的SRP Shader</font>也必须处理同样的问题。

```glsl
	[HideInInspector] _MainTex("Texture for Lightmap", 2D) = "white" {}
	[HideInInspector] _Color("Color for Lightmap", Color) = (0.5, 0.5, 0.5, 1.0)
```

#### [6.2、Copying Properties](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#6.2)

我们必须确保<font color="green">MainTex属性</font>指向与<font color="green">BaseMap</font>相同的纹理并使用相同的UV变换。

两个颜色属性也必须是相同的。我们可以在一个新的<font color="red">CopyLightMappingProperties</font>方法中做到这一点，如果发生了变化，我们会在<font color="blue">CustomShaderGUI.OnGUI</font>的结尾处调用这个方法。如果相关的属性存在，就复制它们的值。

```c#
public override void OnGUI (
		MaterialEditor materialEditor, MaterialProperty[] properties
	) {
		…

		if (EditorGUI.EndChangeCheck()) {
			SetShadowCasterPass();
			CopyLightMappingProperties();
		}
	}

	void CopyLightMappingProperties () {
		MaterialProperty mainTex = FindProperty("_MainTex", properties, false);
		MaterialProperty baseMap = FindProperty("_BaseMap", properties, false);
		if (mainTex != null && baseMap != null) {
			mainTex.textureValue = baseMap.textureValue;
			mainTex.textureScaleAndOffset = baseMap.textureScaleAndOffset;
		}
		MaterialProperty color = FindProperty("_Color", properties, false);
		MaterialProperty baseColor =
			FindProperty("_BaseColor", properties, false);
		if (color != null && baseColor != null) {
			color.colorValue = baseColor.colorValue;
		}
	}
```

<img src="E:\Typora file\SRP\assets\transparent-baked.png" alt="img" style="zoom:50%;" />

<center>Transparency correctly baked.</center>

这也适用于剪切的材料。虽然有可能，但不需要在<font color="red">MetaPassFragment</font>中<font color="green">clip fragments</font>，因为透明度是单独处理的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/baked-transparency/cutout-baked.png" alt="img" style="zoom:50%;" />

<center>*Baked clipping.*</center>

### 7、Mesh Ball

我们通过为<font color="red">MeshBall</font>生成的实例添加对全局光照的支持来结束。由于它的实例是在游戏模式下生成的，所以它们不能被烘焙，但只要稍加努力，它们就可以通过光探针接受烘焙的照明。

<img src="E:\Typora file\SRP\assets\unlit.png" alt="img" style="zoom:50%;" />

<center>Mesh ball with fully-baked lighting.</center>

#### [7.1、Light Probes](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#7.1)

我们通过调用一个变体<font color="red">DrawMeshInstanced</font>方法来表明应该使用光探针，这个方法还需要五个参数。

首先是投影模式，我们希望它是开启的。之后是实例是否应该投射阴影，我们希望如此。接下来是图层，我们只需使用默认的零。然后，我们必须提供一个相机，让实例是可见的。

传递null意味着他们应该对所有的摄像机都进行渲染。最后我们可以设置探照灯模式。我们必须使用<font color="green">LightProbeUsage.CustomProvided</font>，因为没有一个位置可以用来混合探测。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class MeshBall : MonoBehaviour {
	
	…
	
	void Update () {
		if (block == null) {
			block = new MaterialPropertyBlock();
			block.SetVectorArray(baseColorId, baseColors);
			block.SetFloatArray(metallicId, metallic);
			block.SetFloatArray(smoothnessId, smoothness);
		}
		Graphics.DrawMeshInstanced(
			mesh, 0, material, matrices, 1023, block,
			ShadowCastingMode.On, true, 0, null, LightProbeUsage.CustomProvided
		);
	}
```

我们必须手动生成所有实例的<font color="red"> interpolated light probes </font>，并将它们添加到材料属性块中。

这意味着我们需要在<font color="green">configuring the block</font>的时候访问实例的位置。我们可以通过抓取它们的<font color="red">变换矩阵的最后一列</font>来获取它们，并将它们存储在一个临时数组中。

```c#
		if (block == null) {
			block = new MaterialPropertyBlock();
			block.SetVectorArray(baseColorId, baseColors);
			block.SetFloatArray(metallicId, metallic);
			block.SetFloatArray(smoothnessId, smoothness);

			var positions = new Vector3[1023];
			for (int i = 0; i < matrices.Length; i++) {
				positions[i] = matrices[i].GetColumn(3);
			}
		}
```

光探针必须通过一个<font color="red">SphericalHarmonicsL2</font>数组提供。它是通过调用<font color="green">LightProbes.CalculateInterpolatedLightAndOcclusionProbes</font>，以<font color="red">position and light probe arrays</font>为参数来填充的。还有第三个参数用于<font color="red"> occlusion</font>，我们将使用null。

```c#
for (int i = 0; i < matrices.Length; i++) {
				positions[i] = matrices[i].GetColumn(3);
			}
			var lightProbes = new SphericalHarmonicsL2[1023];
			LightProbes.CalculateInterpolatedLightAndOcclusionProbes(
				positions, lightProbes, null
			);
```

之后，我们可以通过<font color="red">CopySHCoefficientArraysFrom</font>将<font color="green"> light probes to the block</font>。

MaterialPropertyBlock .[CopySHCoefficientArraysFrom](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.CopySHCoefficientArraysFrom.html) 此函数将整个源数组转换并复制到 7 个 Vector4 属性数组中

```c#
	LightProbes.CalculateInterpolatedLightAndOcclusionProbes(
				positions, lightProbes, null
			);
			block.CopySHCoefficientArraysFrom(lightProbes);
```

#### [7.2、LPPV](https://catlikecoding.com/unity/tutorials/custom-srp/baked-light/#7.2)

另一种方法是使用LPPV。

这很有意义，因为实例都存在于一个狭小的空间里。这使我们不必计算和存储内插的光探针。另外，只要实例保持在体积内，就可以对其位置进行动态处理，而不必每一帧都提供新的光探测数据。

添加一个<font color="red">LightProbeProxyVolume</font>配置字段。

<font color="blue">如果它在使用中，那么就不要把光探针数据添加到块中</font>。然后把<font color="red">LightProbeUsage.UseProxyVolume</font>传给<font color="green">DrawMeshInstanced</font>而不是<font color="green">LightProbeUsage.CustomProvided</font>。我们总是可以提供体积作为一个额外的参数，即使它是空的，不被使用。

```c#
[SerializeField]
	LightProbeProxyVolume lightProbeVolume = null;
	
	…

	void Update () {
		if (block == null) {
			…

			if (!lightProbeVolume) {
				var positions = new Vector3[1023];
				…
				block.CopySHCoefficientArraysFrom(lightProbes);
			}
		}
		Graphics.DrawMeshInstanced(
			mesh, 0, material, matrices, 1023, block,
			ShadowCastingMode.On, true, 0, null,
			lightProbeVolume ?
				LightProbeUsage.UseProxyVolume : LightProbeUsage.CustomProvided,
			lightProbeVolume
		);
	}
```

你可以在网格球上添加一个LPPV组件，或者把它放在其他地方。自定义边界模式可以用来定义体积所占据的世界空间区域。

<img src="E:\Typora file\SRP\assets\lppv-inspector-1677332697417-24.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\lppv-scene-1677332697418-26.png" alt="scene" style="zoom:50%;" />

<center>Using an LPPV.</center>