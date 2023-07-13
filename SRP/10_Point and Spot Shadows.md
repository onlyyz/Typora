# 10、Point and Spot Shadows

 混合点光源和聚光灯的烘烤和实时阴影。
		添加第二个阴影图集。
		用透视投影渲染和采样阴影。
		使用自定义立方体地图。

![img](E:\Typora file\SRP\assets\tutorial-image-1677686193864-79.jpg)

<center>100% realtime shadows.</center>

### 1、Spot Light Shadows

我们将从支持<font color="red"> spot lights.</font>的<font color="RoyalBlue"> realtime shadows</font>开始。我们将使用与方向性灯光相同的方法，但有一些变化。我们还将尽可能保持简单的支持，使用一个统一的平铺的<font color="green">shadow atlas</font>，并按照Unity提供的顺序用阴影灯填充它。

#### 1.1、Shadow Mixing

第一步是<font color="DarkOrchid ">mix baked and realtime shadows</font>。

调整<font color="green">Shadows</font>中的<font color="red">GetOtherShadowAttenuation</font>，使它的行为与<font color="RoyalBlue">GetDirectionalShadowAttenuation</font>类似，只是它使用其他的阴影数据，并依赖于一个新的<font color="green">GetOtherShadow</font>函数。这个新函数最初返回1，因为其他灯光还没有实时的阴影。

```c#
float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	return 1.0;
}

float GetOtherShadowAttenuation (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	#if !defined(_RECEIVE_SHADOWS)
		return 1.0;
	#endif
	
	float shadow;
	if (other.strength * global.strength <= 0.0) {
		shadow = GetBakedShadow(
			global.shadowMask, other.shadowMaskChannel, abs(other.strength)
		);
	}
	else {
		shadow = GetOtherShadow(other, global, surfaceWS);
		shadow = MixBakedAndRealtimeShadows(
			global, shadow, other.shadowMaskChannel, other.strength
		);
	}
	return shadow;
}
```

<font color="yellowa">The global strength</font>用于确定我们是否可以跳过<font color="DarkOrchid ">realtime shadows</font>的采样，因为我们超出了阴影的距离或者在最大的级联球体之外。然而，级联只适用于<font color="RoyalBlue">directional shadows</font>。它们对其他光线没有意义，因为那些光线有一个固定的位置，因此它们的阴影图不会随着视图移动。

尽管如此，以同样的方式<font color="green">fade </font>所有的阴影是个好主意，否则我们可能会出现屏幕上一些没有<font color="RoyalBlue">directional shadows</font>但有其他阴影的区域。所以我们将对所有东西使用相同的<font color="green"> global shadow strength</font>。

我们必须处理的一个情况是，当我们没有方向性阴影而有其他阴影的时候。当这种情况发生时，没有任何级联，所以它们不应该影响全局阴影强度。

而且我们仍然需要<font color="green">shadow distance fade </font>值。所以让我们把设置级联计数和距离衰减的代码从<font color="blue">Shadows.RenderDirectionShadows</font>移到<font color="red">Shadows.Render</font>，并在适当的时候把<font color="red">级联计数设置为零</font>。

```c#
	public void Render () {
		…
		buffer.SetGlobalInt(
			cascadeCountId,
			shadowedDirLightCount > 0 ? settings.directional.cascadeCount : 0
		);
		float f = 1f - settings.directional.cascadeFade;
		buffer.SetGlobalVector(
			shadowDistanceFadeId, new Vector4(
				1f / settings.maxDistance, 1f / settings.distanceFade,
				1f / (1f - f * f)
			)
		);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}

	void RenderDirectionalShadows () {
		…

		//buffer.SetGlobalInt(cascadeCountId, settings.directional.cascadeCount);
		buffer.SetGlobalVectorArray(
			cascadeCullingSpheresId, cascadeCullingSpheres
		);
		buffer.SetGlobalVectorArray(cascadeDataId, cascadeData);
		buffer.SetGlobalMatrixArray(dirShadowMatricesId, dirShadowMatrices);
		//float f = 1f - settings.directional.cascadeFade;
		//buffer.SetGlobalVector(
		//	shadowDistanceFadeId, new Vector4(
		//		1f / settings.maxDistance, 1f / settings.distanceFade,
		//		1f / (1f - f * f)
		//	)
		//);
		…
	}
```

然后我们必须确保在<font color="green">Shadow.GetShadowData</font>的级联循环之后，全局强度不会被错误地设置为零。

```c#
if (i == _CascadeCount && _CascadeCount > 0) {
		data.strength = 0.0;
	}
```

#### 1.2、[Other Realtime Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#1.2)

方向性的阴影有它们自己的<font color="green">atlas map.</font>

我们将为所有其他有阴影的灯光使用一个单独的<font color="green">atlas map.</font>，并单独计算它们。让我们最多使用<font color="red">十六个有实时阴影</font>的其他灯光。

```c#
	const int maxShadowedDirLightCount = 4, maxShadowedOtherLightCount = 16;
	const int maxCascades = 4;

	…

	int shadowedDirLightCount, shadowedOtherLightCount;
	
	…
	
	public void Setup (…) {
		…
		shadowedDirLightCount = shadowedOtherLightCount = 0;
		useShadowMask = false;
	}
```

这意味着我们可能会出现启用了阴影的灯光，但却不适合放在<font color="green"> atlas</font>里。

哪些灯光不会得到阴影取决于它们在<font color="red">可见光列表</font>中的位置。我们不会为那些被淘汰的灯光保留阴影，但是如果它们有被<font color="DarkOrchid "> baked shadows </font>，我们仍然可以允许它们。

为了实现这一点，首先重构<font color="green">ReserveOtherShadows</font>，使它在灯光没有阴影的时候立即返回。否则，它将检查一个<font color="RoyalBlue">shadow mask channel</font>--默认使用-1，然后总是返回<font color="red"> shadow strength and channel.</font>

```c#
	public Vector4 ReserveOtherShadows (Light light, int visibleLightIndex) {
		if (light.shadows == LightShadows.None || light.shadowStrength <= 0f) {
			return new Vector4(0f, 0f, 0f, -1f);
		}

		float maskChannel = -1f;
		//if (light.shadows != LightShadows.None && light.shadowStrength > 0f) {
		LightBakingOutput lightBaking = light.bakingOutput;
		if (
			lightBaking.lightmapBakeType == LightmapBakeType.Mixed &&
			lightBaking.mixedLightingMode == MixedLightingMode.Shadowmask
		) {
			useShadowMask = true;
			maskChannel = lightBaking.occlusionMaskChannel;
		}
		return new Vector4(
			light.shadowStrength, 0f, 0f,
			maskChannel
		);
			//}
		//}
		//return new Vector4(0f, 0f, 0f, -1f);
	}
```

然后在返回之前，检查增加光的数量是否会超过最大值，或者有没有阴影，但是需要渲染的光。

