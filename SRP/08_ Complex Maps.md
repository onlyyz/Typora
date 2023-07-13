# 08_Complex Maps

- 创建类似电路的材料。
- 添加对 MODS 遮罩贴图的支持。
- 引入辅助详细地图。
- 执行切线空间法线映射。

![img](E:\Typora file\SRP\assets\tutorial-image-1677596081539-23.jpg)

<center>An artistic impression of circuitry.</center>

### 1、Circuitry Material

到目前为止，我们总是使用非常简单的材料来测试我们的RP。但它也应该支持复杂的材料，这样我们才能表现出更有趣的表面。在本教程中，我们将在一些纹理的帮助下，创建一个类似电路的艺术材料。

#### 1.1、Albedo

我们材料的基础是其<font color="red">albedo map</font>。它由几层不同色调的绿色组成，上面是金色。每个颜色区域都是统一的，除了一些棕色的污点，这使得我们更容易分辨出后面要添加的细节。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/circuitry-material/circuitry-albedo.png" alt="img" style="zoom:50%;" />

<center>Albedo map.</center>

用我们的<font color="red">Lit shader</font>创建一个新材料，使用这个反照率贴图。我把它的平铺设置为2乘1，这样正方形纹理就可以包裹住一个球体而不会被拉得太长。默认球体的两极总是会发生很大的变形，这是无法避免的。

<img src="E:\Typora file\SRP\assets\albedo-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/circuitry-material/albedo-scene.png" alt="scene" style="zoom:50%;" />

<center>Circuitry sphere.</center>

#### 1.2、Emission

我们已经支持<font color="red">emission maps,</font>，所以让我们使用一个在金色电路之上添加浅蓝色照明图案的图。

<img src="E:\Typora file\SRP\assets\circuitry-emission.png" alt="img" style="zoom:50%;" />

<center>Emission map.</center>

将其分配给<font color="DarkOrchid "> material </font>，并将<font color="green">emission color </font>设置为白色，这样它就变得可见。

<img src="E:\Typora file\SRP\assets\emission-inspector.png" alt="inspector" style="zoom:50%;" />
<center><img src="E:\Typora file\SRP\assets\emission-scene-light.png" alt="scene light" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\emission-scene-dark.png" alt="scene dark" style="zoom:50%;" /></center>

<center>Emissive circuitry.</center>

### [2、Mask Map](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#2)

现在，我们不能做很多其他事情来使我们的材料更有趣。金色的电路应该是金属的，而绿色的电路板则不是，但我们目前只能配置统一的<font color="DarkOrchid ">metallic and smoothness</font>值。我们需要额外的<font color="green">maps</font>来支持整个表面的变化。

<img src="E:\Typora file\SRP\assets\metallic-smooth.png" alt="img" style="zoom:50%;" />

<center>Metallic 1 and smoothness 0.95.</center>

#### 2.1、MODS

我们可以为<font color="green">metallic</font>添加一个单独的贴图，为<font color="green">smoothness</font>添加另一个贴图，但这两个贴图都只需要一个通道，所以我们可以将它们合并在一个贴图中。这个贴图被称为<font color="red">mask map</font>，它的各个通道遮罩着不同的着色器属性。

我们将使用与Unity的HDRP相同的格式，也就是<font color="DarkOrchid ">MODS</font>贴图。这代表<font color="DarkOrchid ">Metallic, Occlusion, Detail, and Smoothness,</font>，按照这个顺序存储在RGBA通道中。

这里是我们的电路的地图。它的所有通道都有数据，但目前我们只使用它的R和A通道。由于这个纹理包含遮罩数据而不是颜色，所以要确保它的sRGB（Color Texture）纹理导入属性被禁用。不这样做会导致GPU在对纹理进行采样时错误地应用<font color="green">gamma-to-linear </font>转换。

<img src="E:\Typora file\SRP\assets\circuitry-mask-mods.png" alt="img" style="zoom:50%;" />

<center>Mask MODS map.</center>

为遮罩图添加一个属性，即Lit。因为它是一个遮罩，我们将使用白色作为默认值，这不会改变什么。

```glsl
	[NoScaleOffset] _MaskMap("Mask (MODS)", 2D) = "white" {}
		_Metallic ("Metallic", Range(0, 1)) = 0
		_Smoothness ("Smoothness", Range(0, 1)) = 0.5
```

<img src="E:\Typora file\SRP\assets\mask-inspector.png" alt="img" style="zoom:50%;" />

<center>Mask shader property.</center>

#### 2.2、Mask Input

给<font color="red">LitInput</font>增加一个<font color="green">GetMask</font>函数，简单地对<font color="DarkOrchid ">mask texture</font>进行采样并返回。

```glsl
TEXTURE2D(_BaseMap);
TEXTURE2D(_MaskMap);
…

float4 GetMask (float2 baseUV) {
	return SAMPLE_TEXTURE2D(_MaskMap, sampler_BaseMap, baseUV);
}
```

在我们继续之前，也让我们把<font color="red">LitInput</font>的代码整理一下。定义一个带有名称参数的<font color="blue">INPUT_PROP</font>宏，为使用<font color="RoyalBlue">UNITY_ACCESS_INSTANCED_PROP</font>宏提供一个速记方法。

```glsl
#define INPUT_PROP(name) UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, name)
```

