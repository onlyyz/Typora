# 06_Shadow Masks

烘烤静态阴影。
		结合实时照明和烘烤阴影。
		混合实时和烘烤的阴影。
		支持最多四个阴影遮罩的灯光。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/tutorial-image.jpg)

<center>Realtime shadows nearby, baked shadows farther away.</center>

### [1、Baking Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/#1)

使用<font color="green">light map</font>的好处是，我们不受限于最大的阴影距离。烘焙的阴影不会被剔除，但它们也不能改变。理想情况下，我们可以在最大阴影距离内使用实时阴影，超过这个距离则使用烘烤阴影。Unity的<font color="red">shadow mask mixed lighting</font>模式使这成为可能。

#### 1.1、Distance Shadow Mask

让我们考虑一下上一个教程中的同一个场景，但是减少了最大阴影距离，这样结构内部的一部分就不会有阴影了。这使得实时阴影的终点非常清楚。我们从只有一个光源开始。

<img src="E:\Typora file\SRP\assets\max-distance-11.png" alt="img" style="zoom:50%;" />

<center>Baked indirect mixed lighting, max distance 11.</center>

将混合光照模式切换到<font color="red">Shadowmask</font>。这将使光照数据失效，所以它必须再次被Baked。

<img src="E:\Typora file\SRP\assets\shadow-mask-lighting-mode.png" alt="img" style="zoom:50%;" />

<center>*Shadowmask* mixed lighting mode.</center>

有两种方法可以使用<font color="red">shadow mask mixed lighting</font>，可以通过<font color="green">*Quality* project settings. </font>进行配置。我们将使用<font color="green">Distance Shadowmask</font>模式。另一种模式被称为只是<font color="blue">Shadowmask</font>，我们将在后面介绍。

<img src="E:\Typora file\SRP\assets\shadow-mask-mode.png" alt="img" style="zoom:50%;" />

<center>Shadow mask mode set to distance.</center>

这两种类型的阴影遮罩模式使用相同的<font color="green">baked lighting data</font>。

在这两种情况下，<font color="blue">light map</font>最终都包含了<font color="red"> indirect lighting</font>，与烘焙的间接混合光照模式完全一样。不同的是，现在也有一个烘烤的<font color="red"> shadow mask map</font>，你可以通过烘烤的<font color="red">light map</font>预览窗口检查。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/baking-shadows/baked-indirect-light.png" alt="indirect light" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\baked-shadow-mask.png" alt="shadow mask" style="zoom:50%;" />

 <center>*Baked indirect light and shadow mask.*</center>

<font color="green">shadow mask map</font>包含了我们<font color="blue">shadow attenuation of our single mixed directional light - 单一混合方向光的阴影衰减</font>，代表了所有对全局照明有贡献的静态物体投下的阴影。该数据被存储在红色通道中，所以地图是黑色和红色的。

就像<font color="green"> baked indirect lighting</font>一样，烘焙的阴影在运行时不能改变。然而，无论光线的强度或颜色如何，阴影将保持有效。

但是灯光不应该被旋转，否则它的阴影就没有意义了。另外，如果灯光的间接照明被烘烤，你不应该改变太多。例如，如果间接照明在灯光关闭后仍然存在，这显然是错误的。如果一盏灯变化很大，那么你可以把它的间接倍数设置为零，这样就不会有间接光被烘烤。

#### 1.2、Detecting a Shadow Mask

要使用<font color="red">阴影遮罩</font>，我们的管线必须首先知道它的存在。因为这都是关于阴影的，这是我们的<font color="red">Shadows类</font>的工作。我们将使用着色器关键字来控制是否使用<font color="red">shadow masks</font>。由于有两种模式，我们将引入另一个静态关键字数组，尽管它现在只包含一个关键字。<font color="green">_shadow_mask_distance。</font>

```c#
	static string[] shadowMaskKeywords = {
		"_SHADOW_MASK_DISTANCE"
	};
```

添加一个布尔字段来跟踪我们是否使用了<font color="red">阴影遮罩</font>。我们每一帧都会重新评估，所以在设置中初始化为false。

```c#
	bool useShadowMask;

	public void Setup (…) {
		…
		useShadowMask = false;
	}
```

在<font color="green">Render</font>的结尾处启用或取消关键字。即使我们最终没有渲染任何实时的阴影，我们也必须这样做，因为<font color="red">阴影遮罩不是实时的。</font>

