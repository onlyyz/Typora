# 07_LOD and Reflections 

使用LOD组。
		在LOD级别之间交叉淡化。
		通过采样反射探针来反射环境。
		支持可选的菲涅尔反射。![img](E:\Typora file\SRP\assets\tutorial-image-1677522004469-1.jpg)

<center>A bunch of LOD groups and reflection probes.</center>

### 1、LOD Groups

许多小物体为场景添加细节并使其更有趣。 但是，太小而无法覆盖多个像素的细节会退化为模糊的噪声。 在这些视觉比例下，最好不要渲染它们，这也可以释放 CPU 和 GPU 来渲染更重要的东西。 

我们也可以决定更早地剔除这些对象，当它们仍然可以被区分时。 这进一步提高了性能，但会导致事物根据其视觉大小突然出现和消失。 我们还可以添加中间步骤，在最终完全剔除一个对象之前切换到连续的不太详细的可视化。 Unity 可以通过使用 LOD 组来完成所有这些事情。

#### 1.1、LOD Group Component

你可以通过创建一个空的游戏对象并向其添加一个<font color="green">LODGroup</font>组件来向场景添加一个细节级别组。默认的组定义了四个级别。

<font color="red">LOD 0, LOD 1, LOD 2, 最后是culled</font>，这意味着没有任何东西被渲染。百分比代表了相对于显示窗口尺寸的估计视觉尺寸的阈值。因此，LOD 0用于覆盖窗口60%以上的物体，通常考虑垂直尺寸，因为那是最小的。

<img src="E:\Typora file\SRP\assets\group-component.png" alt="img" style="zoom:50%;" />

<center>Default LOD group component.</center>

然而，<font color="DarkOrchid ">*Quality* project settings</font>部分包含一个<font color="green">LOD *Bias*</font>，它可以缩放这些阈值。

默认情况下，它被设置为2，这意味着它将该评估的估计视觉尺寸翻倍。因此，LOD 0最终被用于30%以上的一切，而不是只有60%。当偏差设置为1以外的时候，组件的检查器会显示一个警告。除此之外，还有一个最大LOD级别选项，可以用来限制最高LOD级别。例如，如果它被设置为1，那么LOD 1也会被使用，而不是LOD 0。

这个想法是，你让所有可视化<font color="green">LOD</font>级别的游戏对象都成为组对象的子对象。例如，我用三个相同大小的彩色球体来表示三个LOD级别。

<img src="E:\Typora file\SRP\assets\lod-group-sphere.png" alt="img" style="zoom:50%;" />

<center>LOD group containing three spheres.</center>

每个对象都必须被分配到适当的<font color="green">LOD</font>级别。你可以通过在组组件中选择一个级别块，然后将对象拖到它的渲染器列表中，或者直接将它放到一个<font color="green">LOD</font>级别块上。

<img src="E:\Typora file\SRP\assets\lod-renderers.png" alt="img" style="zoom:50%;" />

<center>Renderers for LOD 0.</center>

Unity会自动渲染相应的对象。在编辑器中选择一个特定的对象将覆盖这一行为，所以你可以在场景中看到你的选择。如果你选择了一个LOD组本身，编辑器也会显示当前可见的LOD级别。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/lod-groups/rendering-lod-spheres.png" alt="img" style="zoom:50%;" />

<center>Scene with LOD sphere prefab instances.</center>

移动摄像机会改变每组使用的LOD级别。另外，你也可以调整LOD偏差，以看到可视化的变化，而其他方面保持不变。

<center><iframe src="https://gfycat.com/ifr/imaginativedifficultdevilfish" style="border: none; overflow: hidden; width: 340px; height: calc(150% + 44px);"></iframe></center>

<center>Adjusting LOD bias.</center>

#### 1.2、Additive LOD Groups

对象可以被添加到一个以上的<font color="green">LOD</font>级别。

你可以利用这一点将较小的细节添加到较高的层次，而同样较大的物体则用于多个层次。