现在我们可以简化所有<font color="red">getter</font>函数的代码了。我只展示了在<font color="green">GetBase</font>中检索<font color="RoyalBlue">_BaseMap_ST</font>的变化。

```C#
	float4 baseST = INPUT_PROP(_BaseMap_ST);
```

这一变化也可以应用于UnlitInput中的代码。

#### 2.3、[Metallic](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#2.3)

<font color="green">LitPass</font>不应该需要知道某些属性是否依赖<font color="green">mask map</font>。单独的函数可以在需要时检索<font color="green">mask map</font>。在<font color="red">GetMetallic</font>中这样做，通过乘法将其结果与掩码的R通道进行<font color="DarkOrchid ">Mask</font>。

```glsl
float GetMetallic (float2 baseUV) {
	float metallic = INPUT_PROP(_Metallic);
	metallic *= GetMask(baseUV).r;
	return metallic;
}
```

<img src="E:\Typora file\SRP\assets\metallic-scene.png" alt="img" style="zoom:50%;" />

<center>Only golden circuitry is metallic.</center>

金属图通常大多是二进制的。在我们的案例中，金色的电路是完全金属化的，而绿色的电路板则不是。金色的染色区域是个例外，其金属性稍差。

#### 2.4、[Smoothness](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#2.4)

在 <font color="red">GetSmoothness </font>中做同样的事情，这次依赖于<font color="DarkOrchid ">Mask</font>的 A 通道。 金色电路非常平滑，而绿色电路板则不然。

```glsl
float GetSmoothness (float2 baseUV) {
	float smoothness = INPUT_PROP(_Smoothness);
	smoothness *= GetMask(baseUV).a;
	return smoothness;
}
```

<img src="E:\Typora file\SRP\assets\smoothness-scene.png" alt="img" style="zoom:50%;" />

<center>Smoothness map in use.</center>

#### 2.5、Occlusion

遮罩的G通道包含<font color="green">occlusion data</font>。我们的想法是，像缝隙和洞这样的小的后退区域大多被物体的其他部分所遮挡，但如果这些特征是由纹理表示的，那么就会被灯光所忽略。

缺少的<font color="red">occlusion data</font>是由<font color="DarkOrchid ">Mask</font>提供的。添加一个新的<font color="RoyalBlue">GetOcclusion</font>函数来获取它，最初总是返回0，以展示其最大效果。

```glsl
float GetOcclusion (float2 baseUV) {
	return 0.0;
}
```

Add the occlusion data to the **Surface** struct.

```glsl
struct Surface {
	…
	float occlusion;
	float smoothness;
	float fresnelStrength;
	float dither;
};
```

并在<font color="green">LitPassFragment</font>中初始化它。

```glsl
	surface.metallic = GetMetallic(input.baseUV);
	surface.occlusion = GetOcclusion(input.baseUV);
	surface.smoothness = GetSmoothness(input.baseUV);
```

这个想法是，<font color="green">occlusion</font>只适用于<font color="DarkOrchid ">indirect environmental lighting - 间接环境照明</font>。直接光不受影响，所以当光源直接对准它们时，间隙不会保持黑暗。因此，我们只用<font color="green"> occlusion</font>来调节<font color="red">IndirectBRDF</font>的结果。

```glsl
float3 IndirectBRDF (
	Surface surface, BRDF brdf, float3 diffuse, float3 specular
) {
	…
	
    return (diffuse * brdf.diffuse + reflection) * surface.occlusion;
}
```

<img src="E:\Typora file\SRP\assets\fully-occluded.png" alt="img" style="zoom:50%;" />

<center>Fully occluded.</center>

在验证了它的作用后，让<font color="red">GetOcclusion</font>返回<font color="green">Mask</font>的G通道。

```glsl
float GetOcclusion (float2 baseUV) {
	return GetMask(baseUV).g;
}
```

<img src="E:\Typora file\SRP\assets\occlusion-scene.png" alt="img" style="zoom:50%;" />

<center>Occlusion map in use.</center>

绿板的某些部分比其他部分低，因此它们应该被遮挡一下。这些区域很大，<font color="green">occlusion map</font>处于最大强度以使效果清晰可见，但结果是太强了，没有意义。与其用更好的<font color="green">occlusion data</font>创建另一个<font color="DarkOrchid ">occlusion map</font>，不如给我们的着色器添加一个<font color="RoyalBlue">occlusion strength</font>滑块属性。

```glsl
		[NoScaleOffset] _MaskMap("Mask (MODS)", 2D) = "white" {}
		_Metallic ("Metallic", Range(0, 1)) = 0
		_Occlusion ("Occlusion", Range(0, 1)) = 1
		_Smoothness ("Smoothness", Range(0, 1)) = 0.5
```

<img src="E:\Typora file\SRP\assets\occlusion-inspector.png" alt="img" style="zoom:50%;" />

<center>Occlusion slider; reduced to 0.5.</center>

把它添加到UnityPerMaterial缓冲区。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _Occlusion)
	UNITY_DEFINE_INSTANCED_PROP(float, _Smoothness)
	UNITY_DEFINE_INSTANCED_PROP(float, _Fresnel)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

然后调整<font color="green">GetOcclusion</font>，让它用属性来调节遮罩。在这种情况下，滑块控制遮罩的强度，所以如果它被设置为零，遮罩应该被完全忽略。我们可以根据强度在遮罩和1之间插值来实现这一点。