```c#
public void Render () {
		…
		buffer.BeginSample(bufferName);
		SetKeywords(shadowMaskKeywords, useShadowMask ? 0 : -1);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

要知道是否需要一个阴影遮罩，我们必须检查是否有一个灯光使用它。

我们将在<font color="red">ReserveDirectionalShadows</font>中做这件事，当我们最后有一个有效的<font color="blue">valid shadow-casting light - 投射阴影的灯光.</font>时。

每个灯都包含关于其<font color="green"> baked data</font>的信息。它被存储在<font color="red">LightBakingOutput</font>结构中，可以通过<font color="red">Light.bakingOutput</font>属性来获取。如果我们遇到一个灯光，它的<font color="green">light map bake type set to mixed - light map烘焙模式被设置为混合</font>，并且它的<font color="green"> mixed lighting mode set to shadow mask - 混合照明模式被设置为阴影遮罩</font>，那么我们就会使用阴影遮罩。

```c#
	public Vector3 ReserveDirectionalShadows (
		Light light, int visibleLightIndex
	) {
		if (…) {
			LightBakingOutput lightBaking = light.bakingOutput;
			if (
				lightBaking.lightmapBakeType == LightmapBakeType.Mixed &&
				lightBaking.mixedLightingMode == MixedLightingMode.Shadowmask
			) {
				useShadowMask = true;
			}

			…
		}
		return Vector3.zero;
	}
```

这会在需要时启用<font color="red"> shader</font> 关键字。 在 Lit 着色器的<font color="red"> CustomLit pass </font>中为其添加相应的多编译指令。


```glsl
#pragma multi_compile _ _CASCADE_BLEND_SOFT _CASCADE_BLEND_DITHER
#pragma multi_compile _ _SHADOW_MASK_DISTANCE
#pragma multi_compile _ LIGHTMAP_ON
```

#### 1.3、Shadow Mask Data

在着色器方面，我们必须知道是否使用了<font color="green"> shadow mask</font>，如果使用了，那么烘烤的阴影是什么。

让我们为<font color="red">Shadows.hlsl</font>添加一个<font color="blue">ShadowMask</font>结构来跟踪这两方面的信息，包括一个布尔值和一个浮点矢量场。给布尔值<font color="green">distance</font>命名，以表示是否启用了<font color="blue">distance shadow mask - 距离阴影遮罩</font>模式。然后将这个结构作为一个字段添加到全局<font color="red">ShadowData</font>结构中。



```glsl
struct ShadowMask {
	bool distance;
	float4 shadows;
};

struct ShadowData {
	int cascadeIndex;
	float cascadeBlend;
	float strength;
	ShadowMask shadowMask;
};
```

在<font color="red">GetShadowData</font>中默认将阴影掩码初始化为不使用。

```C#
ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.shadowMask.distance = false;
	data.shadowMask.shadows = 1.0;
	…
}
```

尽管阴影遮罩是用于阴影的，但它是<font color="green">场景的烘烤照明数据的一部分</font>。因此，检索它是<font color="red">GI</font>的责任。所以在<font color="red">GI结构</font>中也添加一个<font color="blue">阴影遮罩</font>字段，并在<font color="red">GetGI</font>中初始化为不使用。

```glsl
struct GI {
	float3 diffuse;
	ShadowMask shadowMask;
};

…

GI GetGI (float2 lightMapUV, Surface surfaceWS) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	gi.shadowMask.distance = false;
	gi.shadowMask.shadows = 1.0;
	return gi;
}
```

Unity通过<font color="red">unity_ShadowMask</font>纹理和相应的<font color="green">Sample</font>采样器状态使<font color="blue">shadow mask map </font>在<font color="green">Shader</font>中可用。在GI中与其他<font color="green">light map texture and sampler state.</font>一起定义。

```glsl
TEXTURE2D(unity_Lightmap);
SAMPLER(samplerunity_Lightmap);