例如，我用堆叠的扁平立方体做了一个三层的金字塔。基层立方体是所有三个层次的一部分。中间的立方体是LOD 0和LOD 1的一部分，而最小的顶层立方体只是LOD 0的一部分。 因此，细节并根据视觉尺寸添加和删除到组中，而不是替换整个东西。

<img src="E:\Typora file\SRP\assets\stacked-cubes-lod.png" alt="img" style="zoom:50%;" />

<center>Stacked cubes LOD groups.</center>

#### 1.3、LOD Transitions

LOD级别的突然交换在视觉上可能会很刺眼，尤其是当一个物体由于自身或摄像机的轻微移动而最终快速来回切换时。通过将组的淡出模式设置为交叉淡出，可以使这种过渡变得渐进。这使得旧的层次淡出，而新的层次同时淡入。

<img src="E:\Typora file\SRP\assets\cross-fade-mode.png" alt="img" style="zoom:50%;" />

<center>Cross-fade mode.</center>

你可以控制每个LOD级别何时开始<font color="green">cross-fade</font>到下一个级别。这个选项在<font color="green">cross-fade 交叉渐变</font>被启用时是可见的。

渐变过渡宽度为0意味着在这个级别和下一个级别之间没有渐变，而数值为1意味着它立即开始渐变。在0.5时，在默认设置下，LOD 0会在80%时开始交叉淡化到LOD 1。

<img src="E:\Typora file\SRP\assets\fade-transition-width.png" alt="img" style="zoom:50%;" />

<center>Fade transition width.</center>

当交叉渐变被激活时，两个LOD级别会同时被渲染。这就需要着色器以某种方式来混合它们。Unity为<font color="green">LOD_FADE_CROSSFADE</font>关键字挑选了一个<font color="red"> shader variant </font>，所以在我们的<font color="green">Lit</font>着色器中为它添加了一个多编译指令。对<font color="red">CustomLit和ShadowCaster</font>通道都要这样做。

```glsl
			#pragma multi_compile _ LOD_FADE_CROSSFADE
```

一个物体被淡化的程度通过<font color="green">UnityPerDraw</font>缓冲区的<font color="RoyalBlue">unity_LODFade</font>向量来传达，我们已经定义了它。

它的X分量包含<font color="red">fade factor</font>。它的Y分量包含相同的因子，但被量化为16个步骤，我们不会使用它。让我们通过在<font color="red">LitPassFragment</font>的开始部分返回<font color="green">淡出系数</font>来直观地看到它是否被使用。

```glsl
float4 LitPassFragment (Varyings input) : SV_TARGET {
	UNITY_SETUP_INSTANCE_ID(input);
	#if defined(LOD_FADE_CROSSFADE)
		return unity_LODFade.x;
	#endif
	
	…
}
```

<img src="E:\Typora file\SRP\assets\fade-factor.png" alt="img" style="zoom:50%;" />

<center>LOD fade factor.</center>

逐渐消失的物体开始时的系数为1，然后减少到0，如预期。但是我们也看到了代表更高LOD级别的纯黑色物体。这是因为正在淡出的对象的淡出系数被否定了。我们可以通过返回被否定的淡出系数来看到这一点。

```glsl
	return -unity_LODFade.x;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/lod-groups/fade-factor-negated.png" alt="img" style="zoom:50%;" />

<center>Negated fade factor.</center>

注意，处于两个LOD级别的物体不会与自己交叉淡化。

#### 1.4、Dithering

为了混合这两个LOD级别，我们可以使用<font color="red">clipping</font>，应用一种类似于接近<font color="green">semitransparent shadows - 半透明阴影</font>的方法。

由于我们需要对两个表面和它们的阴影都这样做，让我们为<font color="DarkOrchid ">Common</font>添加一个<font color="red">ClipLOD</font>函数。给它提供<font color="RoyalBlue">clip-space XY coordinates</font>和<font color="RoyalBlue">fade factor</font>作为参数。然后--如果交叉渐变是有效的--根据<font color="red"> fade factor</font>减去<font color="red">dither</font>模式来剪辑。

```glsl
void ClipLOD (float2 positionCS, float fade) {
	#if defined(LOD_FADE_CROSSFADE)
		float dither = 0;
		clip(fade - dither);
	#endif
}
```

为了检查剪裁是否像预期的那样工作，我们将从一个每32像素重复的垂直渐变开始。这应该会产生交替的水平条纹。

```glsl
		float dither = (positionCS.y % 32) / 32;