如果是这样的话，就返回一个<font color="green">negative shadow strength and the mask channel</font>，以便在适当的时候使用<font color="red">baked shadows</font>。

否则就继续增加光照灯数并设置<font color="RoyalBlue">tile index.</font>

```c#
	if (
			shadowedOtherLightCount >= maxShadowedOtherLightCount ||
			!cullingResults.GetShadowCasterBounds(visibleLightIndex, out Bounds b)
		) {
			return new Vector4(-light.shadowStrength, 0f, 0f, maskChannel);
		}

		return new Vector4(
			light.shadowStrength, shadowedOtherLightCount++, 0f,
			maskChannel
		);
```

#### 1.3、[Two Atlases](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#1.3)

因为<font color="green"> directional and other shadows </font>是分开的，我们可以对它们进行不同的配置。在<font color="red">ShadowSettings</font>中为其他阴影添加一个新的配置结构和字段，只包<font color="RoyalBlue"> atlas size and filter</font>，因为级联并不适用。

```c#
[System.Serializable]
	public struct Other {

		public MapSize atlasSize;

		public FilterMode filter;
	}

	public Other other = new Other {
		atlasSize = MapSize._1024,
		filter = FilterMode.PCF2x2
	};
```

<img src="E:\Typora file\SRP\assets\other-shadows-settings.png" alt="img" style="zoom:50%;" />

<center>Settings for other shadows.</center>

为我们的<font color="green">*Lit* shader</font>的<font color="green">CustomLit</font>通道添加一个多编译指令，以支持对其他阴影的<font color="red"> filtering </font>。

```glsl
#pragma multi_compile _ _OTHER_PCF3 _OTHER_PCF5 _OTHER_PCF7
```

并给Shadows添加一个相应的<font color="red"> keyword array - 关键词数组 </font>。

```c#
static string[] otherFilterKeywords = {
		"_OTHER_PCF3",
		"_OTHER_PCF5",
		"_OTHER_PCF7",
	};
```

我们还需要跟踪其他<font color="red"> shadow atlas and matrices</font>的<font color="green">shader property identifiers</font>，另外还有一个数组来保存矩阵。

```c#
	static int
		dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas"),
		dirShadowMatricesId = Shader.PropertyToID("_DirectionalShadowMatrices"),
		otherShadowAtlasId = Shader.PropertyToID("_OtherShadowAtlas"),
		otherShadowMatricesId = Shader.PropertyToID("_OtherShadowMatrices"),
		…;
		
	…
		
	static Matrix4x4[]
		dirShadowMatrices = new Matrix4x4[maxShadowedDirLightCount * maxCascades],
		otherShadowMatrices = new Matrix4x4[maxShadowedOtherLightCount];
```

我们已经用一个<font color="red">矢量的XY分量</font>向GPU发送了<font color="green">directional atlas</font>的尺寸。现在我们还需要发送另一个图集的大小，我们可以把它放在同一个<font color="red">向量的ZW</font>分量中。

将其提升为一个字段，并将全局向量从<font color="green">RenderDirectionalShadows</font>移动设置为<font color="red">Render</font>。然后<font color="RoyalBlue">RenderDirectionalShadows</font>只需要分配给<font color="DarkOrchid "> field</font>的<font color="red">XY分量</font>。

```c#
	Vector4 atlasSizes;
	
	…
	
	public void Render () {
		…
		buffer.SetGlobalVector(shadowAtlasSizeId, atlasSizes);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
	
	void RenderDirectionalShadows () {
		int atlasSize = (int)settings.directional.atlasSize;
		atlasSizes.x = atlasSize;
		atlasSizes.y = 1f / atlasSize;
		…
		//buffer.SetGlobalVector(
		//	shadowAtlasSizeId, new Vector4(atlasSize, 1f / atlasSize)
		//);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

之后，复制<font color="green">RenderDirectionalShadows</font>并将其重命名为<font color="RoyalBlue">RenderOtherShadows</font>。改变它，使其使用正确的<font color="RoyalBlue"> settings, atlas, matrices</font>，并设置正确的尺寸组件。

然后删除其中的级联和剔除球体的代码。同时删除对<font color="green">RenderDirectionalShadows</font>的调用，但保留循环。

```c#
	void RenderOtherShadows () {
		int atlasSize = (int)settings.other.atlasSize;
		atlasSizes.z = atlasSize;
		atlasSizes.w = 1f / atlasSize;
		buffer.GetTemporaryRT(
			otherShadowAtlasId, atlasSize, atlasSize,
			32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap
		);
		buffer.SetRenderTarget(
			otherShadowAtlasId,
			RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store
		);
		buffer.ClearRenderTarget(true, false, Color.clear);
		buffer.BeginSample(bufferName);
		ExecuteBuffer();

		int tiles = shadowedOtherLightCount;
		int split = tiles <= 1 ? 1 : tiles <= 4 ? 2 : 4;
		int tileSize = atlasSize / split;

		for (int i = 0; i < shadowedOtherLightCount; i++) {
			//RenderDirectionalShadows(i, split, tileSize);
		}

		//buffer.SetGlobalVectorArray(
		//	cascadeCullingSpheresId, cascadeCullingSpheres
		//);
		//buffer.SetGlobalVectorArray(cascadeDataId, cascadeData);
		buffer.SetGlobalMatrixArray(otherShadowMatricesId, otherShadowMatrices);
		SetKeywords(
			otherFilterKeywords, (int)settings.other.filter - 1
		);
		//SetKeywords(
		//	cascadeBlendKeywords, (int)settings.directional.cascadeBlend - 1
		//);
		buffer.EndSample(bufferName);
		ExecuteBuffer();
	}
```

现在我们可以在需要时在<font color="green">RenderShadows</font>中渲染<font color="red">directional and other shadows</font>。如果没有其他的阴影，那么我们就需要一个假的纹理，就像方向性阴影一样。我们可以简单地使用<font color="DarkOrchid ">shadow atlas as the dummy.</font>

```c#
	public void Render () {
		if (shadowedDirLightCount > 0) {
			RenderDirectionalShadows();
		}
		else {
			buffer.GetTemporaryRT(
				dirShadowAtlasId, 1, 1,
				32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap
			);
		}
		if (shadowedOtherLightCount > 0) {
			RenderOtherShadows();
		}
		else {
			buffer.SetGlobalTexture(otherShadowAtlasId, dirShadowAtlasId);
		}
		
		…
	}
```

并在清理中释放另一个<font color="green"> shadow atlas</font>，在这种情况下，只有当我们确实得到了一个<font color="green"> shadow atlas</font>时才可以这么做。

```c#
public void Cleanup () {
		buffer.ReleaseTemporaryRT(dirShadowAtlasId);
		if (shadowedOtherLightCount > 0) {
			buffer.ReleaseTemporaryRT(otherShadowAtlasId);
		}
		ExecuteBuffer();
	}