TEXTURE2D(unity_ShadowMask);
SAMPLER(samplerunity_ShadowMask);
```

然后添加一个<font color="red">SampleBakedShadows</font>函数，使用<font color="blue"> light map UV coordinates.</font>对其进行采样。就像普通的<font color="green"> light map</font>一样，这只对<font color="green">light map</font>的几何体有用，所以只有当<font color="red">LIGHTMAP_ON</font>被定义，否则就不会有烘焙的阴影且attenuation总是1。

```glsl
float4 SampleBakedShadows (float2 lightMapUV) {
	#if defined(LIGHTMAP_ON)
		return SAMPLE_TEXTURE2D(
			unity_ShadowMask, samplerunity_ShadowMask, lightMapUV
		);
	#else
		return 1.0;
	#endif
}
```

现在我们可以调整<font color="red">GetGI</font>，<font color="red">使其在定义了<font color="green">_SHADOW_MASK_DISTANCE</font>的情况下启用<font color="blue">distance shadow mask mode and samples the baked shadows - 距离阴影遮罩模式并对烘焙的阴影进行采样</font></font>。

注意，这使得<font color="blue">distance</font>布尔值成为编译时常量，所以它的使用不会导致<font color="green"> dynamic branching - 动态分支。</font>

```glsl
GI GetGI (float2 lightMapUV, Surface surfaceWS) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	gi.shadowMask.distance = false;
	gi.shadowMask.shadows = 1.0;

	#if defined(_SHADOW_MASK_DISTANCE)
		gi.shadowMask.distance = true;
		gi.shadowMask.shadows = SampleBakedShadows(lightMapUV);
	#endif
	return gi;
}
```

在<font color="red">GetLighting</font>中，由<font color="green">Lighting</font>将<font color="green">shadow mask data </font>从<font color="red">GI</font>复制到<font color="blue">ShadowData</font>中，然后再通过灯光循环。在这一点上，我们也可以通过直接返回<font color="green">ShadowData</font>作为最终的照明颜色来进行调试。

```glsl
float3 GetLighting (Surface surfaceWS, BRDF brdf, GI gi) {
	ShadowData shadowData = GetShadowData(surfaceWS);
	shadowData.shadowMask = gi.shadowMask;
	return gi.shadowMask.shadows.rgb;
	
	…
}
```

最初它似乎并不工作，因为一切都以白色结束。我们必须指示<font color="green">Unity向GPU</font>发送相关数据，就像我们在前面的教程中为<font color="green">CameraRenderer.DrawVisibleGeometry</font>中的光照图和探头所做的那样。

在这种情况下，我们必须将<font color="red">PerObjectData.ShadowMask</font>添加到每个物体的数据中。

[PerObjectData](https://docs.unity3d.com/ScriptReference/Rendering.PerObjectData.html)

![image-20230227211137844](E:\Typora file\SRP\assets\image-20230227211137844.png)

```glsl
	perObjectData =
				PerObjectData.Lightmaps | PerObjectData.ShadowMask |
				PerObjectData.LightProbe |
				PerObjectData.LightProbeProxyVolume
```

<img src="E:\Typora file\SRP\assets\sampled-shadow-mask.png" alt="img" style="zoom:50%;" />

<center>Sampling shadow mask.</center>

#### 1.4、Occlusion Probes

我们可以看到，<font color="green">shadow mask</font>被正确地应用于<font color="green">lightmapped objects</font>。我们还看到，<font color="red">动态对象没有<font color="green">shadow mask data</font>，正如预期的那样。它们使用<font color="RoyalBlue">光探针而不是光照图</font>。</font>

然而，Unity也将<font color="blue">shadow mask data</font>纳入<font color="RoyalBlue"> light probes</font>，将其称为<font color="red">occlusion probes</font>。我们可以通过在<font color="red">UnityInput</font>的<font color="RoyalBlue">UnityPerDraw缓冲区</font>中添加一个<font color="red">unity_ProbesOcclusion向量</font>来访问这些数据。把它放在世界变换参数和光照图UV变换向量之间。

```glsl
real4 unity_WorldTransformParams;

	float4 unity_ProbesOcclusion;

	float4 unity_LightmapST;
```

现在我们可以简单地在<font color="red">SampleBakedShadows</font>中为动态对象返回该向量。

```glsl
float4 SampleBakedShadows (float2 lightMapUV) {
	#if defined(LIGHTMAP_ON)
		…
	#else
		return unity_ProbesOcclusion;
	#endif
}
```

再次，我们必须指示<font color="RoyalBlue">Unity将这些数据发送到GPU</font>，这次是通过启用<font color="red">PerObjectData.OcclusionProbe</font>标志。

```glsl
		perObjectData =
				PerObjectData.Lightmaps | PerObjectData.ShadowMask |
				PerObjectData.LightProbe | PerObjectData.OcclusionProbe |
				PerObjectData.LightProbeProxyVolume
```

<img src="E:\Typora file\SRP\assets\sampled-occlusion-probes.png" alt="img" style="zoom:50%;" />

<center>Sampling occlusion probes.</center>

<font color="red"><font color="DarkOrchid ">shadow mask</font>的未使用通道被设置为<font color="RoyalBlue">探针的白色</font>，因此<font color="green">动态物体</font>在完全被照亮时是白色的</font>，在完全被阴影遮住时是青色的，而不是红色和黑色。

虽然这足以让阴影遮罩通过探针工作，但它破坏了<font color="red">GPU实例化</font>。

<font color="DarkOrchid ">occlusion data —遮挡数据可以自动得到实例化</font>，但是<font color="green">UnityInstancing</font>只有在定义了<font color="green">SHADOWS_SHADOWMASK</font>时才会这样做。

所以在包括<font color="red">UnityInstancing</font>之前，需要时在Common中定义它。这是唯一一个我们必须明确检查<font color="RoyalBlue">_SHADOW_MASK_DISTANCE</font>是否被定义的地方。

```glsl
#if defined(_SHADOW_MASK_DISTANCE)
	#define SHADOWS_SHADOWMASK