```glsl
float GetOcclusion (float2 baseUV) {
	float strength = INPUT_PROP(_Occlusion);
	float occlusion = GetMask(baseUV).g;
	occlusion = lerp(occlusion, 1.0, strength);
	return occlusion;
}
```

<img src="E:\Typora file\SRP\assets\occlusion-half-strength.png" alt="img" style="zoom:50%;" />

<center>Occlusion at half strength.</center>

### [3、Detail Map](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#3)

下一步是为我们的<font color="green">material.</font>添加一些细节。我们通过对一个比基础贴图更高的细节纹理进行采样，并将其与基础和<font color="green"> mask data</font>相结合。这使得表面更加有趣，并且在近距离观察表面时提供了更高的分辨率信息，而基础贴图本身就会显得像素化。

<font color="DarkOrchid ">Details </font>应该只对表面的属性做一点修改，所以我们再一次将数据结合在一个<font color="RoyalBlue"> non-color map</font>中。<font color="red">HDRP</font>使用<font color="green">ANySNx</font>格式，这意味着它用R存储<font color="DarkOrchid "> albedo modulation - 反照率调制</font>，用B存储<font color="DarkOrchid ">smoothness modulation - 平滑度调制</font>，用AG存储细节<font color="blue">detail normal vector's XY</font>。但是我们的贴图不包含法线矢量，所以我们只使用RB通道。因此它是一个RGB纹理，而不是RGBA。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/circuitry-detail.png" alt="img" style="zoom:50%;" />

<center>Detail map; not sRGB.</center>

#### 3.1、Detail UV coordinates

因为细节图应该使用比<font color="green">detail map</font>更高的平铺，它需要自己的平铺和偏移。为它添加一个材质属性，这次没有<font color="green">NoScaleOffset</font>属性。它的默认值应该不会引起任何变化，我们通过使用线性灰色来获得，因为<font color="red">0.5</font>的值将被视为中性。

```glsl
		[NoScaleOffset] _EmissionMap("Emission", 2D) = "white" {}
		[HDR] _EmissionColor("Emission", Color) = (0.0, 0.0, 0.0, 0.0)

		_DetailMap("Details", 2D) = "linearGrey" {}
```

<img src="E:\Typora file\SRP\assets\detail-map-inspector.png" alt="img" style="zoom:50%;" />

<center>Detail map property; tiling four times as much.</center> 

在<font color="green">LitInput</font>中添加所需的纹理、采样器状态和<font color="DarkOrchid ">scale-offset</font>属性，同时添加<font color="red">TransformDetailUV</font>函数来转换<font color="RoyalBlue">detail texture coordinates.</font>。

```c#
TEXTURE2D(_DetailMap);
SAMPLER(sampler_DetailMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _DetailMap_ST)
	…
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float2 TransformDetailUV (float2 detailUV) {
	float4 detailST = INPUT_PROP(_DetailMap_ST);
	return detailUV * detailST.xy + detailST.zw;
}
```

然后添加一个<font color="red">GetDetail</font>函数来检索所有<font color="green">detail data</font>，给定<font color="RoyalBlue"> detail UV.</font>

```c#
float4 GetDetail (float2 detailUV) {
	float4 map = SAMPLE_TEXTURE2D(_DetailMap, sampler_DetailMap, detailUV);
	return map;
}
```

对<font color="green">LitPassVertex</font>中的坐标进行转换，并通过<font color="green">Varyings</font>将它们传递出去。

```glsl
struct Varyings {
	…
	float2 baseUV : VAR_BASE_UV;
	float2 detailUV : VAR_DETAIL_UV;
	…
};

Varyings LitPassVertex (Attributes input) {
	…
	output.baseUV = TransformBaseUV(input.baseUV);
	output.detailUV = TransformDetailUV(input.baseUV);
	return output;
}
```

#### 3.2、Detailed Albedo

为了给<font color="green"> albedo</font>添加细节，我们必须给<font color="green">GetBase</font>添加一个<font color="RoyalBlue">detail UV</font>的参数，我们将默认设置为0，这样现有的代码就不会中断。开始时，在计入颜色色调之前，简单地将所有细节直接添加到基础地图中。

```glsl
float4 GetBase (float2 baseUV, float2 detailUV = 0.0) {
	float4 map = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, baseUV);
	float4 color = INPUT_PROP(_BaseColor);
	
	float4 detail = GetDetail(detailUV);
	map += detail;
	
	return map * color;
}
```

Then pass the <font color="DarkOrchid ">detail UV</font> to it in<font color="green"> LitPassFragment.</font>