```

在<font color="red">LitPassFragment</font>中调用<font color="green">ClipLOD</font>，而不是返回渐变系数。

```glsl
	//#if defined(LOD_FADE_CROSSFADE)
	//	return unity_LODFade.x;
	//#endif
	ClipLOD(input.positionCS.xy, unity_LODFade.x);
```

也可以在<font color="green">ShadowCasterPassFragment</font>开始时调用它来交叉淡化阴影。

```
void ShadowCasterPassFragment (Varyings input) {
	UNITY_SETUP_INSTANCE_ID(input);
	ClipLOD(input.positionCS.xy, unity_LODFade.x);

	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/lod-groups/striped-lod-half.png" alt="img" style="zoom:50%;" />

<center>LOD stripes, half.</center>

我们得到了条纹状的渲染，但是在交叉淡入时只有两个<font color="green">LOD</font>级别中的一个显示出来。这是因为其中一个有一个负的渐变系数。在这种情况下，我们通过添加而不是减去<font color="green">dither pattern </font>来解决这个问题。

```glsl
	clip(fade + (fade < 0.0 ? dither : -dither));
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/lod-groups/striped-lod-complete.png" alt="img" style="zoom:50%;" />

<center>LOD stripes, complete.</center>

现在它已经工作了，我们可以切换到一个适当的抖动模式。让我们选择与我们用于半透明阴影的相同的模式。

```glsl
	float dither = InterleavedGradientNoise(positionCS.xy, 0);
```

<img src="E:\Typora file\SRP\assets\dithered-lod.png" alt="img" style="zoom:50%;" />

<center>Dithered LOD.</center>

#### 1.5、Animated Cross-Fading

虽然抖动创造了一个相当平滑的过渡，但模式很明显。而且，就像半透明的阴影一样，褪色的阴影是不稳定的，会让人分心。理想情况下，交叉渐变只是暂时的，即使这样也没有其他变化。

我们可以通过启用LOD组的<font color="red">Animate Cross-fading</font>选项来使它变成这样。这就不考虑淡化过渡的宽度，而是一旦一个组通过了LOD的阈值，就迅速交叉淡化。

<img src="E:\Typora file\SRP\assets\animated-cross-fading.png" alt="inspector" style="zoom:50%;" />

<center><iframe src="https://gfycat.com/ifr/assuredmarrieddove" style="border: none; overflow: hidden; width: 340px; height: calc(100% + 44px);"></iframe></center>

<center>Animated cross-fading.</center>

默认的动画持续时间是半秒，可以通过设置静态的<font color="red">LODGroup.crossFadeAnimationDuration</font>属性来改变所有组。

### 2、Reflections

另一个为场景增加细节和真实感的现象是环境的<font color="red"> specular reflection </font>，镜子是最明显的例子，我们还不支持。

这对金属表面尤其重要，目前金属表面大多是黑色的。为了使这一点更加明显，我在 "烘焙之光 "场景中增加了一个新的具有不同<font color="green">color and smoothness </font>的金属球体。

<img src="E:\Typora file\SRP\assets\without-reflections.png" alt="img" style="zoom:50%;" />

<center>Scene without reflections.</center>

#### [2.1、Indirect BRDF](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.1)

我们已经支持漫反射全局光照，它取决于<font color="red">BRDF</font>的漫反射颜色。

现在我们也增加了<font color="red">镜面全局光照</font>，它也取决于<font color="green">BRDF</font>。因此，让我们为<font color="green">BRDF</font>添加一个<font color="RoyalBlue">IndirectBRDF</font>函数，使用<font color="green">surface and BRDF </font>参数，加上从全局光照中获得的漫反射和镜面色。最初让它只返回反射的漫反射光。

```glsl
float3 IndirectBRDF (
	Surface surface, BRDF brdf, float3 diffuse, float3 specular
) {
    return diffuse * brdf.diffuse;
}
```

添加镜面反射的开始是类似的：简单地包括<font color="DarkOrchid ">specular GI multiplied with the BRDF's specular color.</font>

```glsl
	float3 reflection = specular * brdf.specular;

    return diffuse * brdf.diffuse + reflection;
```

但是<font color="red">roughness </font>会分散这种反射，所以它应该减少我们最终看到的<font color="DarkOrchid ">specular reflection</font>。我们通过将其除以粗糙度的平方加一来做到这一点。

因此，低的粗糙度值并不重要，而最大的粗糙度会使反射减半。

```glsl
	float3 reflection = specular * brdf.specular;
	reflection /= brdf.roughness * brdf.roughness + 1.0;
```

在<font color="red">GetLighting</font>中调用<font color="RoyalBlue">IndirectBRDF</font>，而不是直接计算<font color="green">diffuse indirect light directly - 漫反射间接光.</font>。开始时，使用白色作为<font color="DarkOrchid ">specular GI color - 镜面GI的颜色。.</font>

```glsl
float3 GetLighting (Surface surfaceWS, BRDF brdf, GI gi) {
	ShadowData shadowData = GetShadowData(surfaceWS);
	shadowData.shadowMask = gi.shadowMask;
	
	float3 color = IndirectBRDF(surfaceWS, brdf, gi.diffuse, 1.0);
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		Light light = GetDirectionalLight(i, surfaceWS, shadowData);
		color += GetLighting(surfaceWS, brdf, light);
	}
	return color;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/reflecting-white-environment.png" alt="img" style="zoom:50%;" />

<center>Reflecting a white environment.</center>

一切都变得至少有点亮，因为我们增加了以前没有的照明。金属表面的变化是戏剧性的：它们的颜色现在明亮而明显。

#### [2.2、Sampling the Environment](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.2)

镜面反射反映的是环境，默认情况下是天空盒。通过<font color="red">unity_SpecCube0</font>，它可以作为一个立方体贴图纹理使用。在<font color="red">GI</font>中声明它和它的采样器状态，这次使用TEXTURECUBE宏。

```glsl
TEXTURECUBE(unity_SpecCube0);
SAMPLER(samplerunity_SpecCube0);
```

然后添加一个带有<font color="red">world-space surface parameter</font>的<font color="RoyalBlue">SampleEnvironment</font>函数，对纹理进行采样，并返回其<font color="green">RGB分量</font>。

我们通过<font color="RoyalBlue">SAMPLE_TEXTURECUBE_LOD</font>宏对<font color="RBlue"> cube map</font>进行采样，它以贴图、采样器状态、UVW坐标和<font color="red">mip level </font>为参数。

由于它是一个立方体贴图，我们需要3D纹理坐标，因此需要UVW。我们开始时总是使用最高的mip级别，所以我们对全分辨率的纹理进行采样。

```glsl
float3 SampleEnvironment (Surface surfaceWS) {
	float3 uvw = 0.0;
	float4 environment = SAMPLE_TEXTURECUBE_LOD(
		unity_SpecCube0, samplerunity_SpecCube0, uvw, 0.0
	);
	return environment.rgb;
}
```

对<font color="red">cube map</font>的取样是通过一个方向来完成的，在这个例子中，这个方向是<font color="green">相机到表面的视图方向</font>，由表面反射出来。我们通过调用<font color="green">reflect</font>函数，以负的视角方向和表面的法线作为参数来获得它。

```glsl
	float3 uvw = reflect(-surfaceWS.viewDirection, surfaceWS.normal);
```

接下来，在<font color="red">GI</font>中添加一个<font color="green">specular color</font>，并在<font color="green">GetGI</font>中存储其中的采样环境。

```glsl
struct GI {
	float3 diffuse;
	float3 specular;
	ShadowMask shadowMask;
};

…

GI GetGI (float2 lightMapUV, Surface surfaceWS) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	gi.specular = SampleEnvironment(surfaceWS);
	…
}
```

现在我们可以在<font color="red">GetLighting</font>中向<font color="green">IndirectBRDF</font>传递正确的颜色。

```glsl
	float3 color = IndirectBRDF(surfaceWS, brdf, gi.diffuse, gi.specular);
```

最后，为了让它工作，我们必须在<font color="green">CameraRenderer.DrawVisibleGeometry</font>中指示Unity在设置每个物体数据时包括<font color="red">reflection probe - 反射探头。</font>

```c#
	perObjectData =
				PerObjectData.ReflectionProbes |
				PerObjectData.Lightmaps | PerObjectData.ShadowMask |
				PerObjectData.LightProbe | PerObjectData.OcclusionProbe |
				PerObjectData.LightProbeProxyVolume |
				PerObjectData.OcclusionProbeProxyVolume
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/reflecting-environment.png" alt="img" style="zoom:50%;" />

<center>Reflecting the environment probe.</center>

表面现在反映了环境。这对金属表面来说是很明显的，但其他表面也会反射。因为这只是天空盒，没有其他东西被反射，但我们稍后会看一下。

<img src="E:\Typora file\SRP\assets\environment-probe.png" alt="img" style="zoom:50%;" />

<center>Environment probe.</center>

#### [2.3、Rough Reflections](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.3)

由于粗糙度分散了镜面反射，它不仅降低了其强度，而且还使其变得模糊，就像失焦一样。Unity通过在较低的mip级别存储模糊的环境贴图来接近这种效果。为了获得正确的mip级别，我们需要知道<font color="green">perceptual roughness - 感知粗糙度</font>，所以让我们把它添加到<font color="red">BRDF</font>结构中。

```glsl
struct BRDF {
	…
	float perceptualRoughness;
};

…

BRDF GetBRDF (Surface surface, bool applyAlphaToDiffuse = false) {
	…

	brdf.perceptualRoughness =
		PerceptualSmoothnessToPerceptualRoughness(surface.smoothness);
	brdf.roughness = PerceptualRoughnessToRoughness(brdf.perceptualRoughness);
	return brdf;
}
```

我们可以依靠<font color="red">PerceptualRoughnessToMipmapLevel</font>函数来计算给定的<font color="green">perceptual roughness</font>的正确<font color="RoyalBlue"> mip level</font>。它

被定义在<font color="DarkOrchid ">*Core RP Library*</font>的<font color="blue">ImageBasedLighting</font>文件中。这需要我们为<font color="red">SampleEnvironment</font>添加一个BRDF参数。

```glsl
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/EntityLighting.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/ImageBasedLighting.hlsl"

…

float3 SampleEnvironment (Surface surfaceWS, BRDF brdf) {
	float3 uvw = reflect(-surfaceWS.viewDirection, surfaceWS.normal);
	float mip = PerceptualRoughnessToMipmapLevel(brdf.perceptualRoughness);
	float4 environment = SAMPLE_TEXTURECUBE_LOD(
		unity_SpecCube0, samplerunity_SpecCube0, uvw, mip
	);
	return environment.rgb;
}
```

将所需的参数也添加到<font color="red">GetGI</font>中并通过它传递下去。

```glsl
GI GetGI (float2 lightMapUV, Surface surfaceWS, BRDF brdf) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	gi.specular = SampleEnvironment(surfaceWS, brdf);
	…
}
```

最后，在<font color="red">LitPassFragment</font>中提供它。

```glsl
	GI gi = GetGI(GI_FRAGMENT_DATA(input), surface, brdf);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/rough-reflections.png" alt="img" style="zoom:50%;" />