#endif

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
```

#### [1.5、LPPVs](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/#1.5)

<font color="green">Light probe proxy volumes </font>也可以与<font color="red"> shadow masks</font>一起工作。我们必须再次通过设置一个标志来启用它，这次是<font color="RoyalBlue">PerObjectData.OcclusionProbeProxyVolume。</font>

```glsl
	perObjectData =
				PerObjectData.Lightmaps | PerObjectData.ShadowMask |
				PerObjectData.LightProbe | PerObjectData.OcclusionProbe |
				PerObjectData.LightProbeProxyVolume |
				PerObjectData.OcclusionProbeProxyVolume
```

检索<font color="red"> LPPV occlusion data </font>的工作原理与检索其光照数据相同，只是我们必须调用<font color="green">SampleProbeOcclusion</font>而不是<font color="green">SampleProbeVolumeSH4</font>。

它存储在相同的纹理中，需要相同的参数，唯一的例外是不需要法线矢量。在<font color="green">SampleBakedShadows</font>中添加一个分支，以及一个现在需要的<font color="DarkOrchid "> surface parameter for  world position.</font>。

 SampleProbeOcclusion

```glsl
float4 SampleBakedShadows (float2 lightMapUV, Surface surfaceWS) {
	#if defined(LIGHTMAP_ON)
		…
	#else
		if (unity_ProbeVolumeParams.x) {
			return SampleProbeOcclusion(
				TEXTURE3D_ARGS(unity_ProbeVolumeSH, samplerunity_ProbeVolumeSH),
				surfaceWS.position, unity_ProbeVolumeWorldToObject,
				unity_ProbeVolumeParams.y, unity_ProbeVolumeParams.z,
				unity_ProbeVolumeMin.xyz, unity_ProbeVolumeSizeInv.xyz
			);
		}
		else {
			return unity_ProbesOcclusion;
		}
	#endif
}
```

在<font color="red">GetGI</font>中调用函数时添加新的表面参数。

```c#
	gi.shadowMask.shadows = SampleBakedShadows(lightMapUV, surfaceWS);
```

<img src="E:\Typora file\SRP\assets\sampled-lppv-occlusion.png" alt="img" style="zoom:50%;" />

<center>Sampling LPPV occlusion.</center>

#### [1.6、Mesh Ball](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/#1.6)

如果我们的<font color="green">Mesh</font>球使用LPPV，它已经支持阴影遮蔽。

但是当它自己插值光线探针的时候，我们必须在<font color="green">MeshBall.Update</font>中添加遮挡探针数据。这是通过使用一个临时的<font color="green">Vector4</font>数组作为<font color="red">CalculateInterpolatedLightAndOcclusionProbes</font>的最后一个参数，并通过<font color="RoyalBlue">CopyProbeOcclusionArrayFrom</font>方法将其传递给属性块。

```glsl
	var lightProbes = new SphericalHarmonicsL2[1023];
				var occlusionProbes = new Vector4[1023];
				LightProbes.CalculateInterpolatedLightAndOcclusionProbes(
					positions, lightProbes, occlusionProbes
				);
				block.CopySHCoefficientArraysFrom(lightProbes);
				block.CopyProbeOcclusionArrayFrom(occlusionProbes);
```

在验证了阴影遮罩数据被正确地发送到着色器后，我们可以从<font color="red">GetLighting</font>中删除其调试可视化。

```glsl
	//return gi.shadowMask.shadows.rgb;
```

### 2、Mixing Shadows

现在我们有了阴影遮罩，下一步就是在没有实时阴影的情况下使用它，也就是当一个片段最后超出最大阴影距离的情况。

#### [2.1、Use Baked when Available](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/#2.1)

混合烘烤和实时阴影将使<font color="red">GetDirectionalShadowAttenuation</font>的工作更加复杂。让我们首先隔离所有的实时阴影采样代码，将其转移到<font color="green">Shadows</font>中的一个新的<font color="blue">GetCascadedShadow</font>函数。

```glsl
float GetCascadedShadow (
	DirectionalShadowData directional, ShadowData global, Surface surfaceWS
) {
	float3 normalBias = surfaceWS.normal *
		(directional.normalBias * _CascadeData[global.cascadeIndex].y);
	float3 positionSTS = mul(
		_DirectionalShadowMatrices[directional.tileIndex],
		float4(surfaceWS.position + normalBias, 1.0)
	).xyz;
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
	return shadow;
}