```

#### 1.4、[Rendering Spot Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#1.4)

为了渲染一个聚光灯的阴影，我们需要知道它的<font color="red"> visible light index, slope scale bias, and normal bias</font>。因此，创建一个带有这些字段的<font color="RoyalBlue">ShadowedOtherLight</font>结构，并为它们添加一个数组字段，类似于我们跟踪方向性阴影的数据的方式。

```c#
struct ShadowedOtherLight {
		public int visibleLightIndex;
		public float slopeScaleBias;
		public float normalBias;
	}

	ShadowedOtherLight[] shadowedOtherLights =
		new ShadowedOtherLight[maxShadowedOtherLightCount];
```

在<font color="red">ReserveOtherShadows</font>的结尾处复制相关数据，然后再返回。

```c#
	public Vector4 ReserveOtherShadows (Light light, int visibleLightIndex) {
		…

		shadowedOtherLights[shadowedOtherLightCount] = new ShadowedOtherLight {
			visibleLightIndex = visibleLightIndex,
			slopeScaleBias = light.shadowBias,
			normalBias = light.shadowNormalBias
		};

		return new Vector4(
			light.shadowStrength, shadowedOtherLightCount++, 0f,
			maskChannel
		);
	}
```

然而，此时我们应该意识到，我们不能保证向<font color="green">Lighting</font>中的<font color="red">ReserveOtherShadows</font>发送正确的<font color="RoyalBlue">light index</font>，因为它在传递自己的<font color="DarkOrchid "> other lights</font>。

<font color="green">当有阴影的方向性灯光时，索引会是错误的。</font>

我们通过在灯光设置方法中添加一个正确的可见光指数参数来解决这个问题，并在<font color="RoyalBlue">保留阴影</font>时使用这个参数。为了保持一致性，我们也对方向性灯光这样做。

```c#
void SetupDirectionalLight (
		int index, int visibleIndex, ref VisibleLight visibleLight
	) {
		…
		dirLightShadowData[index] =
			shadows.ReserveDirectionalShadows(visibleLight.light, visibleIndex);
	}

	void SetupPointLight (
		int index, int visibleIndex, ref VisibleLight visibleLight
	) {
		…
		otherLightShadowData[index] =
			shadows.ReserveOtherShadows(light, visibleIndex);
	}

	void SetupSpotLight (
		int index, int visibleIndex, ref VisibleLight visibleLight
	) {
		…
		otherLightShadowData[index] =
			shadows.ReserveOtherShadows(light, visibleIndex);
	}
```

调整<font color="green">SetupLights</font>，使其将可见光指数传递给<font color="red">setup </font>方法。

```c#
	switch (visibleLight.lightType) {
				case LightType.Directional:
					if (dirLightCount < maxDirLightCount) {
						SetupDirectionalLight(
							dirLightCount++, i, ref visibleLight
						);
					}
					break;
				case LightType.Point:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupPointLight(otherLightCount++, i, ref visibleLight);
					}
					break;
				case LightType.Spot:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupSpotLight(otherLightCount++, i, ref visibleLight);
					}
					break;
			}
```

回到<font color="green">Shadows</font>，创建一个<font color="red">RenderSpotShadows</font>方法，它与<font color="green">RenderDirectionalShadows</font>方法的参数相同，只是它不在<font color="DarkOrchid ">multiple tiles</font>循环，没有<font color="green">cascades</font>，也没有<font color="green"> culling factor</font>。

在这种情况下，我们可以使用<font color="DarkOrchid ">CullingResults.ComputeSpotShadowMatricesAndCullingPrimitives</font>，它的作用与<font color="DarkOrchid ">ComputeDirectionalShadowMatricesAndCullingPrimitives</font>类似

它只有<font color="RoyalBlue"> visible light index, matrices, and split data - 可见光指数、矩阵和分割数据</font>作为参数。

CullingResults.[ComputeSpotShadowMatricesAndCullingPrimitives](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.ComputeSpotShadowMatricesAndCullingPrimitives.html)

```c#
	void RenderSpotShadows (int index, int split, int tileSize) {
		ShadowedOtherLight light = shadowedOtherLights[index];
		var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
		cullingResults.ComputeSpotShadowMatricesAndCullingPrimitives(
			light.visibleLightIndex, out Matrix4x4 viewMatrix,
			out Matrix4x4 projectionMatrix, out ShadowSplitData splitData
		);
		shadowSettings.splitData = splitData;
		otherShadowMatrices[index] = ConvertToAtlasMatrix(
			projectionMatrix * viewMatrix,
			SetTileViewport(index, split, tileSize), split
		);
		buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
		buffer.SetGlobalDepthBias(0f, light.slopeScaleBias);
		ExecuteBuffer();
		context.DrawShadows(ref shadowSettings);
		buffer.SetGlobalDepthBias(0f, 0f);
	}
```

在<font color="green">RenderOtherShadows</font>的循环中调用这个方法。

```c#
	for (int i = 0; i < shadowedOtherLightCount; i++) {
			RenderSpotShadows(i, split, tileSize);
		}
```

<img src="E:\Typora file\SRP\assets\spot-shadow-map.png" alt="img" style="zoom: 50%;" />

<center>Shadow atlas for three spot lights.</center>

#### 1.5、[No Pancaking](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#1.5)

阴影现在被渲染为聚光灯，使用的是用于定向阴影的<font color="green">ShadowCaster</font>通道。

这样做很好，除了<font color="green">shadow pancaking - 影子折痕</font>只对正交影子投影有效，用于假设无限远的方向性灯光。

在聚光灯的情况下--它们确实有一个位置--<font color="DarkOrchid ">shadow casters</font>最终可能会部分地落后于灯光的位置。由于我们在这种情况下使用的是透视投影，把顶点夹在近平面上会严重扭曲这种阴影。所以我们应该在不合适的情况下关闭夹持。

我们可以通过一个全局着色器属性来告诉着色器，即我们命名为<font color="red">_ShadowPancaking</font>的着色器是否被激活。追踪它在<font color="green">Shadows</font>中的标识符。

```c#
	static int
		…
		shadowDistanceFadeId = Shader.PropertyToID("_ShadowDistanceFade"),
		shadowPancakingId = Shader.PropertyToID("_ShadowPancaking");
```

在渲染阴影之前，将其设置为1 在<font color="green">RenderDirectionalShadows</font>哪里.

```c#
	buffer.ClearRenderTarget(true, false, Color.clear);
		buffer.SetGlobalFloat(shadowPancakingId, 1f);
		buffer.BeginSample(bufferName);
```

而在<font color="green">RenderOtherShadows</font>中为零。

```c#
	buffer.ClearRenderTarget(true, false, Color.clear);
		buffer.SetGlobalFloat(shadowPancakingId, 0f);
		buffer.BeginSample(bufferName);