```c#
	float4 base = GetBase(input.baseUV, input.detailUV);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/albedo-added.png" alt="img" style="zoom:50%;" />

<center>Albedo details added.</center>

这证实了<font color="DarkOrchid ">detail data</font>得到了正确的采样，但我们还没有正确解释它。首先，0.5的值是中性的。较高的值应该增加或变亮，而较低的值应该减少或变暗。要使其发挥作用，第一步是在<font color="red">GetDetail</font>中将细节值范围从<font color="green">（0-1）</font>转换为<font color="red">（-1 - 1）</font>。

```c#
float4 GetDetail (float2 detailUV) {
	float4 map = SAMPLE_TEXTURE2D(_DetailMap, sampler_DetailMap, detailUV);
	return map * 2.0 - 1.0;
}
```

第二，只有R通道会影响<font color="green">albedo</font>，将其推向黑色或白色。这可以通过用0或1插值颜色来完成，这取决于细节的符号。然后，内插器就是绝对的<font color="green">detail value.</font>。这应该只影响<font color="red">albedo - 反照率</font>，而不是 <font color="red">base'salpha通道</font>。

```c#
	float detail = GetDetail(detailUV).r;
	//map += detail;
	map.rgb = lerp(map.rgb, detail < 0.0 ? 0.0 : 1.0, abs(detail));
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/albedo-interpolated.png" alt="img" style="zoom:50%;" />

<center>Albedo interpolated.</center>

这很有效，而且非常明显，因为我们的<font color="red"detail map</font>非常有力。但是变亮的效果似乎比变暗的效果更强。这是因为我们是在<font color="green"> linear space.</font>中应用修改。在<font color="RoyalBlue"> gamma space would </font>中进行修改会更好地匹配视觉上的平等分布。我们可以通过<font color="red">interpolating the square root of the albedo - 内插反照率的平方根</font>，并在之后进行平方处理来近似于此。1

```glsl
	map.rgb = lerp(sqrt(map.rgb), detail < 0.0 ? 0.0 : 1.0, abs(detail));
	map.rgb *= map.rgb;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/albedo-perceptual.png" alt="img" style="zoom:50%;" />

<center>Perceptual interpolation;darkening appears stronger. </center>

目前，细节被应用于整个表面，但我们的想法是，大部分的黄金电路是不受影响的。这就是<font color="red">detail mask</font>的作用，存储在<font color="green">mask map</font>的B通道中。我们可以通过把它纳入插值器来应用它。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/albedo-detail-masked.png" alt="img" style="zoom:50%;" />

<center>Masked details.</center>

我们的细节处于可能的最大强度，这太强了。让我们引入一个<font color="red">detail albedo strength</font>滑块属性来缩小它们。

```glsl
		_DetailMap("Details", 2D) = "linearGrey" {}
		_DetailAlbedo("Detail Albedo", Range(0, 1)) = 1
```

把它添加到<font color="red">UnityPerMaterial</font>中，并与<font color="green">GetBase</font>中的细节相乘。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _DetailAlbedo)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float4 GetBase (float2 baseUV, float2 detailUV = 0.0) {
	…
	float detail = GetDetail(detailUV).r * INPUT_PROP(_DetailAlbedo);
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/detail-albedo-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\detail-albedo-scene.png" alt="scene" style="zoom:50%;" />

<center>Albedo details scaled down to 0.2.</center>

#### 3.3、Detailed Smoothness

为<font color="green">smoothness</font>添加细节的方法是一样的。首先，为它也添加一个强度滑块属性。

```glsl
	_DetailAlbedo("Detail Albedo", Range(0, 1)) = 1
		_DetailSmoothness("Detail Smoothness", Range(0, 1)) = 1
```

然后将该属性添加到<font color="RoyalBlue">UnityPerMaterial</font>中，在<font color="red">GetSmoothness</font>中检索缩放后的细节，并以同样的方式进行插值。这一次我们需要<font color="green">detail map.</font>的B通道。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _DetailSmoothness)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float GetSmoothness (float2 baseUV, float2 detailUV = 0.0) {
	float smoothness = INPUT_PROP(_Smoothness);
	smoothness *= GetMask(baseUV).a;

	float detail = GetDetail(detailUV).b * INPUT_PROP(_DetailSmoothness);
	float mask = GetMask(baseUV).b;
	smoothness = lerp(smoothness, detail < 0.0 ? 0.0 : 1.0, abs(detail) * mask);
	
	return smoothness;
}
```

让<font color="green">LitPassFragment</font>把<font color="RoyalBlue">detail UV</font>也传给<font color="red">GetSmoothness。</font>

```glsl
surface.smoothness = GetSmoothness(input.baseUV, input.detailUV);
```

<img src="E:\Typora file\SRP\assets\detail-smoothness-inspector.png" alt="inspector" style="zoom:50%;" />

<center><img src="E:\Typora file\SRP\assets\detail-smoothness-scene-02.png" alt="scene 0.2" style="zoom:50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/detail-smoothness-scene-10.png" alt="scene 1.0" style="zoom: 50%;" /></center>

<center>*Smoothness details; at 0.2 and full strength.*</center>

#### 3.4、Fading Details

细节只有在它们足够大的时候才重要，在视觉上。细节不应该在它们太小的时候应用，因为那会产生一个嘈杂的结果。Mip映射像往常一样模糊了数据，但对于细节，我们想更进一步，把它们也淡化掉。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/not-faded.png" alt="img" style="zoom:50%;" />

<center>Details at full strength.</center>

如果我们启用了<font color="DarkOrchid ">detail texture</font>的<font color="green">Fadeout Mip Maps</font>导入选项，Unity可以为我们自动淡化细节。

一个范围滑块会显示出来，它可以控制在哪个mip级别开始和结束淡化。Unity简单地将Mip贴图插值为灰色，这意味着贴图变得中性。为了使其发挥作用，纹理的过滤模式必须设置为三线，这应该是自动发生的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/faded-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/detail-map/faded-scene.png" alt="scene" style="zoom:50%;" />