float GetDirectionalShadowAttenuation (
	DirectionalShadowData directional, ShadowData global, Surface surfaceWS
) {
	#if !defined(_RECEIVE_SHADOWS)
		return 1.0;
	#endif
	
	float shadow;
	if (directional.strength <= 0.0) {
		shadow = 1.0;
	}
	else {
		shadow = GetCascadedShadow(directional, global, surfaceWS);
		shadow = lerp(1.0, shadow, directional.strength);
	}
	return shadow;
}
```

然后添加一个新的<font color="red">GetBakedShadow</font>函数，返回给<font color="DarkOrchid "> shadow mask</font>的<font color="green"> baked shadow attenuation </font>。<font color="RoyalBlue">如果遮罩的距离模式被启用</font>，那么我们需要它的阴影向量的第一个分量，否则就没有可用的衰减，结果是1。

```glsl
float GetBakedShadow (ShadowMask mask) {
	float shadow = 1.0;
	if (mask.distance) {
		shadow = mask.shadows.r;
	}
	return shadow;
}
```

接下来，创建一个<font color="green">MixBakedAndRealtimeShadows</font>函数，带有<font color="RoyalBlue">ShadowData、realtime shadow和阴影强度参数</font>。

它只是简单地将强度应用于阴影，除非有一个<font color="green">distance shadow mask</font>。如果有的话，就用<font color="red">烘焙的阴影替换实时阴影。</font>

```glsl
float MixBakedAndRealtimeShadows (
	ShadowData global, float shadow, float strength
) {
	float baked = GetBakedShadow(global.shadowMask);
	if (global.shadowMask.distance) {
		shadow = baked;
	}
	return lerp(1.0, shadow, strength);
}
```

让<font color="green">GetDirectionalShadowAttenuation</font>使用该函数，而不是应用强度本身。

```glsl
	shadow = GetCascadedShadow(directional, global, surfaceWS);
		shadow = MixBakedAndRealtimeShadows(global, shadow, directional.strength);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/mixing-shadows/faded-baked-shadows.png" alt="img" style="zoom:50%;" />

<center>Faded baked shadows.</center>

结果是，我们现在总是使用<font color="green">shadow mask</font>，所以我们可以看到它是有效的。然而，烘焙出来的阴影会随着距离的增加而褪色，就像现实中的阴影一样。

#### 2.2、Transitioning to Baked

为了从<font color="green">实时阴影过渡到基于深度的烘烤阴影</font>，我们必须根据全局阴影强度在它们之间进行插值。

然而，我们还必须应用<font color="RoyalBlue"> light's shadow strength </font>，这必须在插值之后进行。所以我们不能再立即在<font color="green">GetDirectionalShadowData</font>中结合两种强度。

Scripts at Light.hlsl

```glsl
data.strength =
		_DirectionalLightShadowData[lightIndex].x; // * shadowData.strength;
```

在<font color="red">MixBakedAndRealtimeShadows</font>中，根据<font color="DarkOrchid ">根据全局强度在 baked and realtime based </font>之间进行插值，<font color="RoyalBlue">然后再应用灯光的阴影强度。</font>

但是当<font color="green">没有shadow mask</font>的时候，就像我们之前做的那样，只对实时阴影应用综合强度。

```glsl
float MixBakedAndRealtimeShadows (
	ShadowData global, float shadow, float strength
) {
	float baked = GetBakedShadow(global.shadowMask);
	if (global.shadowMask.distance) {
		shadow = lerp(baked, shadow, global.strength);
		return lerp(1.0, shadow, strength);
	}
	return lerp(1.0, shadow, strength * global.strength);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/mixing-shadows/mixed-shadows.png" alt="img" style="zoom:50%;" />

<center>Mixed shadows.</center>

其结果是，动态物体投下的阴影照常消退，而静态物体投下的阴影则过渡到阴影遮罩。

#### [2.3、Only Baked Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/#2.3)

目前，我们的方法只在有实时阴影需要渲染时才起作用。如果没有，那么阴影遮罩也就消失了。这可以通过放大场景视图直到所有东西都在最大阴影距离之外来验证。

![img](E:\Typora file\SRP\assets\neither-realtime-nor-baked.png)

<center>Neither realtime nor baked shadows.</center>

我们必须支持有<font color="red">shadow mask but no realtime shadows</font>的情况。

让我们首先创建一个<font color="green">GetBakedShadow</font>函数的变体，它也有一个强度参数，所以我们可以方便地得到一个强度调制的<font color="RoyalBlue">baked shadow.</font>

```glsl
float GetBakedShadow (ShadowMask mask, float strength) {
	if (mask.distance) {
		return lerp(1.0, GetBakedShadow(mask), strength);
	}
	return 1.0;
}
```

接下来，在<font color="DarkOrchid ">GetDirectionalShadowAttenuation</font>中检查<font color="green">combined Shadow strengths</font>是否最终为零或更少。如果是，而不是总是返回1，只返回调制后的烘烤阴影，仍然跳过实时的阴影采样。

```
if (directional.strength * global.strength <= 0.0) {
		shadow = GetBakedShadow(global.shadowMask, directional.strength);
	}