```

然后把它作为一个布尔值添加到我们的<font color="green">Lit shader</font>的<font color="green">ShadowCaster</font>通道中，在适当的时候使用它来  **clamp**。

```c#
bool _ShadowPancaking;

Varyings ShadowCasterPassVertex (Attributes input) {
	…

	if (_ShadowPancaking) {
		#if UNITY_REVERSED_Z
			output.positionCS.z = min(
				output.positionCS.z, output.positionCS.w * UNITY_NEAR_CLIP_VALUE
			);
		#else
			output.positionCS.z = max(
				output.positionCS.z, output.positionCS.w * UNITY_NEAR_CLIP_VALUE
			);
		#endif
	}

	output.baseUV = TransformBaseUV(input.baseUV);
	return output;
}
```

#### 1.6、Sampling Spot Shadows

为了对其他阴影进行采样，我们必须调整阴影。首先定义<font color="green">other filter and max shadowed other light count macros</font>。然后添加其他<font color="green"> shadow atlas</font>和其他阴影<font color="green">matrices array</font>。

```c#
#if defined(_OTHER_PCF3)
	#define OTHER_FILTER_SAMPLES 4
	#define OTHER_FILTER_SETUP SampleShadow_ComputeSamples_Tent_3x3
#elif defined(_OTHER_PCF5)
	#define OTHER_FILTER_SAMPLES 9
	#define OTHER_FILTER_SETUP SampleShadow_ComputeSamples_Tent_5x5
#elif defined(_OTHER_PCF7)
	#define OTHER_FILTER_SAMPLES 16
	#define OTHER_FILTER_SETUP SampleShadow_ComputeSamples_Tent_7x7
#endif

#define MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT 4
#define MAX_SHADOWED_OTHER_LIGHT_COUNT 16
#define MAX_CASCADE_COUNT 4

TEXTURE2D_SHADOW(_DirectionalShadowAtlas);
TEXTURE2D_SHADOW(_OtherShadowAtlas);
#define SHADOW_SAMPLER sampler_linear_clamp_compare
SAMPLER_CMP(SHADOW_SAMPLER);

CBUFFER_START(_CustomShadows)
	…
	float4x4 _DirectionalShadowMatrices
		[MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT * MAX_CASCADE_COUNT];
	float4x4 _OtherShadowMatrices[MAX_SHADOWED_OTHER_LIGHT_COUNT];
	…
CBUFFER_END
```

复制<font color="green">SampleDirectionalShadowAtlas</font>和<font color="green">FilterDirectionalShadow</font>，并重新命名和调整它们，以便它们能用于<font color="DarkOrchid ">FilterDirectionalShadow</font>。注意，我们需要在这个版本中使用图集大小向量的另一个分量对。

```c#
float SampleOtherShadowAtlas (float3 positionSTS) {
	return SAMPLE_TEXTURE2D_SHADOW(
		_OtherShadowAtlas, SHADOW_SAMPLER, positionSTS
	);
}

float FilterOtherShadow (float3 positionSTS) {
	#if defined(OTHER_FILTER_SETUP)
		real weights[OTHER_FILTER_SAMPLES];
		real2 positions[OTHER_FILTER_SAMPLES];
		float4 size = _ShadowAtlasSize.wwzz;
		OTHER_FILTER_SETUP(size, positionSTS.xy, weights, positions);
		float shadow = 0;
		for (int i = 0; i < OTHER_FILTER_SAMPLES; i++) {
			shadow += weights[i] * SampleOtherShadowAtlas(
				float3(positions[i].xy, positionSTS.z)
			);
		}
		return shadow;
	#else
		return SampleOtherShadowAtlas(positionSTS);
	#endif
}
```

<font color="green">OtherShadowData</font>结构现在也需要一个<font color="green"> tile index.</font>

```c#
struct OtherShadowData {
	float strength;
	int tileIndex;
	int shadowMaskChannel;
};
```

这是由<font color="green">Light</font>的<font color="red">GetOtherShadowData</font>设置的。

```c#
OtherShadowData GetOtherShadowData (int lightIndex) {
	OtherShadowData data;
	data.strength = _OtherLightShadowData[lightIndex].x;
	data.tileIndex = _OtherLightShadowData[lightIndex].y;
	data.shadowMaskChannel = _OtherLightShadowData[lightIndex].w;
	return data;
}
```

现在我们可以在<font color="green">GetOtherShadow</font>中对阴影图进行采样，而不是总是返回1。它的工作原理与<font color="RoyalBlue">GetCascadedShadow</font>类似，只是没有第二个级联来混合，而且它是一个透视投影，所以我们必须用变换后的位置的XYZ分量除以其W分量。另外，我们还没有一个实用的<font color="green">normal bias</font>，所以我们现在先把它乘以0。

```c#
float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	float3 normalBias = surfaceWS.interpolatedNormal * 0.0;
	float4 positionSTS = mul(
		_OtherShadowMatrices[other.tileIndex],
		float4(surfaceWS.position + normalBias, 1.0)
	);
	return FilterOtherShadow(positionSTS.xyz / positionSTS.w);
}
```

<img src="E:\Typora file\SRP\assets\with-shadows.png" alt="with" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\without-shadows.png" alt="without" style="zoom:50%;" />

<center>Direct spot lighting only, with and without realtime shadows.</center>

#### 1.7、Normal Bias

聚光灯和定向灯一样，都有<font color="green">shadow acne</font>的问题。但由于透视投影的原因，<font color="green"> texel size </font>并不恒定，所以<font color="green">shadow acne</font>也不恒定。离灯越远，<font color="green">shadow acne</font>就越大。

<img src="E:\Typora file\SRP\assets\variable-texel-size.png" alt="img" style="zoom:50%;" />

<center>Texel size increases with distance from light.</center>

<font color="green">Texel size</font>l随着与光平面的距离呈线性增加，而光平面是将世界投射到光前面或后面的平面。

所以我们可以计算<font color="green">texel</font>的大小，从而计算出距离为1时的 <font color="RoyalBlue">normal bias </font>，并将其发送给着色器，在那里我们将把它放大到适当的大小。

在世界空间中，在离光平面的距离为1时，<font color="green"> shadow tile</font>的大小是弧度为一半的<font color="green">spot angle</font>的切线的两倍。

<img src="E:\Typora file\SRP\assets\tile-size-diagram.png" alt="img" style="zoom:50%;" />

<center>World-space tile size derivation.</center>

这与透视投影相匹配，所以距离1的世界空间<font color="green">texel size</font>等于2除以<font color="green">projection scale</font>，对此我们可以使用其矩阵的左上角值。

我们可以用它来计算<font color="RoyalBlue"> normal bias</font>，就像我们对定向光所做的那样，只是我们可以立即将光的<font color="RoyalBlue"> normal bias</font>因素纳入其中，因为没有多个级联。在<font color="red">Shadows.RenderSpotShadows</font>中设置<font color="green">shadow matrix</font>之前做这个。

```c#
	float texelSize = 2f / (tileSize * projectionMatrix.m00);
		float filterSize = texelSize * ((float)settings.other.filter + 1f);
		float bias = light.normalBias * filterSize * 1.4142136f;
		otherShadowMatrices[index] = ConvertToAtlasMatrix(
			projectionMatrix * viewMatrix,
			SetTileViewport(index, split, tileSize), tileScale
		);