<center>Roughness blurs reflections.</center>

#### [2.4、Fresnel Reflection](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.4)

所有表面的一个特性是，当以<font color="green"> grazing angles </font>观察时，它们开始类似于完美的镜子，因为光线从它们身上反弹时大多不受影响。这种现象被称为<font color="red">Fresnel reflection</font>。

它实际上比这更复杂，因为它与光波在不同介质边界的<font color="DarkOrchid "> transmission and reflection - 透射和反射</font>有关，但我们只是简单地使用了与<font color="red">Universal RP</font>相同的近似，即假设空气-固体边界。

我们对菲涅尔使用了一个变体<font color="red">Schlick</font>近似法。在理想情况下，它用纯白色取代了<font color="blue">specular BRDF color </font>色，但<font color="red"> roughness </font>会阻止反射显示出来。

我们通过将表面<font color="RoyalBlue">smoothness and reflectivity </font>相加得出最终的颜色，最大值为1。由于它是灰度的，我们只需在<font color="red">BRDF</font>中加入一个值就足够了。

```glsl
struct BRDF {
	…
	float fresnel;
};

…

BRDF GetBRDF (Surface surface, bool applyAlphaToDiffuse = false) {
	…
	
	brdf.fresnel = saturate(surface.smoothness + 1.0 - oneMinusReflectivity);
	return brdf;
}
```