```

除此之外，我们必须改变<font color="red">Shadows.ReserveDirectionalShadows</font>，这样它就不会立即跳过那些最终没有<font color="green">实时投影的灯光。</font>

相反，首先确定该灯是否使用了<font color="DarkOrchid ">Shadow Mask</font>。然后检查是否没有<font color="DarkOrchid ">realtime shadow casters,</font>，在这种情况下，只有<font color="RoyalBlue">阴影强度</font>是相关的。

```glsl
if (
			shadowedDirLightCount < maxShadowedDirLightCount &&
			light.shadows != LightShadows.None && light.shadowStrength > 0f //&&
			//cullingResults.GetShadowCasterBounds(visibleLightIndex, out Bounds b)
		) {
			LightBakingOutput lightBaking = light.bakingOutput;
			if (
				lightBaking.lightmapBakeType == LightmapBakeType.Mixed &&
				lightBaking.mixedLightingMode == MixedLightingMode.Shadowmask
			) {
				useShadowMask = true;
			}

			if (!cullingResults.GetShadowCasterBounds(
				visibleLightIndex, out Bounds b
			)) {
				return new Vector3(light.shadowStrength, 0f, 0f);
			}

			…
		}
```

但是当阴影强度大于0时，<font color="blue">Shader</font>将对<font color="green"> shadow map</font>进行采样，尽管这是不正确的。在这种情况下，我们可以通过负值阴影强度来使其工作。这样就没有阴影灯的影响

```glsl
	return new Vector3(-light.shadowStrength, 0f, 0f);