<center>*Fading details.*</center>

### [4、Normal Maps](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#4)

尽管我们把我们的表面做得很复杂，但它仍然显得很平坦，因为它就是这样。

光照与表面法线相互作用，法线在每个三角形中都被平滑地插值。如果光照也能与它的小特征相互作用，我们的表面就会更加可信。我们可以通过增加对<font color="green">normal maps.</font>的支持来做到这一点。

通常情况下，法线贴图是由一个高多边形密度的三维模型生成的，它被烘焙成一个低多边形模型，以供实时使用。高多边形几何体的法线向量如果丢失了，就会在法线贴图中被烘烤出来。或者，法线图是按程序生成的。下面是我们的电路的法线贴图。导入后将其纹理类型设置为法线贴图。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/circuitry-normal.png" alt="img" style="zoom:50%;" />

<center>Normal map.</center>

这个<font color="green"> map</font>遵循标准的切线空间法线图惯例，将上轴--在这里指定为Z--存储在B通道中，而右边和前边的XY轴则存储在RG中。就像细节图一样，法线分量的-1-1范围被转换，所以0.5是中点。因此，平坦的区域显得偏蓝。

#### 4.1、Sampling Normals

为了对法线进行采样，我们必须给我们的着色器添加一个法线图纹理属性，默认为<font color="green">*bump*</font>，代表一个平面图。还要添加一个法线比例属性，这样我们就可以控制地图的强度。

```glsl
	[NoScaleOffset] _NormalMap("Normals", 2D) = "bump" {}
	_NormalScale("Normal Scale", Range(0, 1)) = 1
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/normals-inspector.png" alt="img" style="zoom:50%;" />

<center>Normal map and scale.</center>

存储法线信息的最直接的方法是如上所述--RGB通道中的XYZ，但这并不是最有效的方法。如果我们假设法线向量总是指向上方而不是下方，我们就可以省略向上的分量，而从其他两个分量中得到它。然后，这些通道可以被存储在压缩的纹理格式中，这样可以使精度损失降到最低。

XY被存储在RG或AG中，这取决于纹理格式。这将改变纹理的外观，但Unity编辑器只显示<font color="green">original map.</font>的预览和缩略图。

法线图是否被改变取决于目标平台。如果贴图没有被改变，那么<font color="red">UNITY_NO_DXT5nm</font>就被定义了。

如果是这样，我们可以使用<font color="RoyalBlue">UnpackNormalRGB</font>函数来转换采样的法线数据，否则我们可以使用<font color="blue">UnpackNormalmapRGorAG</font>。

这两个函数都有<font color="red">sample and a scale </font>参数，都是在<font color="green"> *Core RP Library*.</font>的打包文件中定义的。在<font color="DarkOrchid ">Common</font>中添加一个函数，使用这些函数对法线数据进行解码。

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Packing.hlsl"

…

float3 DecodeNormal (float4 sample, float scale) {
	#if defined(UNITY_NO_DXT5nm)
	    return UnpackNormalRGB(sample, scale);
	#else
	    return UnpackNormalmapRGorAG(sample, scale);
	#endif
}
```

现在将<font color="DarkOrchid "> normal map, normal scale</font>和一个<font color="red">GetNormalTS</font>函数添加到<font color="RoyalBlue">LitInpu</font>中，并检索和解码法线向量。

```glsl
TEXTURE2D(_NormalMap);
…

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _NormalScale)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float3 GetNormalTS (float2 baseUV) {
	float4 map = SAMPLE_TEXTURE2D(_NormalMap, sampler_BaseMap, baseUV);
	float scale = INPUT_PROP(_NormalScale);
	float3 normal = DecodeNormal(map, scale);
	return normal;
}
```

#### [4.2、Tangent Space](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#4.2)

由于纹理环绕着几何体，它们在物体和世界空间的方向并不统一。

因此，存储法线的空间会出现曲线，以配合几何体的表面。唯一不变的是，这个空间是与表面相切的，这就是为什么它被称为切线空间。这个空间的Y上轴与表面法线相匹配。除此之外，它必须有一个与表面相切的X右轴。如果我们有了这两个，我们就可以从中产生Z轴。

因为切线空间的X轴并不是恒定的，它必须被定义为网格顶点数据的一部分。它被存储为一个四分量的切线向量。它的XYZ分量定义物体空间的轴。它的W分量是-1或1，用来控制Z轴指向的方向。这用于翻转具有<font color="red">bilateral symmetry 双边对称性</font>的网格的法线贴图--大多数动物都有这种情况--因此同一贴图可以用于网格的两边，将所需的纹理大小减半。

因此，如果我们有<font color="green">world-space normal and tangent</font>，我们可以构建一个从切线到世界空间的转换矩阵。我们可以使用现有的<font color="green">CreateTangentToWorld</font>函数，将法线、切线XYZ和切线W作为参数传递给它。

然后我们可以用<font color="red">tangent-space normal and conversion matrix </font>作为参数调用<font color="RoyalBlue">TransformTangentToWorld</font>。在<font color="green">Common</font>中添加一个能完成所有这些工作的函数。