在<font color="red">IndirectBRDF</font>中，我们通过取表面法线和视线方向的点积，从1中减去，并将结果提高到四次方，来找到<font color="DarkOrchid "> Fresnel effect </font>的强度。我们可以在这里使用<font color="green"> *Core RP Library*</font>中方便的Pow4函数。

```
	float fresnelStrength =
		Pow4(1.0 - saturate(dot(surface.normal, surface.viewDirection)));
	float3 reflection = specular * brdf.specular;
```

然后我们在<font color="red"> <font color="blue">BRDF </font>specular and<font color="blue"> fresnel</font> color based </font>之间根据强度进行插值，然后用这个结果来影响环境反射。

```glsl
	float3 reflection =
		specular * lerp(brdf.specular, brdf.fresnel, fresnelStrength);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/fresnel-reflections.png" alt="img" style="zoom:50%;" />

<center>Fresnel reflections.</center>

#### [2.5、Fresnel Slider](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.5)

<font color="red">Fresnel reflections</font>主要沿着几何体的边缘添加反射。当环境贴图与物体后面的颜色正确匹配时，效果是微妙的，但如果不是这样的话，反射就会显得很奇怪，让人分心。沿着结构内球体边缘的明亮反射就是一个很好的例子。

降低<font color="green"> smoothness</font>可以摆脱菲涅尔反射，但也会使整个表面变暗。另外，在某些情况下，菲涅尔近似值并不合适，例如在水下。

因此，让我们添加一个滑块，将其缩减到<font color="red">Lit shader</font>。

```glsl
		_Metallic ("Metallic", Range(0, 1)) = 0
		_Smoothness ("Smoothness", Range(0, 1)) = 0.5
		_Fresnel ("Fresnel", Range(0, 1)) = 1