```

现在，我们必须将<font color="green">bias</font>的数据发送给Shader。我们稍后需要为<font color="red">per tile</font>发送更多的数据，所以让我们添加一个<font color="RoyalBlue">_OtherShadowTiles</font>矢量阵列着色器属性。为它添加一个标识符和数组到Shadows中，并在<font color="red">RenderOtherShadows</font>中与矩阵一起设置。

```c#
	static int
		…
		otherShadowMatricesId = Shader.PropertyToID("_OtherShadowMatrices"),
		otherShadowTilesId = Shader.PropertyToID("_OtherShadowTiles"),
		…;

	static Vector4[]
		cascadeCullingSpheres = new Vector4[maxCascades],
		cascadeData = new Vector4[maxCascades],
		otherShadowTiles = new Vector4[maxShadowedOtherLightCount];
	
	…
	
	void RenderOtherShadows () {
		…

		buffer.SetGlobalMatrixArray(otherShadowMatricesId, otherShadowMatrices);
		buffer.SetGlobalVectorArray(otherShadowTilesId, otherShadowTiles);
		…
	}
```

创建一个新的<font color="red">SetOtherTileData</font>方法，有一个<font color="RoyalBlue">index and bias.</font>。让它把<font color="green"> bias.</font>放在一个向量的最后一个分量中，然后把它存储在<font color="DarkOrchid "> tile data array.</font>中。

```c#
void SetOtherTileData (int index, float bias) {
		Vector4 data = Vector4.zero;
		data.w = bias;
		otherShadowTiles[index] = data;
	}
```

一旦我们有了<font color="red">Bias</font>，就在<font color="green">RenderSpotShadows</font>中调用它。

```c#
	float bias = light.normalBias * filterSize * 1.4142136f;
		SetOtherTileData(index, bias);
```

然后将另一个<font color="green"> other shadow tile</font>阵列添加到<font color="red">shadow buffer </font>，用它来缩放阴影中的<font color="DarkOrchid ">shadow buffer </font>。

```glsl
CBUFFER_START(_CustomShadows)
	…
	float4x4 _OtherShadowMatrices[MAX_SHADOWED_OTHER_LIGHT_COUNT];
	float4 _OtherShadowTiles[MAX_SHADOWED_OTHER_LIGHT_COUNT];
	float4 _ShadowAtlasSize;
	float4 _ShadowDistanceFade;
CBUFFER_END

…

float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	float4 tileData = _OtherShadowTiles[other.tileIndex];
	float3 normalBias = surfaceWS.interpolatedNormal * tileData.w;
	…
}
```

<img src="E:\Typora file\SRP\assets\normal-bias-constant.png" alt="img" style="zoom:50%;" />

<center>Constant normal bias, set to 1.</center>

在这一点上，我们有一个只在固定距离上正确的<font color="green">normal bias</font>。为了使它与光平面的距离成比例，我们需要知道世界空间的<font color="green">world-space light position and spot direction</font>，所以把它们添加到<font color="red">OtherShadowData</font>中。

```glsl
struct OtherShadowData {
	float strength;
	int tileIndex;
	int shadowMaskChannel;
	float3 lightPositionWS;
	float3 spotDirectionWS;
};
```

让Light把这些值复制给它。因为这些值来自灯光本身，而不是阴影数据，所以在<font color="red">GetOtherShadowData</font>中把它们设置为零，然后在<font color="green">GetOtherLight</font>中复制它们。

```glsl
OtherShadowData GetOtherShadowData (int lightIndex) {
	…
	data.lightPositionWS = 0.0;
	data.spotDirectionWS = 0.0;
	return data;
}

Light GetOtherLight (int index, Surface surfaceWS, ShadowData shadowData) {
	Light light;
	light.color = _OtherLightColors[index].rgb;
	float3 position = _OtherLightPositions[index].xyz;
	float3 ray = position - surfaceWS.position;
	…
	float3 spotDirection = _OtherLightDirections[index].xyz;
	float spotAttenuation = Square(
		saturate(dot(spotDirection, light.direction) *
		spotAngles.x + spotAngles.y)
	);
	OtherShadowData otherShadowData = GetOtherShadowData(index);
	otherShadowData.lightPositionWS = position;
	otherShadowData.spotDirectionWS = spotDirection;
	…
}
```

我们通过在<font color="green">GetOtherShadow</font>中取表面到光的<font color="DarkOrchid "> vector</font>和<font color="green"> spot direction </font>的点积来找到与平面的距离。用它来缩放<font color="green">normal bias.</font>。

```c#
	float4 tileData = _OtherShadowTiles[other.tileIndex];
	float3 surfaceToLight = other.lightPositionWS - surfaceWS.position;
	float distanceToLightPlane = dot(surfaceToLight, other.spotDirectionWS);
	float3 normalBias =
		surfaceWS.interpolatedNormal * (distanceToLightPlane * tileData.w);
```

<img src="E:\Typora file\SRP\assets\normal-bias-variable.png" alt="img" style="zoom:50%;" />

<center>Correct normal bias everywhere.</center>

#### 1.8、Clamped Sampling

我们为定向阴影配置了级联球体，以确保我们最终不会在相应的阴影瓦片之外取样，但我们不能对其他阴影使用同样的方法。在聚光灯的情况下，它们的<font color="green">tiles</font>与它们的<font color="green">cones</font>紧密贴合，所以<font color="green">normal bias and filter size </font>会在锥体边缘接近<font color="green">tile bounds</font>的地方把采样推到<font color="DarkOrchid "> tile edge</font>之外。

<img src="E:\Typora file\SRP\assets\without-clamping.png" alt="img" style="zoom:50%;" />

<center>Shadows from wrong tiles intrude near edges.</center>

解决这个问题的最简单的方法是手动钳制采样，<font color="red">使其保持在瓦片的边界内</font>，就像每个瓦片是它自己独立的纹理一样。这仍然会拉伸边缘的阴影，但不会引入无效的阴影。

调整<font color="green">SetOtherTileData</font>方法，使其也能根据通过新参数提供的<font color="DarkOrchid ">offset and scale </font>来计算和存储<font color="red"> tile bounds</font>。

<font color="green">The tile's minimum texture coordinates </font>是缩放的偏移量，我们将把它存储在数据向量的<font color="red">XY分量</font>中。由于瓦片是正方形的，我们可以把<font color="green">tile's scale</font>在Z分量中，把W留给偏移量就够了。

我们还必须在两个维度上将边界缩小半个texel，以确保采样不会渗出到边缘之外。

```c#
	void SetOtherTileData (int index, Vector2 offset, float scale, float bias) {
		float border = atlasSizes.w * 0.5f;
		Vector4 data;
		data.x = offset.x * scale + border;
		data.y = offset.y * scale + border;
		data.z = scale - border - border;
		data.w = bias;
		otherShadowTiles[index] = data;
	}