```c#
float3 NormalTangentToWorld (float3 normalTS, float3 normalWS, float4 tangentWS) {
	float3x3 tangentToWorld =
		CreateTangentToWorld(normalWS, tangentWS.xyz, tangentWS.w);
	return TransformTangentToWorld(normalTS, tangentToWorld);
}
```

接下来，在<font color="green">LitPass</font>中，将具有<font color="RoyalBlue">TANGENT</font>语义的物体空间切线向量添加到<font color="red">Attributes</font>中，将世界空间切线添加到<font color="DarkOrchid ">Varyings</font>。

```glsl
struct Attributes {
	float3 positionOS : POSITION;
	float3 normalOS : NORMAL;
	float4 tangentOS : TANGENT;
	…
};

struct Varyings {
	float4 positionCS : SV_POSITION;
	float3 positionWS : VAR_POSITION;
	float3 normalWS : VAR_NORMAL;
	float4 tangentWS : VAR_TANGENT;
	…
};
```

切线矢量的XYZ部分可以通过调用<font color="DarkOrchid ">TransformObjectToWorldDir</font>转换为<font color="green">LitPassVertex</font>的世界空间。

```glsl
	output.normalWS = TransformObjectToWorldNormal(input.normalOS);
	output.tangentWS =
		float4(TransformObjectToWorldDir(input.tangentOS.xyz), input.tangentOS.w);
```

最后，我们通过调用<font color="red">LitPassFragment</font>中的<font color="green">NormalTangentToWorld</font>来获得最终的映射法线。

```glsl
surface.normal = NormalTangentToWorld(
		GetNormalTS(input.baseUV), input.normalWS, input.tangentWS
	);
```

![img](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/normal-mapped.png)

<center>Normal-mapped sphere.</center>

#### 4.3、[Interpolated Normal for Shadow Bias](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#4.3)

<font color="red">Perturbing the normal - 扰动法线</font>向量对于照亮表面是合适的，但我们也使用片段法线来<font color="green"> bias shadow sampling</font>。我们应该使用原来的表面法线来做这个。所以在<font color="DarkOrchid ">Surface</font>中为它添加一个字段。

```glsl
struct Surface {
	float3 position;
	float3 normal;
	float3 interpolatedNormal;
	…
};
```

在<font color="green">LitPassFragment</font>中分配法线向量。

在这种情况下，我们通常可以跳过向量的归一化，因为大多数网格的顶点法线在每个三角形中的曲线不会太多，以至于会对<font color="green">shadow biasing</font>产生负面影响。

```glsl
	surface.normal = NormalTangentToWorld(
		GetNormalTS(input.baseUV), input.normalWS, input.tangentWS
	);
	surface.interpolatedNormal = input.normalWS;
```

然后在<font color="green">GetCascadedShadow</font>中使用这个向量。

```
float GetCascadedShadow (
	DirectionalShadowData directional, ShadowData global, Surface surfaceWS
) {
	float3 normalBias = surfaceWS.interpolatedNormal *
		(directional.normalBias * _CascadeData[global.cascadeIndex].y);
	…
	if (global.cascadeBlend < 1.0) {
		normalBias = surfaceWS.interpolatedNormal *
			(directional.normalBias * _CascadeData[global.cascadeIndex + 1].y);
		…
	}
	return shadow;
}
```

#### 4.4、[Detailed Normals](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#4.4)

我们也可以包括一个细节的法线图。虽然<font color="RoyalBlue">HDRP</font>将<font color="red"> detail normal with the albedo and smoothness in a single map - 细节法线与反照率和平滑度结合在一张贴图</font>中，但我们将使用一个单独的纹理。将导入的纹理变成法线贴图，并启用<font color="green">*Fadeout Mip Maps* </font>，使其像其他细节一样淡出。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/circuitry-detail-normal.png" alt="img" style="zoom:50%;" />

<center>Detail normal map.</center>

Add shader properties for the map and again for the normal scale.

```glsl
	_DetailMap("Details", 2D) = "linearGrey" {}
		[NoScaleOffset] _DetailNormalMap("Detail Normals", 2D) = "bump" {}
		_DetailAlbedo("Detail Albedo", Range(0, 1)) = 1
		_DetailSmoothness("Detail Smoothness", Range(0, 1)) = 1
		_DetailNormalScale("Detail Normal Scale", Range(0, 1)) = 1
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/detail-normal-inspector.png" alt="img" style="zoom:50%;" />

<center>Detail normal properties; set to half strength.</center>

通过添加<font color="RoyalBlue">detail UV</font>参数调整<font color="green">GetNormalTS</font>，并对<font color="DarkOrchid ">detail map</font>进行采样。在这种情况下，我们可以通过将其分解为<font color="green">detail normal strength</font>来应用<font color="RoyalBlue">Mask</font>。

之后，我们必须将两个法线结合起来，我们可以通过调用<font color="red">BlendNormalRNM</font>与<font color="green">original and detail normals</font>来实现。这个函数将<font color="DarkOrchid ">rotates the detail normal around the base normal. - 细节法线围绕基本法线旋转</font>。

```glsl
float3 GetNormalTS (float2 baseUV, float2 detailUV = 0.0) {
	float4 map = SAMPLE_TEXTURE2D(_NormalMap, sampler_BaseMap, baseUV);
	float scale = INPUT_PROP(_NormalScale);
	float3 normal = DecodeNormal(map, scale);

	map = SAMPLE_TEXTURE2D(_DetailNormalMap, sampler_DetailMap, detailUV);
	scale = INPUT_PROP(_DetailNormalScale) * GetMask(baseUV).b;
	float3 detail = DecodeNormal(map, scale);
	normal = BlendNormalRNM(normal, detail);

	return normal;
}
```

最后，将<font color="red">detail UV</font>递给<font color="green">GetNormalTS。</font>

```glsl
surface.normal = NormalTangentToWorld(
		GetNormalTS(input.baseUV, input.detailUV), input.normalWS, input.tangentWS
	);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/normal-maps/detail-normal-mapped.png" alt="img" style="zoom:50%;" />