```

把它添加到<font color="red">LitInput</font>的<font color="green">UnityPerMaterial</font>缓冲区，并为它创建一个<font color="RoyalBlue">GetFresnel</font>函数。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	…
	UNITY_DEFINE_INSTANCED_PROP(float, _Fresnel)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

…

float GetFresnel (float2 baseUV) {
	return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Fresnel);
}
```

同时为它添加一个假函数到<font color="green">UnlitInput</font>，以保持它们的同步。

```glsl
float GetFresnel (float2 baseUV) {
	return 0.0;
}
```

<font color="green">**Surface**</font>现在得到一个关于其<font color="red">Fresnel strength.</font>的字段。

```glsl
struct Surface {
	…
	float smoothness;
	float fresnelStrength;
	float dither;
};
```

我们将其设置为等于<font color="red">LitPassFragment</font>中滑块的值。

```glsl
	surface.smoothness = GetSmoothness(input.baseUV);
	surface.fresnelStrength = GetFresnel(input.baseUV);
```

最后，用它来缩放我们在<font color="red">IndirectBRDF</font>中使用的<font color="RoyalBlue">Fresnel strength </font>。

```glsl
	float fresnelStrength = surface.fresnelStrength *
		Pow4(1.0 - saturate(dot(surface.normal, surface.viewDirection)));
```

<center><iframe src="https://gfycat.com/ifr/sinfullikablegiraffe" style="border: none; overflow: hidden; width: 360px; height: calc(100% + 44px);"></iframe></center>

<center>Adjusting Fresnel strength.</center>