```

在<font color="green">RenderSpotShadows</font>中，使用通过<font color="red">SetTileViewport</font>找到的偏移量以及<font color="DarkOrchid ">SetOtherTileData</font>的新参数的<font color="RoyalBlue"> inverse of the split - 分割倒数</font>。

```c#
	Vector2 offset = SetTileViewport(index, split, tileSize);
		SetOtherTileData(index, offset, 1f / split, bias);
		otherShadowMatrices[index] = ConvertToAtlasMatrix(
			projectionMatrix * viewMatrix, offset, split
		);
```

<font color="red">ConverToAtlasMatrix</font>方法也使用<font color="green">inverse of the split</font>，所以我们可以计算一次，并将其传递给两个方法。

```c#
	float tileScale = 1f / split;
		SetOtherTileData(index, offset, tileScale);
		otherShadowMatrices[index] = ConvertToAtlasMatrix(
			projectionMatrix * viewMatrix, offset, tileScale
		);
```

那么<font color="green">ConvertToAtlasMatrix</font>就不需要自己进行除法。

```c#
Matrix4x4 ConvertToAtlasMatrix (Matrix4x4 m, Vector2 offset, float scale) {
		…
		//float scale = 1f / split;
		…
	}
```

这需要<font color="red">RenderDirectionalShadows</font>来代替执行划分，它只需要对所有级联做一次。

```c#
void RenderDirectionalShadows (int index, int split, int tileSize) {
		…
		float tileScale = 1f / split;
		
		for (int i = 0; i < cascadeCount; i++) {
			…
			dirShadowMatrices[tileIndex] = ConvertToAtlasMatrix(
				projectionMatrix * viewMatrix,
				SetTileViewport(tileIndex, split, tileSize), tileScale
			);
			…
		}
	}
```

为了应用边界，在<font color="red">SampleOtherShadowAtlas</font>中为它添加一个<font color="green">float3参数</font>，用它来<font color="DarkOrchid ">clamp the position in shadow tile space - 钳制阴影瓦片空间的位置</font>。

<font color="red">FilterOtherShadows</font>需要同样的参数，这样它就可以把它传递出去。而<font color="green">GetOtherShadow</font>则从<font color="green"> shadow tile space</font>中检索它。

<font color="RoyalBlue">Scipts in Shadow.hlsl</font>

```c#
float SampleOtherShadowAtlas (float3 positionSTS, float3 bounds) {
	positionSTS.xy = clamp(positionSTS.xy, bounds.xy, bounds.xy + bounds.z);
	return SAMPLE_TEXTURE2D_SHADOW(
		_OtherShadowAtlas, SHADOW_SAMPLER, positionSTS
	);
}

float FilterOtherShadow (float3 positionSTS, float3 bounds) {
	#if defined(OTHER_FILTER_SETUP)
		…
		for (int i = 0; i < OTHER_FILTER_SAMPLES; i++) {
			shadow += weights[i] * SampleOtherShadowAtlas(
				float3(positions[i].xy, positionSTS.z), bounds
			);
		}
		return shadow;
	#else
		return SampleOtherShadowAtlas(positionSTS, bounds);
	#endif
}

float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	…
	return FilterOtherShadow(positionSTS.xyz / positionSTS.w, tileData.xyz);
}
```

<img src="E:\Typora file\SRP\assets\with-clamping.png" alt="img" style="zoom:50%;" />

<center>No more shadows from wrong tiles.</center>

### 2、[Point Light Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2)

点光源的阴影和聚光灯的阴影一样。

不同的是，点光源并不局限于一个圆锥体，所以我们需要将它们的阴影渲染到<font color="green">a cube map.</font>上。这是通过分别渲染立方体所有六个面的阴影来实现的。

因此，我们将把一个点灯当作六个灯来处理，以达到实时阴影的目的。它将在<font color="DarkOrchid ">shadow atlas</font>中占据<font color="green">six tiles</font>。这意味着我们最多可以同时支持两个点光源的实时阴影，因为它们会占用16个可用<font color="red">Tiles</font>中的12个。如果少于六块瓦片，一个点灯就不能得到实时阴影。

#### 2.1、[Six Tiles for One Light](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2.1)

首先，我们需要知道我们在渲染阴影时处理的是一个点光源，所以在<font color="red">ShadowedOtherLight</font>中添加一个布尔值来表明这一点。

```c#
	struct ShadowedOtherLight {
		…
		public bool isPoint;
	}
```

检查我们在<font color="red">ReserveOtherShadows</font>中是否有一个点光源。如果有的话，包括这个的新灯光数量将比当前的数量多6个，否则就只多一个。

如果这将超过最大值，那么这个灯最多只能<font color="RoyalBlue"> baked shadows</font>。

如果图集里有足够的空间，那么也可以在返回的<font color="red">shadow data,</font>的第三个分量中存储它是否是一个点光源，以便在着色器中容易检测到点光源。

```c#
public Vector4 ReserveOtherShadows (Light light, int visibleLightIndex) {
		…

		bool isPoint = light.type == LightType.Point;
		int newLightCount = shadowedOtherLightCount + (isPoint ? 6 : 1);
		if (
			newLightCount > maxShadowedOtherLightCount ||
			!cullingResults.GetShadowCasterBounds(visibleLightIndex, out Bounds b)
		) {
			return new Vector4(-light.shadowStrength, 0f, 0f, maskChannel);
		}

		shadowedOtherLights[shadowedOtherLightCount] = new ShadowedOtherLight {
			visibleLightIndex = visibleLightIndex,
			slopeScaleBias = light.shadowBias,
			normalBias = light.shadowNormalBias,
			isPoint = isPoint
		};

		Vector4 data = new Vector4(
			light.shadowStrength, shadowedOtherLightCount,
			isPoint ? 1f : 0f, maskChannel
		);
		shadowedOtherLightCount = newLightCount;
		return data;
	}
```

#### 2.2、[Rendering Point Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2.2)

调整<font color="red">RenderOtherShadows</font>，使其在循环中调用一个新的<font color="red">RenderPointShadows</font>方法或现有的<font color="green">RenderSpotShadows</font>方法，视情况而定。

另外，由于点光源的数量为6，所以要为每种光源类型增加正确的迭代器，而不是仅仅增加它。

```c#
	for (int i = 0; i < shadowedOtherLightCount;) { //i++) {
			if (shadowedOtherLights[i].isPoint) {
				RenderPointShadows(i, split, tileSize);
				i += 6;
			}
			else {
				RenderSpotShadows(i, split, tileSize);
				i += 1;
			}
		}