```

然后当我们跳过实时阴影时，在<font color="green">GetDirectionalShadowAttenuation</font>中传递绝对强度给<font color="red">GetBakedShadow</font>。

这样，当没有实时阴影投射者时，以及当我们超出最大阴影距离时，它都能发挥作用。

```
shadow = GetBakedShadow(global.shadowMask, abs(directional.strength));
```

![img](E:\Typora file\SRP\assets\baked-only.png)

<center>Only baked shadows.</center>

#### 2.4、Always use the Shadow Mask

还有一种<font color="DarkOrchid ">shadow mask mode</font>，它被简单地称为<font color="green">Shadowmask</font>。它的工作原理与<font color="red">distance mode</font>完全相同，只是Unity会省略使用<font color="red">Shadow Mask</font>的灯光的<font color="RoyalBlue">static shadow casters</font>。

<font color="green">setting at quality</font>

<img src="E:\Typora file\SRP\assets\shadow-mask-mode-always.png" alt="project settings" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\no-static-shadows.png" alt="scene" style="zoom:50%;" />

<center>No realtime shadows cast by static geometry.</center>

我们的想法是，由于阴影遮罩在任何地方都是可用的，我们也可以把它用于任何地方的静态阴影。

这意味着更少的实时阴影，这使得渲染更快，但代价是近距离的静态阴影质量更低。

为了支持这种模式，在<font color="green">Shadows</font>中添加一个<font color="red">_SHADOW_MASK_ALWAYS</font>关键字作为<font color="DarkOrchid "> shadow mask</font>关键字阵列的第一个元素。

我们可以通过检查<font color="DarkOrchid ">QualitySettings.shadowmaskMode</font>属性来确定在Render中应该启用哪个模式。

```glsl
	static string[] shadowMaskKeywords = {
		"_SHADOW_MASK_ALWAYS",
		"_SHADOW_MASK_DISTANCE"
	};
	
	…
	
	public void Render () {
		…
		buffer.BeginSample(bufferName);
		SetKeywords(shadowMaskKeywords, useShadowMask ?
			QualitySettings.shadowmaskMode == ShadowmaskMode.Shadowmask ? 0 : 1 :
			-1
		);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

在我们的着色器中的多编译指令中添加该关键字。

```glsl
		#pragma multi_compile _ _SHADOW_MASK_ALWAYS _SHADOW_MASK_DISTANCE
```

而且在决定定义<font color="red">SHADOWS_SHADOWMASK</font>时，还要在<font color="DarkOrchid ">Common</font>中检查它。 

```
#if defined(_SHADOW_MASK_ALWAYS) || defined(_SHADOW_MASK_DISTANCE)
	#define SHADOWS_SHADOWMASK
#endif
```

给<font color="green">ShadowMask</font>结构一个单独的布尔字段，以指示是否应始终使用<font color="DarkOrchid ">Shaodw Mask</font>。

```glsl
struct ShadowMask {
	bool always;
	bool distance;
	float4 shadows;
};

…

ShadowData GetShadowData (Surface surfaceWS) {
	ShadowData data;
	data.shadowMask.always = false;
	…
}
```

然后在适当的时候在<font color="green">GetGI</font>中设置它，以及它的<font color="red">shadow data.</font>

```glsl
GI GetGI (float2 lightMapUV, Surface surfaceWS) {
	GI gi;
	gi.diffuse = SampleLightMap(lightMapUV) + SampleLightProbe(surfaceWS);
	gi.shadowMask.always = false;
	gi.shadowMask.distance = false;
	gi.shadowMask.shadows = 1.0;

	#if defined(_SHADOW_MASK_ALWAYS)
		gi.shadowMask.always = true;
		gi.shadowMask.shadows = SampleBakedShadows(lightMapUV, surfaceWS);
	#elif defined(_SHADOW_MASK_DISTANCE)
		gi.shadowMask.distance = true;
		gi.shadowMask.shadows = SampleBakedShadows(lightMapUV, surfaceWS);
	#endif
	return gi;
}
```

两个版本的<font color="green">GetBakedShadow</font>在使用任一模式时都应该选择<font color="red">mask</font>。

```glsl
float GetBakedShadow (ShadowMask mask) {
	float shadow = 1.0;
	if (mask.always || mask.distance) {
		shadow = mask.shadows.r;
	}
	return shadow;
}

float GetBakedShadow (ShadowMask mask, float strength) {
	if (mask.always || mask.distance) {
		return lerp(1.0, GetBakedShadow(mask), strength);
	}
	return 1.0;
}
```

最后，<font color="DarkOrchid ">MixBakedAndRealtimeShadows</font>现在必须在<font color="red">shadow mask </font>始终处于活动状态时使用一种不同的方法。

首先，<font color="red">realtime shadow</font>必须由<font color="green">global strength - 全局强度</font>来调制，以根据深度来<font color="red">Fade</font>它。

然后，通过取其最小值，将<font color="red"> baked and realtime shadows </font>结合起来。之后，<font color="RoyalBlue">light's shadow strength </font>被应用到合并后的阴影中。

```glsl
float MixBakedAndRealtimeShadows (
	ShadowData global, float shadow, float strength
) {
	float baked = GetBakedShadow(global.shadowMask);
	if (global.shadowMask.always) {
		shadow = lerp(1.0, shadow, global.strength);
		shadow = min(baked, shadow);
		return lerp(1.0, shadow, strength);
	}
	if (global.shadowMask.distance) {
		shadow = lerp(baked, shadow, global.strength);
		return lerp(1.0, shadow, strength);
	}
	return lerp(1.0, shadow, strength * global.strength);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/shadow-masks/mixing-shadows/baked-static-shadows.png" alt="img" style="zoom:50%;" />

<center>Baked static shadows mixed with realtime dynamic shadows.</center>

### 3、Multiple Lights

因为阴影遮罩图有四个通道，它最多可以支持四个混合光。烘烤时最重要的灯光得到红色通道，第二个灯光得到绿色通道，以此类推。让我们通过复制我们的单一方向的光，旋转一下，并降低它的强度，使新的光最终使用绿色通道来尝试一下。

<img src="E:\Typora file\SRP\assets\lights-sharing-baked-shadows.png" alt="img" style="zoom:50%;" />

<center>Two lights sharing the same baked shadows.</center>

第二盏灯的实时阴影按预期工作，但它最终使用第一盏灯的遮罩来烘烤阴影，这显然是错误的。这在使用 "<font color="red">always-shadow-mask</font>"模式时最容易看到。

#### 3.1、Shadow Mask Channels

检查烘焙后的<font color="DarkOrchid ">shadow mask map </font>可以发现，阴影的烘焙是正确的。只有第一个灯光照射的区域是红色的，只有第二个灯光照射的区域是绿色的，而两个灯光照射的区域是黄色的。这对最多四盏灯都有效，尽管第四盏灯在预览中是不可见的，因为没有显示alpha通道。

<img src="E:\Typora file\SRP\assets\baked-shadows-two-lights.png" alt="img" style="zoom:50%;" />

<center>Baked shadows for two lights.</center>

两个灯都使用相同的<font color="green">baked shadows</font>，因为我们总是使用红色通道。

为了使其发挥作用，我们必须将灯光的通道索引发送到GPU上。我们不能依赖灯光的顺序，因为它可以在运行时变化，因为灯光可以被改变，甚至被禁用。

我们可以在<font color="red">Shadows.ReserveDirectionalShadows</font>中通过<font color="DarkOrchid ">LightBakingOutput.occlusionMaskChannel</font>字段检索灯光的<font color="RoyalBlue">light's mask channel index - 遮蔽通道索引</font>。

由于我们要向GPU发送一个4D向量，我们可以把它存储在我们返回的向量的第四个通道中，把返回类型改为<font color="red">Vector4</font>。而当灯光不使用<font color="green">shadow mask</font>时，我们通过将其索引设置为-1来表示。

[LightBakingOutput](https://docs.unity3d.com/ScriptReference/LightBakingOutput.html). [occlusionMaskChannel ](https://docs.unity3d.com/ScriptReference/LightBakingOutput-occlusionMaskChannel.html)

在 [光照贴图烘焙类型.混合](https://docs.unity3d.com/ScriptReference/LightmapBakeType.Mixed.html) light，包含要使用的<font color="red">occlusion mask channel </font>的索引（如果有），否则为 -1。

```c#
	public Vector4 ReserveDirectionalShadows (
		Light light, int visibleLightIndex
	) {
		if (
			shadowedDirLightCount < maxShadowedDirLightCount &&
			light.shadows != LightShadows.None && light.shadowStrength > 0f
		) {
			float maskChannel = -1;
			LightBakingOutput lightBaking = light.bakingOutput;
			if (
				lightBaking.lightmapBakeType == LightmapBakeType.Mixed &&
				lightBaking.mixedLightingMode == MixedLightingMode.Shadowmask
			) {
				useShadowMask = true;
				maskChannel = lightBaking.occlusionMaskChannel;
			}

			if (!cullingResults.GetShadowCasterBounds(
				visibleLightIndex, out Bounds b
			)) {
				return new Vector4(-light.shadowStrength, 0f, 0f, maskChannel);
			}

			shadowedDirectionalLights[shadowedDirLightCount] =
				new ShadowedDirectionalLight {
					visibleLightIndex = visibleLightIndex,
					slopeScaleBias = light.shadowBias,
					nearPlaneOffset = light.shadowNearPlane
				};
			return new Vector4(
				light.shadowStrength,
				settings.directional.cascadeCount * shadowedDirLightCount++,
				light.shadowNormalBias, maskChannel
			);
		}
		return new Vector4(0f, 0f, 0f, -1f);
	}
```

#### 3.2、Selecting the Appropriate Channel

在<font color="green">shader size</font>上，将<font color="red"> shadow mask channel - 阴影遮罩通道</font>作为一个额外的整数字段添加到Shadows中定义的<font color="RoyalBlue">DirectionalShadowData</font>结构中。

代码在<font color="DarkOrchid ">Shadow.hlsl</font>

```glsl
struct DirectionalShadowData {
	float strength;
	int tileIndex;
	float normalBias;
	int shadowMaskChannel;
};
```

然后<font color="DarkOrchid ">GI</font>必须在<font color="red">GetDirectionalShadowData</font>中设置通道。

代码在<font color="DarkOrchid ">Light.hlsl</font>

```glsl
DirectionalShadowData GetDirectionalShadowData (
	int lightIndex, ShadowData shadowData
) {
	…
	data.shadowMaskChannel = _DirectionalLightShadowData[lightIndex].w;
	return data;
}
```

在<font color="red">GetBakedShadow</font>的两个版本中都添加一个通道参数，用它来返回适当的<font color="green">shadow mask data</font>。但只有在灯光使用<font color="DarkOrchid ">e shadow mask,</font>时才这样做，所以当通道至少为零时。

```glsl
float GetBakedShadow (ShadowMask mask, int channel) {
	float shadow = 1.0;
	if (mask.always || mask.distance) {
		if (channel >= 0) {
			shadow = mask.shadows[channel];
		}
	}
	return shadow;
}

float GetBakedShadow (ShadowMask mask, int channel, float strength) {
	if (mask.always || mask.distance) {
		return lerp(1.0, GetBakedShadow(mask, channel), strength);
	}
	return 1.0;
}
```

调整<font color="red">MixBakedAndRealtimeShadows</font>，使其沿着所需的<font color="RoyalBlue"> shadow mask channel.</font>传递。

```
float MixBakedAndRealtimeShadows (
	ShadowData global, float shadow, int shadowMaskChannel, float strength
) {
	float baked = GetBakedShadow(global.shadowMask, shadowMaskChannel);
	…
}
```

最后，在<font color="red">GetDirectionalShadowAttenuation</font>中添加需要的通道参数。

```glsl
float GetDirectionalShadowAttenuation (
	DirectionalShadowData directional, ShadowData global, Surface surfaceWS
) {
	#if !defined(_RECEIVE_SHADOWS)
		return 1.0;
	#endif
	
	float shadow;
	if (directional.strength * global.strength <= 0.0) {
		shadow = GetBakedShadow(
			global.shadowMask, directional.shadowMaskChannel,
			abs(directional.strength)
		);
	}
	else {
		shadow = GetCascadedShadow(directional, global, surfaceWS);
		shadow = MixBakedAndRealtimeShadows(
			global, shadow, directional.shadowMaskChannel, directional.strength
		);
	}
	return shadow;
}
```

<img src="E:\Typora file\SRP\assets\two-lights-two-channels.png" alt="img" style="zoom:50%;" />

<center>Both lights using their own channel.</center>