<center>Detailed normals.</center>

### 5、Optional Maps

并非每个材质都需要我们目前支持的所有贴图。不指定一个贴图意味着结果不会被改变，但着色器仍然使用默认的纹理来做所有的工作。我们可以通过添加一些着色器功能来控制着色器使用哪些贴图，从而避免不需要的工作。

Unity的着色器是根据编辑器中分配的贴图自动进行的，但我们将通过明确的切换来控制这个。

#### 5.1、[Normal Maps](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#5.1)

我们从法线图开始，这是<font color="green"> expensive features</font>。添加一个切换着色器属性，链接到一个适当的关键字。

```glsl
[Toggle(_NORMAL_MAP)] _NormalMapToggle ("Normal Map", Float) = 0
		[NoScaleOffset] _NormalMap("Normals", 2D) = "bump" {}
		_NormalScale("Normal Scale", Range(0, 1)) = 1
```

<img src="E:\Typora file\SRP\assets\optional-normal-maps.png" alt="img" style="zoom:50%;" />

<center>*Optional normal maps enabled.*</center>

只在<font color="green">CustomLit</font>通道上添加一个匹配的<font color="red">shader feature pragma</font>。其他通道都不需要映射法线，所以不应该得到这个特性。

```glsl
			#pragma shader_feature _NORMAL_MAP
```

在<font color="green">LitPassFragment</font>中，根据关键词，要么使用切线空间法线，要么直接将插值法线归一化。而在后一种情况下，我们还不如使用插值法线的归一化版本。

```glsl
	#if defined(_NORMAL_MAP)
		surface.normal = NormalTangentToWorld(
			GetNormalTS(input.baseUV, input.detailUV),
			input.normalWS, input.tangentWS
		);
		surface.interpolatedNormal = input.normalWS;
	#else
		surface.normal = normalize(input.normalWS);
		surface.interpolatedNormal = surface.normal;
	#endif
```

另外，如果可能的话，从<font color="green">Varyings</font>中省略切线矢量。我们不需要在<font color="red">Attributes</font>中省略它，因为如果我们不使用它，它会被自动忽略。

#### 5.2、[Input Config](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#5.2)

在这一点上，我们应该重新考虑我们如何将数据传递给<font color="red">LitInput</font>的<font color="green">getter</font>函数。

我们最终可能会出现使用或不使用多种东西的任何组合，我们必须以某种方式沟通。让我们通过引入一个<font color="red">InputConfig</font>结构来做到这一点，最初绑定<font color="RoyalBlue">base and detail UV</font>。同时创建一个方便的<font color="green">GetInputConfig</font>函数，返回一个给定的基础UV和可选的<font color="green">detail UV</font>的配置。

```glsl
struct InputConfig {
	float2 baseUV;
	float2 detailUV;
};

InputConfig GetInputConfig (float2 baseUV, float2 detailUV = 0.0) {
	InputConfig c;
	c.baseUV = baseUV;
	c.detailUV = detailUV;
	return c;
}
```

现在调整所有<font color="green">LitInput</font>函数，除了<font color="RoyalBlue">TransformBaseUV和TransformDetailUV</font>，以便它们有一个配置参数。我只展示了对<font color="red">GetBase</font>的修改。

```glsl
float4 GetBase (InputConfig c) {
	float4 map = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, c.baseUV);
	float4 color = INPUT_PROP(_BaseColor);
	
	float detail = GetDetail(c).r * INPUT_PROP(_DetailAlbedo);
	float mask = GetMask(c).b;
	…
}
```

然后调整<font color="green">LitPassFragment</font>，使其使用新的配置方法。

```glsl
InputConfig config = GetInputConfig(input.baseUV, input.detailUV);
	float4 base = GetBase(config);
	#if defined(_CLIPPING)
		clip(base.a - GetCutoff(config));
	#endif
	
	Surface surface;
	surface.position = input.positionWS;
	#if defined(_NORMAL_MAP)
		surface.normal = NormalTangentToWorld(
			GetNormalTS(config), input.normalWS, input.tangentWS
		);
	#else
		surface.normal = normalize(input.normalWS);
	#endif
	…
	surface.metallic = GetMetallic(config);
	surface.occlusion = GetOcclusion(config);
	surface.smoothness = GetSmoothness(config);
	surface.fresnelStrength = GetFresnel(config);
	…
	color += GetEmission(config);
```

调整其他通行证--MetaPass、ShadowCasterPass和UnlitPass--也使用新方法。这意味着我们也必须让UnlitPass使用新方法。

#### 5.3、[Optional Mask Map](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#5.3)