```

新的<font color="red">RenderPointShadows</font>方法是<font color="RoyalBlue">RenderSpotShadows</font>的翻版，但有两个区别。首先，它必须渲染六次，而不是只有一次，在它的<font color="green"> six tiles</font>中循环。第二，它必须使用<font color="RoyalBlue">ComputePointShadowMatricesAndCullingPrimitives</font>而不是<font color="slategray">ComputeSpotShadowMatricesAndCullingPrimitives</font>。

这个方法需要在光照指数之后有两个额外的参数：一个<font color="DarkOrchid ">CubemapFace index</font>和一个<font color="DarkOrchid ">bias</font>。我们对每个面进行一次渲染，并将<font color="green">bias</font>保留为零。

```c#
void RenderPointShadows (int index, int split, int tileSize) {
		ShadowedOtherLight light = shadowedOtherLights[index];
		var shadowSettings =
			new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
		for (int i = 0; i < 6; i++) {
			cullingResults.ComputePointShadowMatricesAndCullingPrimitives(
				light.visibleLightIndex, (CubemapFace)i, 0f,
				out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix,
				out ShadowSplitData splitData
			);
			shadowSettings.splitData = splitData;
			int tileIndex = index + i;
			float texelSize = 2f / (tileSize * projectionMatrix.m00);
			float filterSize = texelSize * ((float)settings.other.filter + 1f);
			float bias = light.normalBias * filterSize * 1.4142136f;
			Vector2 offset = SetTileViewport(tileIndex, split, tileSize);
			float tileScale = 1f / split;
			SetOtherTileData(tileIndex, offset, tileScale, bias);
			otherShadowMatrices[tileIndex] = ConvertToAtlasMatrix(
				projectionMatrix * viewMatrix, offset, tileScale
			);

			buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
			buffer.SetGlobalDepthBias(0f, light.slopeScaleBias);
			ExecuteBuffer();
			context.DrawShadows(ref shadowSettings);
			buffer.SetGlobalDepthBias(0f, 0f);
		}
	}
```

<img src="E:\Typora file\SRP\assets\point-shadow-map-back-faces.png" alt="img" style="zoom:50%;" />

<center>Shadow atlas for two point lights.</center>

<font color="green">cubemap</font>面的<font color="red"> field of view</font>总是90°，因此距离1的世界空间<font color="green">tile size</font>总是2。这意味着我们可以将<font color="DarkOrchid ">bias </font>的计算从循环中移出来。我们也可以用<font color="green">tile scale.</font>来做这个事情。

```c#
	float texelSize = 2f / tileSize;
		float filterSize = texelSize * ((float)settings.other.filter + 1f);
		float bias = light.normalBias * filterSize * 1.4142136f;
		float tileScale = 1f / split;
		
		for (int i = 0; i < 6; i++) {
			…
			//float texelSize = 2f / (tileSize * projectionMatrix.m00);
			//float filterSize = texelSize * ((float)settings.other.filter + 1f);
			//float bias = light.normalBias * filterSize * 1.4142136f;
			Vector2 offset = SetTileViewport(tileIndex, split, tileSize);
			//float tileScale = 1f / split;
			…
		}
```

#### 2.3、[Sampling Point Shadows](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2.3)

这个想法是，点光源的阴影被存储在一个<font color="green"> cube map</font>中，我们的着色器对其进行采样。

然而，我们将<font color="green"> cube map</font>的面存储为<font color="DarkOrchid "> as tiles in an atlas,</font>，所以我们不能使用标准的<font color="red"> cube map sampling</font>。我们必须自己确定适当的面来取样。要做到这一点，我们需要知道我们是否在处理一个点光源，以及表面到光源的方向。把两者都加到<font color="red">OtherShadowData</font>中。

```c#
struct OtherShadowData {
	float strength;
	int tileIndex;
	bool isPoint;
	int shadowMaskChannel;
	float3 lightPositionWS;
	float3 lightDirectionWS;
	float3 spotDirectionWS;
};
```

在<font color="green">Light</font>中设置两个值。如果另一个灯的<font color="green">Shaodw Data</font>的第三个分量等于1，它就是一个点光源。

```c#
OtherShadowData GetOtherShadowData (int lightIndex) {
	…
	data.isPoint = _OtherLightShadowData[lightIndex].z == 1.0;
	data.lightPositionWS = 0.0;
	data.lightDirectionWS = 0.0;
	data.spotDirectionWS = 0.0;
	return data;
}

Light GetOtherLight (int index, Surface surfaceWS, ShadowData shadowData) {
	…
	otherShadowData.lightPositionWS = position;
	otherShadowData.lightDirectionWS = light.direction;
	otherShadowData.spotDirectionWS = spotDirection;
	…
}
```

接下来，我们必须在<font color="red">GetOtherShadow</font>中调整<font color="green">tile index and light plane </font>，如果是点光源的话。

首先把它们变成变量，最初配置为点光源。让<font color="green"> tile index</font>成为一个<font color="green"> float</font>，因为我们要给它添加一个<font color="green"> offset</font>，这个<font color="green"> offset</font>也定义为一个<font color="green"> float</font>。

```c#
float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	float tileIndex = other.tileIndex;
	float3 lightPlane = other.spotDirectionWS;
	float4 tileData = _OtherShadowTiles[tileIndex];
	float3 surfaceToLight = other.lightPositionWS - surfaceWS.position;
	float distanceToLightPlane = dot(surfaceToLight, lightPlane);
	float3 normalBias =
		surfaceWS.interpolatedNormal * (distanceToLightPlane * tileData.w);
	float4 positionSTS = mul(
		_OtherShadowMatrices[tileIndex],
		float4(surfaceWS.position + normalBias, 1.0)
	);
	return FilterOtherShadow(positionSTS.xyz / positionSTS.w, tileData.xyz);
}
```

如果我们有一个点光源，那么我们必须使用适当的轴对齐平面来代替。我们可以使用<font color="green">CubeMapFaceID</font>函数来找到面的偏移，通过传递给它被<font color="green">negated light direction</font>。

这个函数要么是内在的，要么是在<font color="green">Core RP</font>库中定义的，返回一个浮点。立体图面的顺序是+X, -X, +Y, -Y, +Z, -Z，这与我们渲染它们的方式一致。<font color="green">Add the offset to the tile index.</font>

```c#
float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	float tileIndex = other.tileIndex;
	float3 lightPlane = other.spotDirectionWS;
	if (other.isPoint) {
		float faceOffset = CubeMapFaceID(-other.lightDirectionWS);
		tileIndex += faceOffset;
	}
	…
}
	if (other.isPoint) {

		plane = pointShadowPlanes[CubeMapFaceID(-other.lightDirectionWS)];
	}