#### [2.6、Reflection Probes](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.6)

默认的环境立方体地图只包含天空盒。为了反射场景中的其他东西，我们必须通过<font color="green">GameObject / Light / Reflection Probe</font>为它添加一个<font color="red">reflection probe - 反射探头。</font>

这些探测器从它们的位置将场景渲染成一个<font color="red">cube map</font>。

因此，只有在靠近探头的表面上，反射才会或多或少显得正确。所以，通常有必要在一个场景中放置多个探头。它们有重要性和盒子大小的属性，可以用来控制每个探针所影响的区域。

<img src="E:\Typora file\SRP\assets\reflection-probe-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/reflection-probe-map.png" alt="cube map" style="zoom:50%;" />

<center>Reflection probe inside structure.</center>

探针的类型默认设置为烘烤，这意味着它被渲染一次，立方体地图被存储在构建中。你也可以把它设置成实时的，这样可以使地图与动态场景保持同步。它就像其他相机一样被渲染，使用<font color="red">our RP,</font>，对立方体地图的六个面各渲染一次。

所以<font color="green">realtime reflection probes</font>是很昂贵的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/reflections/reflection-probes.png" alt="img" style="zoom:50%;" />

<center>Using three reflection probes.</center>

每个物体只使用一个<font color="green">environment probe</font>，但场景中可以有多个探测器。

因此，你可能不得不分割物体以获得可接受的反射。例如，用于建造结构的立方体最好分成独立的内部和外部部分，因此每个部分可以使用不同的反射探头。

另外，这意味着<font color="red"> GPU batching</font>会被反射探针打散。不幸的是，网状球根本无法使用反射探针，总是以天空盒子为结局。

<font color="green">MeshRenderer</font>组件有一个<font color="RoyalBlue">Anchor Override</font>，可以用来微调它们使用的探针，而不必担心盒子的大小和位置。

还有一个<font color="RoyalBlue"> *Reflection Probes* option - 反射探针选项</font>，默认设置为混合探针。这个想法是，Unity允许在最好的两个反射探头之间进行混合。

然而，这种模式与<font color="green">SRP batcher</font>程序不兼容，所以Unity的其他RP不支持它，我们也不会支持。

如果你感到好奇，如何混合探针在我的2018年SRP教程的反射教程中有所解释，但我预计一旦传统管道被移除，这一功能就会消失。我们将在未来研究其他反射技术。因此，唯一的两种功能模式是 "关闭 "和 "简单"，前者总是使用天空盒，后者挑选最重要的探测器。其他的功能与简单模式完全一样。

<img src="E:\Typora file\SRP\assets\simple-probes.png" alt="img" style="zoom:67%;" />

<center>Simple reflection probes mode selected.</center>

除此之外，反射探针还有一个选项可以启用盒子投影模式。这应该会改变反射的确定方式，以更好地匹配其有限的影响区域，但这也不被SRP批处理程序所支持，所以我们也不会支持它。

#### [2.7、Decoding Probes](https://catlikecoding.com/unity/tutorials/custom-srp/lod-and-reflections/#2.7)

最后，我们要确保正确解释<font color="green">cube map</font>的数据。它可能是<font color="DarkOrchid ">HDR或LDR</font>，其强度也可以调整。这些设置通过<font color="RoyalBlue">unity_SpecCube0_HDR</font>向量来实现，它在<font color="green">UnityPerDraw</font>缓冲区的<font color="green">unity_ProbesOcclusion</font>之后。

```glsl
CBUFFER_START(UnityPerDraw)
	…

	float4 unity_ProbesOcclusion;
	
	float4 unity_SpecCube0_HDR;
	
	…
CBUFFER_END
```

我们通过在<font color="red">SampleEnvironment</font>的末尾以原始环境数据和设置为参数调用<font color="RoyalBlue">DecodeHDREnvironment</font>，得到正确的颜色。

```glsl
float3 SampleEnvironment (Surface surfaceWS, BRDF brdf) {
	…
	return DecodeHDREnvironment(environment, unity_SpecCube0_HDR);
}
```