接下来，让我们在<font color="green">InputConfig</font>中为它添加一个布尔值，使<font color="RoyalBlue">mask map </font>成为可选项，默认设置为假。

```glsl
struct InputConfig {
	float2 baseUV;
	float2 detailUV;
	bool useMask;
};

InputConfig GetInputConfig (float2 baseUV, float2 detailUV = 0.0) {
	InputConfig c;
	c.baseUV = baseUV;
	c.detailUV = detailUV;
	c.useMask = false;
	return c;
}
```

我们可以通过在<font color="green">GetMask</font>中简单地返回1来避免对遮罩进行采样。这假定遮罩切换是恒定的，所以它不会导致着色器中的分支。

```glsl
float4 GetMask (InputConfig c) {
	if (c.useMask) {
		return SAMPLE_TEXTURE2D(_MaskMap, sampler_BaseMap, c.baseUV);
	}
	return 1.0;
}
```

Add a toggle for it to our shader.

```glsl
		[Toggle(_MASK_MAP)] _MaskMapToggle ("Mask Map", Float) = 0
		[NoScaleOffset] _MaskMap("Mask (MODS)", 2D) = "white" {}
```

连同<font color="green">CustomLit Pass</font>中的相关<font color="RoyalBlue">pragma</font>一起。

```glsl
			#pragma shader_feature _MASK_MAP
```

<img src="E:\Typora file\SRP\assets\optional-mask-map.png" alt="img" style="zoom:50%;" />

<center>Optional mask map.</center>

现在只在需要的时候打开<font color="green">LitPassFragment</font>中的<font color="RoyalBlue"> mask </font>。

```glsl
	InputConfig config = GetInputConfig(input.baseUV, input.detailUV);
	#if defined(_MASK_MAP)
		config.useMask = true;
	#endif
```

#### 5.4、[Optional Detail](https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/#5.4)

用同样的方法，为<font color="green">InputConfig</font>添加一个<font color="red">detail </font>切换，同样是默认禁用。

```glsl
struct InputConfig {
	…
	bool useDetail;
};

InputConfig GetInputConfig (float2 baseUV, float2 detailUV = 0.0) {
	…
	c.useDetail = false;
	return c;
}
```

只有在需要时才在<font color="green">GetDetail</font>中对<font color="red"> detail map</font>进行采样，否则返回0。

```glsl
float4 GetDetail (InputConfig c) {
	if (c.useDetail) {
		float4 map = SAMPLE_TEXTURE2D(_DetailMap, sampler_DetailMap, c.detailUV);
		return map * 2.0 - 1.0;
	}
	return 0.0;
}
```

这就避免了对<font color="green"> detail map</font>的采样，但纳入细节的情况仍然发生。为了阻止这种情况，也要跳过<font color="red">GetBase</font>中的相关代码。

```glsl
	if (c.useDetail) {
		float detail = GetDetail(c).r * INPUT_PROP(_DetailAlbedo);
		float mask = GetMask(c).b;
		map.rgb =
			lerp(sqrt(map.rgb), detail < 0.0 ? 0.0 : 1.0, abs(detail) * mask);
		map.rgb *= map.rgb;
	}
```

而在<font color="green">GetSmoothness。</font>

```glsl
	if (c.useDetail) {
		float detail = GetDetail(c).b * INPUT_PROP(_DetailSmoothness);
		float mask = GetMask(c).b;
		smoothness =
			lerp(smoothness, detail < 0.0 ? 0.0 : 1.0, abs(detail) * mask);
	}
```

而在<font color="green">GetNormalTS。</font>

```glsl
	if (c.useDetail) {
		map = SAMPLE_TEXTURE2D(_DetailNormalMap, sampler_DetailMap, c.detailUV);
		scale = INPUT_PROP(_DetailNormalScale) * GetMask(c).b;
		float3 detail = DecodeNormal(map, scale);
		normal = BlendNormalRNM(normal, detail);
	}
```

然后为着色器的<font color="green"> details</font>添加一个切换属性。

```glsl
	[Toggle(_DETAIL_MAP)] _DetailMapToggle ("Detail Maps", Float) = 0
		_DetailMap("Details", 2D) = "linearGrey" {}
```

在CustomLit中再次有一个伴随的着色器功能。

```glsl
#pragma shader_feature _DETAIL_MAP
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/complex-maps/optional-maps/optional-detail-maps.png" alt="img" style="zoom:50%;" />

<center>Optional details.</center>

现在我们只需要在定义了相关的关键词时，在<font color="green">Varyings</font>中包含<font color="red"> detail UV </font>

```c#
struct Varyings {
	…
	#if defined(_DETAIL_MAP)
		float2 detailUV : VAR_DETAIL_UV;
	#endif
	…
};

Varyings LitPassVertex (Attributes input) {
	…
	#if defined(_DETAIL_MAP)
		output.detailUV = TransformDetailUV(input.baseUV);
	#endif
	return output;
}
```

最后，只有在需要时才在<font color="green">LitPassFragment</font>中包括<font color="red"> details</font>

```glsl
InputConfig config = GetInputConfig(input.baseUV);
	#if defined(_MASK_MAP)
		config.useMask = true;
	#endif
	#if defined(_DETAIL_MAP)
		config.detailUV = input.detailUV;
		config.useDetail = true;
	#endif
```