```

接下来，我们需要使用一个与光面朝向相匹配的光平面。为它们创建一个静态的常量数组，并使用<font color="red"> face offset to index</font>它。平面法线必须指向与光面朝向相反的方向，就像光点方向指向光线一样。

```c#
static const float3 pointShadowPlanes[6] = {
	float3(-1.0, 0.0, 0.0),
	float3(1.0, 0.0, 0.0),
	float3(0.0, -1.0, 0.0),
	float3(0.0, 1.0, 0.0),
	float3(0.0, 0.0, -1.0),
	float3(0.0, 0.0, 1.0)
};

float GetOtherShadow (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	float tileIndex = other.tileIndex;
	float3 plane = other.spotDirectionWS;
	if (other.isPoint) {
		float faceOffset = CubeMapFaceID(-other.lightDirectionWS);
		tileIndex += faceOffset;
		lightPlane = pointShadowPlanes[faceOffset];
	}
	…
}
```

<center><img src="E:\Typora file\SRP\assets\with-shadows-1677831835622-23.png" alt="with" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\without-shadows-1677831835623-24.png" alt="without" style="zoom:50%;" /></center>
<center>Direct point light only, with and without realtime shadows; no biases.</center>

#### 2.4、[Drawing the Correct Faces](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2.4)

我们现在可以看到点光源的实时阴影。它们似乎没有受到<font color="green">shadow acne</font>的影响，即使是零偏差。

不幸的是，现在光线通过物体漏到了对面非常靠近它们的表面。增加阴影偏差会使情况变得更糟，而且似乎会在靠近其他表面的物体的阴影中开洞。

<img src="E:\Typora file\SRP\assets\normal-bias-3.png" alt="img" style="zoom: 50%;" />

<center>Maximum normal bias 3.</center>

这是因为Unity渲染点光源的阴影的方式发生的。它把它们倒过来画，这就颠覆了三角形的缠绕顺序。

通常情况下，前面的面--从灯光的角度来看--被绘制，但现在后面的面被渲染了。这可以防止大多数的<font color="red">acne </font>，但会带来<font color="green"> introduces light</font>。我们不能停止翻转，但是我们可以通过否定我们从<font color="red">ComputePointShadowMatricesAndCullingPrimitives</font>得到的<font color="DarkOrchid ">view matrix </font>的一行来撤销它。让我们否定其第二行。这在第二时间将图集中的所有内容颠倒过来，从而使所有内容恢复正常。因为该行的第一个分量始终为零，所以我们只需要否定其他三个分量就够了。

```c#
		cullingResults.ComputePointShadowMatricesAndCullingPrimitives(
				light.visibleLightIndex, (CubemapFace)i, fovBias*0,
				out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix,
				out ShadowSplitData splitData
			);
			viewMatrix.m11 = -viewMatrix.m11;
			viewMatrix.m12 = -viewMatrix.m12;
			viewMatrix.m13 = -viewMatrix.m13;
```

<center><img src="E:\Typora file\SRP\assets\normal-shadows-bias-0.png" alt="bias 0" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\normal-shadow-bias-1.png" alt="bias 1" style="zoom:50%;" /></center>

<center>Front-face shadow rendering, normal bias 0 and 1.</center>

在比较阴影贴图时，这如何改变渲染的阴影是最明显的。

<center><img src="E:\Typora file\SRP\assets\point-shadow-map-front-faces.png" alt="front" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\point-shadow-map-back-faces-1677831984236-36.png" alt="back" style="zoom:50%;" /></center>

<center>Front and back versions of the shadow map.</center>

请注意，那些将<font color="green">MeshRenderer</font>的<font color="DarkOrchid ">*Cast Shadows* mode</font>设置为双面的物体不受影响，因为它们的面都不会被剔除。例如，我让所有带有剪辑或透明材质的球体都投下双面阴影，这样它们就显得更有立体感。

<img src="E:\Typora file\SRP\assets\two-sided-sphere-shadows.png" alt="img" style="zoom:50%;" />

<center>Clip and transparent spheres with two-sided shadows.</center>

#### 2.5、[Field of View Bias](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-shadows/#2.5)

由于纹理平面的方向突然发生了90°的变化，所以立方体贴图的面与面之间总是存在着不连续。普通的立方体贴图采样可以在一定程度上隐藏这个问题，因为它可以在面与面之间进行插值，但是我们是从每个片段的<font color="green"> shadow tiles</font>上采样。我们得到了与<font color="green">edge of spot shadow tiles</font>相同的问题，但现在它们并没有被隐藏，因为没有<font color="red">spot attenuation.</font>。

<center><img src="E:\Typora file\SRP\assets\without-fov-bias-with-clamping.png" alt="with claming" style="zoom:50%;" /> <img src="E:\Typora file\SRP\assets\without-fov-bias-without-claming.png" alt="without claming" style="zoom:50%;" /></center>

<center>Discontinuities between faces, with and without tile clamping.</center>

我们可以通过在渲染阴影时增加一点视场-FOV来减少这些<font color="red">artifacts</font>，这样我们就不会在瓦片的边缘以外取样。这就是<font color="green">ComputePointShadowMatricesAndCullingPrimitives</font>的<font color="red">he bias </font>的作用。

我们通过使我们的<font color="green">tile size </font>在离光的距离为1时比2大一点来做到这一点。具体来说，我们在每一边加上<font color="green">normal bias plus </font>加<font color="green">filter size </font>。

然后，一半对应的FOV角度的正切等于1加上<font color="red">bias and filter size.</font>。将其翻倍，转换为度数，减去90°，并将其用于<font color="red">RenderPointShadows</font>中的FOV偏差。

<img src="E:\Typora file\SRP\assets\fov-bias-diagram.png" alt="img" style="zoom:50%;" />

<center>Increasing the world-space tile size.</center>

```c#
	float fovBias =
			Mathf.Atan(1f + bias + filterSize) * Mathf.Rad2Deg * 2f - 90f;
		for (int i = 0; i < 6; i++) {
			cullingResults.ComputePointShadowMatricesAndCullingPrimitives(
				light.visibleLightIndex, (CubemapFace)i, fovBias,
				out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix,
				out ShadowSplitData splitData
			);
			…
		}
```

<img src="E:\Typora file\SRP\assets\with-fov-bias.png" alt="img" style="zoom:67%;" />

<center>With FOV bias.</center>

请注意，这种方法并不完美，因为通过增加<font color="red"> tile size the texel size</font>也会增加。因此，<font color="green"> filter size increases and the normal bias </font>也应该增加，这意味着我们必须再次增加FOV。

然而，这种差异通常很小，我们可以忽略<font color="green"> tile size</font>的增加，除非大的<font color="DarkOrchid ">normal bias</font>和<font color="DarkOrchid ">filter</font>与<font color="green"> small atlas size.</font>结合使用。

