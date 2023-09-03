# 09_Point and Spot Lights.

支持更多的灯光类型，而不仅仅是方向性的。
		包括实时的点光源和聚光灯。
		为点光源和聚光灯烘烤照明和阴影。
		将渲染限制在每个物体最多8个其他灯光。

![img](E:\Typora file\SRP\assets\tutorial-image-1677684850331-42.jpg)

<center>A party of point and spot lights.</center>

### 1、Point Lights

到目前为止，我们只与定向灯合作，因为这些灯影响一切，而且范围无限。其他的灯光类型则不同。它们不是被假定为无限远的，因此它们有一个位置，而且强度也不同。这需要额外的工作来设置和渲染，这就是为什么我们要为它创建单独的代码。我们从点光源开始，它是无限小的点，向所有方向平等地洒下光线。

#### 1.1、Other Light Data

和定向灯一样，我们只能支持有限的其他灯光。场景中经常包含很多不是定向的灯光，因为它们的有效范围是有限的。通常在任何给定的帧中，只有所有其他灯光的一个子集是可见的。

因此，我们所能支持的最大限度适用于单帧，而不是整个场景。如果我们最终有比最大限度更多的可见灯光，那么有些灯光就会被省略掉。Unity根据重要性对可见光列表进行排序，所以只要可见光没有变化，哪些光被省略是一致的。但是如果它们发生了变化--无论是由于摄像机的移动还是其他的变化--这就会导致明显的跳光。

所以我们不希望使用一个太低的最大值。让我们允许最多同时有64个其他灯光，定义为<font color="green">`**Lighting**`.</font>中的另一个常数。

```glsl
const int maxDirLightCount = 4, maxOtherLightCount = 64;
```

就像定向灯一样，对于其他类型的灯光，我们需要向<font color="green">GPU</font>发送灯光数量和灯光颜色。在这种情况下，我们还需要发送灯光的位置。添加<font color="DarkOrchid ">shader property names</font>和<font color="DarkOrchid "> vector array fields</font>，使之成为可能。

```c#
static int
		otherLightCountId = Shader.PropertyToID("_OtherLightCount"),
		otherLightColorsId = Shader.PropertyToID("_OtherLightColors"),
		otherLightPositionsId = Shader.PropertyToID("_OtherLightPositions");

	static Vector4[]
		otherLightColors = new Vector4[maxOtherLightCount],
		otherLightPositions = new Vector4[maxOtherLightCount];
```

在<font color="red">SetupLights</font>中，跟踪<font color="green">其他灯光的数量和方向性灯光的数量</font>。

在对可见光进行循环后，将所有数据发送到<font color="green">GPU</font>。但是，如果我们最终的其他灯光为零，我们就不需要费力地发送数组。另外，现在也可以说只有其他灯光而没有方向性灯光，所以我们也可以跳过发送方向性数组。

我们需要发送确定的灯光数量。

```c#
	void SetupLights () {
		NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
		int dirLightCount = 0, otherLightCount = 0;
		for (int i = 0; i < visibleLights.Length; i++) {
			…
		}

		buffer.SetGlobalInt(dirLightCountId, dirLightCount);
		if (dirLightCount > 0) {
			buffer.SetGlobalVectorArray(dirLightColorsId, dirLightColors);
			buffer.SetGlobalVectorArray(dirLightDirectionsId, dirLightDirections);
			buffer.SetGlobalVectorArray(dirLightShadowDataId, dirLightShadowData);
		}

		buffer.SetGlobalInt(otherLightCountId, otherLightCount);
		if (otherLightCount > 0) {
			buffer.SetGlobalVectorArray(otherLightColorsId, otherLightColors);
			buffer.SetGlobalVectorArray(
				otherLightPositionsId, otherLightPositions
			);
		}
	}
```

在着色器方面，在<font color="green">Light</font>中也定义其他光的最大值和新数据。

```glsl
#define MAX_DIRECTIONAL_LIGHT_COUNT 4
#define MAX_OTHER_LIGHT_COUNT 64

CBUFFER_START(_CustomLight)
	int _DirectionalLightCount;
	float4 _DirectionalLightColors[MAX_DIRECTIONAL_LIGHT_COUNT];
	float4 _DirectionalLightDirections[MAX_DIRECTIONAL_LIGHT_COUNT];
	float4 _DirectionalLightShadowData[MAX_DIRECTIONAL_LIGHT_COUNT];

	int _OtherLightCount;
	float4 _OtherLightColors[MAX_OTHER_LIGHT_COUNT];
	float4 _OtherLightPositions[MAX_OTHER_LIGHT_COUNT];
CBUFFER_END
```

而且我们已经定义了一个<font color="green">GetOtherLightCount</font>函数，我们以后需要使用。

```c#
int GetOtherLightCount () {
	return _OtherLightCount;
}
```

#### 1.2、[Point Light Setup](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#1.2)

在<font color="red">Lighting</font>中创建一个<font color="RoyalBlue">SetupPointLight</font>方法来设置一个点光源的<font color="green">颜色和位置</font>。

给予它和<font color="red">SetupDirectionalLight</font>一样的参数。颜色的设置方式与方向灯相同。位置的设置与方向灯的方向一样，只是我们需要本地到世界矩阵的最后一列而不是第三列。

```C#
	void SetupPointLight (int index, ref VisibleLight visibleLight) {
		otherLightColors[index] = visibleLight.finalColor;
		otherLightPositions[index] = visibleLight.localToWorldMatrix.GetColumn(3);
	}
```

现在我们必须调整<font color="red">SetupLights</font>中的循环，使它能够区分方向性灯光和点状灯光。

一旦我们达到了方向性灯光的最大数量，我们就不应该再结束这个循环。相反，我们要跳过更多的方向性灯光，继续前进。而对于点光源，我们也要做同样的事情，要考虑到其他光源的最大值。让我们用一个开关语句来编程。

```c#
for (int i = 0; i < visibleLights.Length; i++) {
			VisibleLight visibleLight = visibleLights[i];
			//if (visibleLight.lightType == LightType.Directional) {
			//	SetupDirectionalLight(dirLightCount++, ref visibleLight);
			//	if (dirLightCount >= maxDirLightCount) {
			//		break;
			//	}
			//}
			switch (visibleLight.lightType) {
				case LightType.Directional:
					if (dirLightCount < maxDirLightCount) {
						SetupDirectionalLight(dirLightCount++, ref visibleLight);
					}
					break;
				case LightType.Point:
					if (otherLightCount < maxOtherLightCount) {
						SetupPointLight(otherLightCount++, ref visibleLight);
					}
					break;
			}
		}
```

#### 1.3、[Shading](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#1.3)

现在所有支持<font color="DarkOrchid ">点光源的必要数据</font>对<font color="green">Shader</font>来说都是可用的。

为了利用它，我们为<font color="RoyalBlue">Light</font>添加了一个<font color="blue">GetOtherLight</font>函数，参数与<font color="blue">GetDirectionalLight</font>相同。

在这种情况下，光的方向在每个<font color="green">fragment. </font>中都是不同的。我们通过归一化从<font color="green">表面位置到光的光线</font>来找到它。在这一点上我们不支持阴影，所以<font color="DarkOrchid ">attenuation is 1.</font>

```c#
Light GetOtherLight (int index, Surface surfaceWS, ShadowData shadowData) {
	Light light;
	light.color = _OtherLightColors[index].rgb;
	float3 ray = _OtherLightPositions[index].xyz - surfaceWS.position;
	light.direction = normalize(ray);
	light.attenuation = 1.0;
	return light;
}
```

为了应用新的照明，在<font color="green">GetLighting</font>中的方向性灯光的循环之后，为所有其他灯光添加一个循环。

虽然这些循环是分开的，但我们必须为它们的迭代器变量使用不同的名字，否则在某些情况下我们会得到着色器编译器的警告。所以我在第二个循环中使用j而不是i。

```c#
float3 GetLighting (Surface surfaceWS, BRDF brdf, GI gi) {
	ShadowData shadowData = GetShadowData(surfaceWS);
	shadowData.shadowMask = gi.shadowMask;
	
	float3 color = IndirectBRDF(surfaceWS, brdf, gi.diffuse, gi.specular);
	for (int i = 0; i < GetDirectionalLightCount(); i++) {
		Light light = GetDirectionalLight(i, surfaceWS, shadowData);
		color += GetLighting(surfaceWS, brdf, light);
	}

	for (int j = 0; j < GetOtherLightCount(); j++) {
		Light light = GetOtherLight(j, surfaceWS, shadowData);
		color += GetLighting(surfaceWS, brdf, light);
	}
	return color;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/point-lights/point-lights.png" alt="img" style="zoom:50%;" />

<center>Only point lights; no environment lighting.</center>

#### 1.4、[Distance Attenuation](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#1.4)

我们的点光源现在可以使用了，但是它们太亮了。当光线远离它的源头时，它就会散开，变得不那么集中，因此越往后越不亮。

光的强度是

<img src="E:\Typora file\SRP\assets\mylatex20230302_160141.svg" alt="mylatex20230302_160141"  />

<font color="green">i</font>是配置的强度、d是距离，这被称为反平方律。

请注意，这意味着在距离小于1的情况下，强度大于配置。在离光的位置很近的地方会变得非常亮。先前我们推断，我们使用的<font color="DarkOrchid "> final light color</font>代表了从正面照亮的完全白色<font color="RoyalBlue">diffuse surface fragment</font>反射时观察到的数量。这对定向灯来说是真的，但对其他类型的灯来说，它也是专门针对与灯的距离正好为1的<font color="green">fragments</font>。

<img src="E:\Typora file\SRP\assets\distance-attenuation-graph.png" alt="img" style="zoom:80%;" />

<center>Distance attenuation curve.</center>

通过计算光距的平方来应用距离衰减，并将其倒数作为衰减。为了防止潜在的除以零，将平方距离的最小值设置为一个微小的正值。

```c#
	float distanceSqr = max(dot(ray, ray), 0.00001);
	light.attenuation = 1.0 / distanceSqr;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/point-lights/distance-attenuation.png" alt="img" style="zoom:50%;" />

<center>Light fades with distance.</center>

#### 1.5、[Light Range](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#1.5)

虽然现在点光源的强度迅速衰减，但它们的光在理论上仍然影响着一切，尽管它通常是无法被感知的。<font color="green"> Diffuse reflections </font>很快就变得难以察觉，而<font color="green">specular reflections</font>在更远的地方仍然可见。

为了使渲染切实可行，我们将使用一个最大的光照范围，超过这个范围的光照强度将强制为零。

这并不现实，但否则所有的光都会被算作是可见的，不管它们的距离是多少。有了一个额外的范围，点光源就被一个由其位置和范围定义的边界球所包含。

我们不会在球体的边界突然切断光线，相反，我们将通过应用范围衰减来平稳地淡化它。

<font color="RoyalBlue">Unity's Universal RP and lightmapper</font>使用公式

![mylatex20230302_162745](E:\Typora file\SRP\assets\mylatex20230302_162745.svg)

<font color="green">**r**</font>是<font color="DarkOrchid "> light range</font>所以我们也将使用同样的函数。

![img](E:\Typora file\SRP\assets\range-attenuation-graph.png)

<center>Range attenuation curve.</center>

我们可以将范围存储在<font color="red">light position</font>的第四分量中。为了减少<font color="green">Shader</font>中的计算，应该保存<font color="green">1/r*r</font>，再次确保避免被零除。

```c#
	void SetupPointLight (int index, ref VisibleLight visibleLight) {
		otherLightColors[index] = visibleLight.finalColor;
		Vector4 position = visibleLight.localToWorldMatrix.GetColumn(3);
		position.w =
			1f / Mathf.Max(visibleLight.range * visibleLight.range, 0.00001f);
		otherLightPositions[index] = position;
	}
```

然后包括<font color="green">GetOtherLight</font>中的范围衰减。

```c#
	float distanceSqr = max(dot(ray, ray), 0.00001);
	float rangeAttenuation = Square(
		saturate(1.0 - Square(distanceSqr * _OtherLightPositions[index].w))
	);
	light.attenuation = rangeAttenuation / distanceSqr;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/point-lights/range-attenuation.png" alt="img" style="zoom:50%;" />

<center>Range and distance attenuation.</center>

### 2、[Spot Lights](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#2)

我们也会支持点光源。<font color="green">点光源和聚光灯</font>的区别在于，后者的光线被限制在一个圆锥体上。实际上，它是一个被一个有洞的遮挡球体所包围的点光源。孔的大小决定了光锥的大小。

#### 2.1、[Direction](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#2.1)

一个聚光灯有一个方向，也有一个位置，所以为<font color="green">Lighting</font>添加一个着色器属性名称和其他灯的方向<font color="green">array</font>。

```c#
	static int
		otherLightCountId = Shader.PropertyToID("_OtherLightCount"),
		otherLightColorsId = Shader.PropertyToID("_OtherLightColors"),
		otherLightPositionsId = Shader.PropertyToID("_OtherLightPositions"),
		otherLightDirectionsId = Shader.PropertyToID("_OtherLightDirections");

	static Vector4[]
		otherLightColors = new Vector4[maxOtherLightCount],
		otherLightPositions = new Vector4[maxOtherLightCount],
		otherLightDirections = new Vector4[maxOtherLightCount];
```

在<font color="green">SetupLights</font>中向GPU发送新数据。

```c#
	buffer.SetGlobalVectorArray(
				otherLightPositionsId, otherLightPositions
			);
			buffer.SetGlobalVectorArray(
				otherLightDirectionsId, otherLightDirections
			);
```

创建一个<font color="green">SetupSpotLight</font>方法，它是<font color="RoyalBlue">SetupPointLight</font>的副本，只是它也存储了<font color="green"> light direction</font>。我们可以使用本地到世界的矩阵中被的负值第三列来做这个，类似于方向性灯光。

```c#
	void SetupSpotLight (int index, ref VisibleLight visibleLight) {
		otherLightColors[index] = visibleLight.finalColor;
		Vector4 position = visibleLight.localToWorldMatrix.GetColumn(3);
		position.w =
			1f / Mathf.Max(visibleLight.range * visibleLight.range, 0.00001f);
		otherLightPositions[index] = position;
		otherLightDirections[index] =
			-visibleLight.localToWorldMatrix.GetColumn(2);
	}
```

然后在<font color="red">SetupLights</font>循环中加入一个<font color="DarkOrchid "> spot lights</font>的案例。

```c#
			case LightType.Point:
					if (otherLightCount < maxOtherLightCount) {
						SetupPointLight(otherLightCount++, ref visibleLight);
					}
					break;
				case LightType.Spot:
					if (otherLightCount < maxOtherLightCount) {
						SetupSpotLight(otherLightCount++, ref visibleLight);
					}
					break;
```

在着色器方面，将新的数据添加到<font color="green">Light</font>的缓冲区。

```glsl
	float4 _OtherLightPositions[MAX_OTHER_LIGHT_COUNT];
	float4 _OtherLightDirections[MAX_OTHER_LIGHT_COUNT];
```

并在<font color="green">GetOtherLight</font>中应用<font color="DarkOrchid "> spot attenuation</font>。我们首先简单地使用<font color="green">spot light</font>和光线方向的<font color="RoyalBlue">saturated dot</font>。

这将<font color="red">attenuate the light</font>，使其在90°的光斑角度达到零，照亮光线前面的一切。

```c#
	float spotAttenuation =
		saturate(dot(_OtherLightDirections[index].xyz, light.direction));
	light.attenuation = spotAttenuation * rangeAttenuation / distanceSqr;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/spot-lights/spot-lights.png" alt="img" style="zoom:50%;" />

<center>Spot lights.</center>

#### 2.2、Spot Angle

<font color="DarkOrchid ">Spot lights </font>有一个角度来控制其光锥的宽度。这个角度是从它的中间开始测量的，所以一个90°的角度看起来就像我们现在的样子。

除此之外，还有一个单独的内部角度，用来控制光线开始衰减的时间。<font color="RoyalBlue"> Universal RP and lightmapper </font>通过在饱和前对点积进行缩放和添加一些东西，然后对结果进行平方。

具体来说，公式是

<img src="E:\Typora file\SRP\assets\mylatex20230302_193600.svg" alt="mylatex20230302_193600"  />

<img src="E:\Typora file\SRP\assets\mylatex20230302_193858.svg" alt="mylatex20230302_193858" style="zoom:;" />

![mylatex20230302_194030](E:\Typora file\SRP\assets\mylatex20230302_194030.svg)

ri和ro是内角和外角，单位为弧度，<font color="green">d是dot</font>。

![img](E:\Typora file\SRP\assets\spot-angle-attenuation-graph.png)



<center>Angle attenuation with inner 0°, 20°, 45°, 70° and outer 90°.</center>

该函数也可以写成

![mylatex20230302_194356](E:\Typora file\SRP\assets\mylatex20230302_194356.svg)

但可以这样分解，我们可以再<font color="green">**Lighting**</font>中计算<font color="red">a and  b</font>并通过一个新的<font color="green"> spot angles array</font>将它们发送给<font color="RoyalBlue">Shader</font>。所以要定义这个数组和它的属性名称。

```c++
	static int
		otherLightCountId = Shader.PropertyToID("_OtherLightCount"),
		otherLightColorsId = Shader.PropertyToID("_OtherLightColors"),
		otherLightPositionsId = Shader.PropertyToID("_OtherLightPositions"),
		otherLightDirectionsId = Shader.PropertyToID("_OtherLightDirections"),
		otherLightSpotAnglesId = Shader.PropertyToID("_OtherLightSpotAngles");

	static Vector4[]
		otherLightColors = new Vector4[maxOtherLightCount],
		otherLightPositions = new Vector4[maxOtherLightCount],
		otherLightDirections = new Vector4[maxOtherLightCount],
		otherLightSpotAngles = new Vector4[maxOtherLightCount];
```

在<font color="green">SetupLights</font>中把阵列复制到GPU。

```c#
	buffer.SetGlobalVectorArray(
				otherLightDirectionsId, otherLightDirections
			);
			buffer.SetGlobalVectorArray(
				otherLightSpotAnglesId, otherLightSpotAngles
			);
```

并在<font color="green">SetupSpotLight</font>中计算出数值，将它们存储在<font color="RoyalBlue">spot angles array</font>的X和Y分量中。

<font color="DarkOrchid ">outer angle</font>可以通过<font color="red">VisibleLight</font>结构的<font color="green">spotAngle</font>属性获得。然而，对于内角，我们需要首先通过<font color="green">Light</font>属性检索Light游戏对象，而<font color="red">Light</font>又有一个<font color="DarkOrchid ">innerSpotAngle</font>属性。

```c#
	void SetupSpotLight (int index, ref VisibleLight visibleLight) {
		…

		Light light = visibleLight.light;
		float innerCos = Mathf.Cos(Mathf.Deg2Rad * 0.5f * light.innerSpotAngle);
		float outerCos = Mathf.Cos(Mathf.Deg2Rad * 0.5f * visibleLight.spotAngle);
		float angleRangeInv = 1f / Mathf.Max(innerCos - outerCos, 0.001f);
		otherLightSpots[index] = new Vector4(
			angleRangeInv, -outerCos * angleRangeInv
		);
	}
```

回到**Shader**，在<font color="green">Light</font>中添加新的阵列。

```glsl
	float4 _OtherLightDirections[MAX_OTHER_LIGHT_COUNT];
	float4 _OtherLightSpotAngles[MAX_OTHER_LIGHT_COUNT];
```

并在<font color="green">GetOtherLight</font>中调整<font color="DarkOrchid ">聚光灯衰减。</font>

```c#
	float4 spotAngles = _OtherLightSpotAngles[index];
	float spotAttenuation = Square(
		saturate(dot(_OtherLightDirections[index].xyz, light.direction) *
		spotAngles.x + spotAngles.y)
	);
	light.attenuation = spotAttenuation * rangeAttenuation / distanceSqr;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/spot-lights/spot-angle-attenuation.png" alt="img" style="zoom:50%;" />

<center>Angle attenuation in use.</center>

最后，为了确保<font color="red">点光源不受角度衰减</font>计算的影响，将它们的<font color="green"> spot angle</font>值设置为0和1。

```c#
	void SetupPointLight (int index, ref VisibleLight visibleLight) {
		…
		otherLightSpotAngles[index] = new Vector4(0f, 1f);
	}
```

#### 2.3、[Configuring Inner Angles](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#2.3)

聚光灯总是有一个可配置的外角，但在引入<font color="RoyalBlue">Universal RP</font>之前，并不存在一个单独的内角。

因此，灯光的默认检查器并没有暴露出内角。RP可以进一步修改灯光，所以有可能覆盖灯光的默认检查器。这可以通过创建一个扩展了<font color="red">LightEditor</font>的编辑器脚本并赋予它<font color="RoyalBlue">CustomEditorForRenderPipeline</font>属性来实现。

这个属性的第一个参数必须是<font color="green"> Light type</font>。第二个参数必须是我们想要覆盖<font color="green"> inspector</font>的RP资产的类型。

让我们创建这样一个脚本，将其命名为<font color="red">CustomLightEditor</font>，并将其放在<font color="green">Custom RP / Editor</font>文件夹中。同时给它赋予<font color="green">CanEditMultipleObjects</font>，这样它就可以在选择多个灯光时工作。

```c#
using UnityEngine;
using UnityEditor;

[CanEditMultipleObjects]
[CustomEditorForRenderPipeline(typeof(Light), typeof(CustomRenderPipelineAsset))]
public class CustomLightEditor : LightEditor {}
```

为了替换检查器，我们必须重写<font color="red">OnInspectorGUI</font>方法。但是我们要做最小的工作来暴露<font color="RoyalBlue"> inner angle</font>，所以我们首先调用基础方法，像正常一样绘制默认的检查器。

```c#
public override void OnInspectorGUI() {
		base.OnInspectorGUI();
	}
```

之后，我们检查是否只有<font color="RoyalBlue">spot lights</font>被选中。我们可以通过一个方便的名为<font color="red">settings</font>的子类属性来做到这一点，它提供了对<font color="green"> editor selection</font>的序列化属性的访问。

用它来检查我们是否有多个不同的灯光类型，并且类型是<font color="red">LightType.Spot</font>。如果是的话，在设置上调用<font color="DarkOrchid ">DrawInnerAndOuterSpotAngle</font>，在默认的检查器下面添加一个<font color="RoyalBlue"> inner-outer spot angle</font>滑块。之后，调用<font color="red">ApplyModifiedProperties</font>来应用该滑块的任何变化。

```c#
base.OnInspectorGUI();
		if (
			!settings.lightType.hasMultipleDifferentValues &&
			(LightType)settings.lightType.enumValueIndex == LightType.Spot
		)
		{
			settings.DrawInnerAndOuterSpotAngle();
			settings.ApplyModifiedProperties();
		}
```

<img src="E:\Typora file\SRP\assets\inner-outer-spot-angle-slider.png" alt="inspector" style="zoom:50%;" />

<img src="E:\Typora file\SRP\assets\different-inner-angles.png" alt="scene" style="zoom:50%;" />

<center>Different inner angles.</center>

### 3、Baked Light and Shadows

在本教程中，我们不会涉及点光源和聚光灯的实时阴影，但我们现在将支持烘烤这些灯光类型。

#### 3.1、Fully Baked

完全烘烤<font color="RoyalBlue"> point and spot lights</font>仅仅是将它们的模式设置为烘烤的问题。请注意，它们的阴影类型默认设置为无，如果你想让烘焙他们的阴影，请把它改成其他类型。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/baked-light-and-shadows/realtime.png" alt="realtime" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/baked-light-and-shadows/baked-too-bright.png" alt="baked" style="zoom:50%;" />

<center>Realtime and baked, only one point and spot light.</center>

虽然这足以烘烤这些灯光，但事实证明，烘烤后的灯光太亮了。发生这种情况是因为Unity默认使用了一个不正确的光衰，与传统的RP的结果一致。

#### 3.2、[Lights Delegate](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#3.2)

我们可以告诉Unity使用不同的<font color="green">falloff</font>，方法是提供一个委托给一个方法，该方法在Unity在编辑器中执行<font color="green">lightmapping </font>之前应该被调用。

要做到这一点，把<font color="red">CustomRenderPipeline</font>变成一个局部类，并在其构造函数的最后调用一个目前不存在的<font color="DarkOrchid ">InitializeForEditor</font>方法。

```c#
public partial class CustomRenderPipeline : RenderPipeline {

	…

	public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		ShadowSettings shadowSettings
	) {
		…
		InitializeForEditor();
	}

	…
}
```

然后为它创建另一个编辑器专用的局部类--就像<font color="green">CameraRenderer</font>那样--为新方法定义一个<font color="DarkOrchid ">dummy</font>。

命名为<font color="RoyalBlue">CustomRenderPipeline.Editor</font>

除了<font color="green">UnityEngine</font>命名空间，我们还需要使用<font color="green">Unity.Collections</font>和<font color="green">UnityEngine.Experimental.GlobalIllumination</font>。这将导致LightType的类型冲突，所以明确使用<font color="red">UnityEngine.LightType。</font>

```c#
using Unity.Collections;
using UnityEngine;
using UnityEngine.Experimental.GlobalIllumination;
using LightType = UnityEngine.LightType;

public partial class CustomRenderPipeline {

	partial void InitializeForEditor ();
}
```

对于编辑器，我们必须覆盖<font color="green">lightmapper</font>如何设置它的灯光数据。

这是通过为它提供一个方法的委托来实现的，该方法将数据从输入的<font color="green"> `Light` array </font>转移到<font color="green">NativeArray<<font color="green">LightDataGI</font>></font>输出。

这个委托的类型是<font color="DarkOrchid ">Lightmapping.RequestLightsDelegate</font>，我们将用一个<font color="red">lambda</font>表达式来定义这个方法，因为我们在其他地方不需要它。

[Lightmapping](https://docs.unity3d.com/ScriptReference/30_search.html?q=Lightmapping)

[RequestLightsDelegate](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.Lightmapping.RequestLightsDelegate.html) 在将灯光转换为烘焙后端可以理解的形式时调用委托。

委托函数生成的输出。应该跳过的灯光必须添加到输出中，并使用 LightDataGI 结构上的 InitNoBake 进行初始化。

```c#
partial void InitializeForEditor ();
	
#if UNITY_EDITOR

	static Lightmapping.RequestLightsDelegate lightsDelegate =
		(Light[] lights, NativeArray<LightDataGI> output) => {};

#endif
```

我们必须为每个灯配置一个<font color="green">LightDataGI</font>结构，并将其添加到输出中。

我们必须为每个灯光类型使用特殊的代码，所以在循环中使用开关语句。默认情况下，我们用灯光数据的<font color="RoyalBlue"> light's instance ID</font>调用<font color="green">InitNoBake</font>，它<font color="red">指示Unity不烘烤灯光。</font>

[LightDataGI](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.LightDataGI.html)

```c#
static Lightmapping.RequestLightsDelegate lightsDelegate =
		(Light[] lights, NativeArray<LightDataGI> output) => {
			var lightData = new LightDataGI();
			for (int i = 0; i < lights.Length; i++) {
				Light light = lights[i];
				switch (light.type) {
					default:
						lightData.InitNoBake(light.GetInstanceID());
						break;
				}
				output[i] = lightData;
			}
		};
```

接下来，我们必须为每个支持的灯光类型构建一个专门的<font color="green">light struct</font>，以<font color="DarkOrchid ">灯光和对该结构的引用为参数</font>调用<font color="green">LightmapperUtils.Extract</font>，然后对<font color="red">灯光数据</font>调用<font color="RoyalBlue">Init，</font>通过引用传递该结构。对<font color="blue">directional, point, spot, and area lights.</font>都要这样做。



[LightmapperUtils](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.LightmapperUtils.html) 用于将 Unity Lights 转换为烘焙后端识别的灯光类型的实用程序类。

[LightmapperUtils.Extract](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.LightmapperUtils.Extract.html) 从灯光中提取信息。

```c#
		switch (light.type) {
					case LightType.Directional:
						var directionalLight = new DirectionalLight();
						LightmapperUtils.Extract(light, ref directionalLight);
						lightData.Init(ref directionalLight);
						break;
					case LightType.Point:
						var pointLight = new PointLight();
						LightmapperUtils.Extract(light, ref pointLight);
						lightData.Init(ref pointLight);
						break;
					case LightType.Spot:
						var spotLight = new SpotLight();
						LightmapperUtils.Extract(light, ref spotLight);
						lightData.Init(ref spotLight);
						break;
					case LightType.Area:
						var rectangleLight = new RectangleLight();
						LightmapperUtils.Extract(light, ref rectangleLight);
						lightData.Init(ref rectangleLight);
						break;
					default:
						lightData.InitNoBake(light.GetInstanceID());
						break;
				}
```

我们不支持<font color="green"> realtime area lights</font>，所以如果它们存在的话，让我们强制它们的灯光模式为烘焙。

```c#
		case LightType.Area:
						var rectangleLight = new RectangleLight();
						LightmapperUtils.Extract(light, ref rectangleLight);
						rectangleLight.mode = LightMode.Baked;
						lightData.Init(ref rectangleLight);
						break;
```

这只是我们不得不加入的模板代码。这一切的重点是，我们现在可以将所有灯光数据的衰减类型设置为<font color="red">FalloffType.InverseSquared</font>。

[FalloffType](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.FalloffType.html) 用于烘焙的可用衰减模型。

```glsl
	lightData.falloff = FalloffType.InverseSquared;
				output[i] = lightData;
```

为了让Unity调用我们的代码，创建一个<font color="green">InitializeForEditor</font>的<font color="RoyalBlue">editor version - 编辑器版本</font>，调用<font color="red">Lightmapping.SetDelegate</font>，将我们的委托作为参数。

[SetDelegate](https://docs.unity3d.com/ScriptReference/Experimental.GlobalIllumination.Lightmapping.SetDelegate.html) 设置一个委托，将灯光列表转换为传递给烘焙后端的 <font color="green">LightDataGI</font> 结构列表。必须再次调用 ResetDelegate 来重置。

```c#
partial void InitializeForEditor ();
	
#if UNITY_EDITOR

	partial void InitializeForEditor () {
		Lightmapping.SetDelegate(lightsDelegate);
	}
```

当我们的管道被配置时，我们还必须<font color="red">清理和重置委托</font>。这可以通过覆盖Dispose方法来实现，让它调用其基本实现，然后调用<font color="RoyalBlue">Lightmapping.ResetDelegate</font>。

```c#
	partial void InitializeForEditor () {
		Lightmapping.SetDelegate(lightsDelegate);
	}

	protected override void Dispose (bool disposing) {
		base.Dispose(disposing);
		Lightmapping.ResetDelegate();
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/baked-light-and-shadows/baked-correct-falloff.png" alt="img" style="zoom:50%;" />

<center>Baked with correct falloff.</center>

不幸的是，Unity 2019.2的lightmapper不支持自定义聚光灯的内部衰减角度。可以设置内部聚光灯角度，但它会被忽略。

#### 3.3、[Shadow Mask](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#3.3)

<font color="DarkOrchid ">point and spot lights</font>的阴影也可以通过设置它们的模式为混合而被烘烤到<font color="green">shadow mask,</font>中。

每个灯都有一个通道，就像定向灯一样。但是因为它们的范围有限，所以只要它们不重叠，就可以让多个灯光使用同一个通道。因此，<font color="green">shadow mask,</font>可以支持任意数量的灯光，但每个文本框最多只能有四个。

如果多个灯光在试图使用同一个通道时最终发生了重叠，那么最不重要的灯光将被强制转为<font color="DarkOrchid ">*Baked* mode</font>，直到不再有冲突。

<img src="E:\Typora file\SRP\assets\shadow-mask.png" alt="img" style="zoom:50%;" />

<center>Shadow mask with a point and a spot light.</center>

要对点光源和聚光灯使用<font color="green">Shadow Mask</font>，可以给<font color="red">Shadows</font>添加一个<font color="RoyalBlue">ReserveOtherShadows</font>方法。它的工作原理与<font color="RoyalBlue">ReserveDirectionalShadows</font>类似，只是我们只关心<font color="green">Shadow Mask</font>模式，只需要配置<font color="DarkOrchid "> shadow strength and mask channel.</font>

```c#
public Vector4 ReserveOtherShadows (Light light, int visibleLightIndex) {
		if (light.shadows != LightShadows.None && light.shadowStrength > 0f) {
			LightBakingOutput lightBaking = light.bakingOutput;
			if (
				lightBaking.lightmapBakeType == LightmapBakeType.Mixed &&
				lightBaking.mixedLightingMode == MixedLightingMode.Shadowmask
			) {
				useShadowMask = true;
				return new Vector4(
					light.shadowStrength, 0f, 0f,
					lightBaking.occlusionMaskChannel
				);
			}
		}
		return new Vector4(0f, 0f, 0f, -1f);
	}
```

为<font color="green">Lighting</font>的阴影数据添加着色器属性名称和阵列。

```c#
static int
		otherLightCountId = Shader.PropertyToID("_OtherLightCount"),
		otherLightColorsId = Shader.PropertyToID("_OtherLightColors"),
		otherLightPositionsId = Shader.PropertyToID("_OtherLightPositions"),
		otherLightDirectionsId = Shader.PropertyToID("_OtherLightDirections"),
		otherLightSpotAnglesId = Shader.PropertyToID("_OtherLightSpotAngles"),
		otherLightShadowDataId = Shader.PropertyToID("_OtherLightShadowData");

	static Vector4[]
		otherLightColors = new Vector4[maxOtherLightCount],
		otherLightPositions = new Vector4[maxOtherLightCount],
		otherLightDirections = new Vector4[maxOtherLightCount],
		otherLightSpotAngles = new Vector4[maxOtherLightCount],
		otherLightShadowData = new Vector4[maxOtherLightCount];
```

在<font color="red">SetupLights</font>中把它发送给GPU。

```c#
	buffer.SetGlobalVectorArray(
				otherLightSpotAnglesId, otherLightSpotAngles
			);
			buffer.SetGlobalVectorArray(
				otherLightShadowDataId, otherLightShadowData
			);
```

并在<font color="RoyalBlue">SetupPointLight和SetupSpotLight</font>中配置数据。

```glsl
	void SetupPointLight (int index, ref VisibleLight visibleLight) {
		…
		Light light = visibleLight.light;
		otherLightShadowData[index] = shadows.ReserveOtherShadows(light, index);
	}

	void SetupSpotLight (int index, ref VisibleLight visibleLight) {
		…
		otherLightShadowData[index] = shadows.ReserveOtherShadows(light, index);
	}
```

在<font color="green">Shader</font>方面，为<font color="green">Shadows</font>添加一个<font color="RoyalBlue">OtherShadowData</font>结构和<font color="RoyalBlue">GetOtherShadowAttenuation</font>函数。我们再次使用与<font color="red">directional shadows</font>相同的方法，只是我们只有<font color="DarkOrchid ">strength and mask channel</font>。如果强度是正的，那么我们总是调用<font color="red">GetBakedShadow</font>，否则就没有阴影了。

```c#
struct OtherShadowData {
	float strength;
	int shadowMaskChannel;
};

float GetOtherShadowAttenuation (
	OtherShadowData other, ShadowData global, Surface surfaceWS
) {
	#if !defined(_RECEIVE_SHADOWS)
		return 1.0;
	#endif
	
	float shadow;
	if (other.strength > 0.0) {
		shadow = GetBakedShadow(
			global.shadowMask, other.shadowMaskChannel, other.strength
		);
	}
	else {
		shadow = 1.0;
	}
	return shadow;
}
```

在<font color="green"> *Light*</font>中，添加<font color="DarkOrchid ">shadow data</font>，并将其纳入<font color="RoyalBlue">GetOtherLight</font>中的<font color="green">GetOtherLight</font>。

```c#
CBUFFER_START(_CustomLight)
	…
	float4 _OtherLightShadowData[MAX_OTHER_LIGHT_COUNT];
CBUFFER_END

…			

OtherShadowData GetOtherShadowData (int lightIndex) {
	OtherShadowData data;
	data.strength = _OtherLightShadowData[lightIndex].x;
	data.shadowMaskChannel = _OtherLightShadowData[lightIndex].w;
	return data;
}

Light GetOtherLight (int index, Surface surfaceWS, ShadowData shadowData) {
	…
	
	OtherShadowData otherShadowData = GetOtherShadowData(index);
	light.attenuation =
		GetOtherShadowAttenuation(otherShadowData, shadowData, surfaceWS) *
		spotAttenuation * rangeAttenuation / distanceSqr;
	return light;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/baked-light-and-shadows/mixed-mode.png" alt="img" style="zoom:50%;" />

<center>Point and spot light with baked shadows.</center>

### 4、[Lights Per Object](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#4)

目前，所有可见的灯光都要对每个被渲染的片段进行评估。

这对方向性的灯光来说是很好的，但对其他不在片段范围内的灯光来说是不必要的工作。通常情况下，每个点或聚光灯只影响到所有片段中的一小部分，所以有很多工作是白做的，这会大大影响性能。为了以良好的性能支持许多灯光，我们必须以某种方式减少每个片段所评估的灯光数量。这方面有多种方法，其中最简单的是使用Unity的<font color="green">per-object light indices</font>

这个想法是，<font color="red">Unity确定哪些灯光影响了每个物体</font>，并将这些信息发送给GPU。

我们可以在渲染每个物体时只评估相关的灯光，而忽略其他的。因此，灯光是在每个物体的基础上确定的，而不是每个片段。这对小物体来说通常很好，但对大物体来说并不理想，因为如果一个灯光只影响到物体的一小部分，它将被评估到整个表面。另外，每个物体可以影响的灯光数量是有限制的，所以大型物体更容易缺少一些灯光。

因为每个物体的光照指数并不理想，可能会遗漏一些光照，所以我们会把它变成可选项。这样一来，也就可以很容易地比较视觉效果和性能。

#### 4.1、[Per-Object Light Data](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#4.1)

在<font color="red">CameraRenderer.DrawVisibleGeometry</font>中添加一个布尔参数，以指示是否应使用<font color="green"> lights-per-object</font>模式。如果是这样，请为绘图设置的每个物体数据启用<font color="green">PerObjectData.LightData和PerObjectData.LightIndices</font>标志。

```c#
void DrawVisibleGeometry (
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject
	) {
		PerObjectData lightsPerObjectFlags = useLightsPerObject ?
			PerObjectData.LightData | PerObjectData.LightIndices :
			PerObjectData.None;
		var sortingSettings = new SortingSettings(camera) {
			criteria = SortingCriteria.CommonOpaque
		};
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		) {
			enableDynamicBatching = useDynamicBatching,
			enableInstancing = useGPUInstancing,
			perObjectData =
				PerObjectData.ReflectionProbes |
				PerObjectData.Lightmaps | PerObjectData.ShadowMask |
				PerObjectData.LightProbe | PerObjectData.OcclusionProbe |
				PerObjectData.LightProbeProxyVolume |
				PerObjectData.OcclusionProbeProxyVolume |
				lightsPerObjectFlags
		};
		…
	}
```

同样的参数必须被添加到<font color="red">Render</font>中，所以它可以被传递给<font color="RoyalBlue">DrawVisibleGeometry</font>。

```c#
	public void Render (
		ScriptableRenderContext context, Camera camera,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings
	) {
		…
		DrawVisibleGeometry(
			useDynamicBatching, useGPUInstancing, useLightsPerObject
		);
		…
	}
```

而且我们还必须在<font color="green">CustomRenderPipeline</font>中跟踪并传递模式，就像其他布尔选项一样。

```c#
	bool useDynamicBatching, useGPUInstancing, useLightsPerObject;

	ShadowSettings shadowSettings;

	public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher,
		bool useLightsPerObject, ShadowSettings shadowSettings
	) {
		this.shadowSettings = shadowSettings;
		this.useDynamicBatching = useDynamicBatching;
		this.useGPUInstancing = useGPUInstancing;
		this.useLightsPerObject = useLightsPerObject;
		…
	}

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings
			);
		}
	}
```

最后，给<font color="red">CustomRenderPipelineAsset</font>添加切换选项。

```c#
[SerializeField]
	bool
		useDynamicBatching = true,
		useGPUInstancing = true,
		useSRPBatcher = true,
		useLightsPerObject = true;

	[SerializeField]
	ShadowSettings shadows = default;

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows
		);
	}
```

<img src="E:\Typora file\SRP\assets\lights-per-object-toggle.png" alt="img" style="zoom:50%;" />

<center>Lights per object enabled.</center>

#### 4.2、[Sanitizing Light Indices](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#4.2)

Unity简单地创建了一个每个物体的所有<font color="green">活动灯光的列表</font>，大致按其重要性排序。

这个列表包括所有的灯光，不管它们的可见度如何，也包括方向性的灯光。我们必须对这些列表进行<font color="RoyalBlue">sanitize - 清洁</font>，以便只保留可见的非方向性灯光的索引。

我们在<font color="red">Lighting.SetupLights</font>中这样做，所以在该方法中添加一个<font color="RoyalBlue">lights-per-object</font>参数，并在<font color="blue">Lighting.Setup</font>中传递该参数。

```c#
	public void Setup (
		ScriptableRenderContext context, CullingResults cullingResults,
		ShadowSettings shadowSettings, bool useLightsPerObject
	) {
		…
		SetupLights(useLightsPerObject);
		…
	}

	…

	void SetupLights (bool useLightsPerObject) { … }
```

然后在<font color="green">CameraRenderer.Render</font>中添加模式作为<font color="RoyalBlue">Setup</font>的参数。

```c#
	lighting.Setup(
			context, cullingResults, shadowSettings, useLightsPerObject
		);
```

在<font color="red">Lighting.SetupLights</font>中，在我们循环到可见光之前，从剔除的结果中检索出<font color="red">light index map </font>。这是通过调用<font color="green">GetLightIndexMap</font>和<font color="green">Allocator.Temp</font>作为参数来完成的，它给了我们一个临时的<font color="red">NativeArray<<font color="green">int</font>></font>，其中包含了<font color="RoyalBlue"> light indices - 灯光目录</font>，与场景中的<font color="DarkOrchid ">visible light indices </font>和所有其他活动灯光相匹配。

[CullingResults](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.html).GetLightIndexMap 

[NativeArray](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html)<int> 从VisibleLight指数映射到内部每个物体的光列表指数的数组。

[Allocator](https://docs.unity3d.com/ScriptReference/Unity.Collections.Allocator.html) 分配器

```c#
		NativeArray<int> indexMap =
			cullingResults.GetLightIndexMap(Allocator.Temp);
		NativeArray<VisibleLight> visibleLights = cullingResults.visibleLights;
```

我们只需要在每个对象使用灯光时检索这些数据。由<font color="green"> native array</font>是一个结构，我们将其初始化为默认值，否则就不会分配任何东西。

```c#
	NativeArray<int> indexMap = useLightsPerObject ?
			cullingResults.GetLightIndexMap(Allocator.Temp) : default;
```

我们只需要包括<font color="DarkOrchid ">point and spot lights</font>的<font color="green">indices</font>，其他的灯都应该被跳过。我们通过将所有其他灯光的<font color="green"> indices</font>设置为-1来向Unity传达这一点。

我们还必须改变其余灯光的<font color="green"> indices</font>，使其与我们的相匹配。只有在我们检索到<font color="green"> map.</font>的情况下才设置新的索引。

```c#
	for (int i = 0; i < visibleLights.Length; i++) {
			int newIndex = -1;
			VisibleLight visibleLight = visibleLights[i];
			switch (visibleLight.lightType) {
				…
				case LightType.Point:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupPointLight(otherLightCount++, ref visibleLight);
					}
					break;
				case LightType.Spot:
					if (otherLightCount < maxOtherLightCount) {
						newIndex = otherLightCount;
						SetupSpotLight(otherLightCount++, ref visibleLight);
					}
					break;
			}
			if (useLightsPerObject) {
				indexMap[i] = newIndex;
			}
		}
```

我们还必须消除所有<font color="red">不可见的灯光的索引</font>。如果我们在每个物体上使用灯光，那么就在第一个循环之后继续进行第二个循环来完成这个工作。

```c#
	int i;
		for (i = 0; i < visibleLights.Length; i++) {
			…
		}

		if (useLightsPerObject) {
			for (; i < indexMap.Length; i++) {
				indexMap[i] = -1;
			}
		}
```

当我们完成后，我们必须将调整后的<font color="green"> index map </font>送回Unity，在剔除结果上调用<font color="red">SetLightIndexMap</font>。之后就不再需要索引图了，所以我们应该通过对它调用<font color="DarkOrchid ">Dispose</font>来删除它。

[CullingResults](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.html).[SetLightIndexMap](https://docs.unity3d.com/ScriptReference/Rendering.CullingResults.SetLightIndexMap.html)

如果 RenderPipeline 对 VisibleLight 列表进行排序或以其他方式修改，则需要重新映射索引才能正确使用每个对象的灯光列表。

```c#
		if (useLightsPerObject) {
			for (; i < indexMap.Length; i++) {
				indexMap[i] = -1;
			}
			cullingResults.SetLightIndexMap(indexMap);
			indexMap.Dispose();
		}
```

最后，当使用每个物体的灯光时，我们将使用不同的<font color="green">shader variant </font>。我们通过启用或禁用<font color="green">_LIGHTS_PER_OBJECT</font>着色器关键字来发出信号，视情况而定。

```c#
	static string lightsPerObjectKeyword = "_LIGHTS_PER_OBJECT";
	
	…
	
	void SetupLights (bool useLightsPerObject) {
		…

		if (useLightsPerObject) {
			for (; i < indexMap.Length; i++) {
				indexMap[i] = -1;
			}
			cullingResults.SetLightIndexMap(indexMap);
			indexMap.Dispose();
			Shader.EnableKeyword(lightsPerObjectKeyword);
		}
		else {
			Shader.DisableKeyword(lightsPerObjectKeyword);
		}
		
		…
	}
```

#### 4.3、[Using the Indices](https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/#4.3)

为了使用<font color="green"> light indices</font>，在我们的<font color="DarkOrchid ">Lit</font>着色器的<font color="red">CustomLit</font>通道上添加相关的多编译pragma。

```c#
			#pragma multi_compile _ _LIGHTS_PER_OBJECT
```

所需的数据是<font color="green">UnityPerDraw</font>缓冲区的一部分，由两个必须在<font color="RoyalBlue">unity_WorldTransformParams</font>之后直接定义的<font color="green">real4</font>值组成。

首先是<font color="red">unity_LightData</font>，它包含了其Y分量的灯光数量。之后是<font color="red">unity_LightIndices</font>，它是一个长度为2的数组。这两个向量的每个通道都包含一个灯光索引，所以每个物体最多支持八个。

```c#
	real4 unity_WorldTransformParams;

	real4 unity_LightData;
	real4 unity_LightIndices[2];
```

如果定义了<font color="red">_LIGHTS_PER_OBJECT</font>，在<font color="green">GetLighting</font>中使用另一个循环来处理其他灯光。

在这种情况下，灯光的数量是通过<font color="RoyalBlue">unity_LightData.y</font>找到的，灯光索引必须从<font color="green">unity_LightIndices</font>的适当元素和分量中获取。我们可以通过迭代器除以4得到正确的向量，并通过 取4余数 得到正确的分量。

```glsl
	#if defined(_LIGHTS_PER_OBJECT)
		for (int j = 0; j < unity_LightData.y; j++) {
			int lightIndex = unity_LightIndices[j / 4][j % 4];
			Light light = GetOtherLight(lightIndex, surfaceWS, shadowData);
			color += GetLighting(surfaceWS, brdf, light);
		}
	#else
		for (int j = 0; j < GetOtherLightCount(); j++) {
			Light light = GetOtherLight(j, surfaceWS, shadowData);
			color += GetLighting(surfaceWS, brdf, light);
		}
	#endif
```

然而，虽然最多只有<font color="red">八个光照指数</font>，但提供的光照数并没有考虑到这个限制。所以我们必须明确地将循环限制在八个迭代。

```glsl
		for (int j = 0; j < min(unity_LightData.y, 8); j++) { … }
```

在这一点上，<font color="red"> shader compiler</font>可能会抱怨整数除法和modulo操作很慢，至少在为D3D编译时是这样。而<font color="RoyalBlue">unsigned</font>的等价操作则更有效率。我们可以通过在执行操作时将j转换为uint来表明值的符号可以被忽略。

```glsl
	int lightIndex = unity_LightIndices[(uint)j / 4][(uint)j % 4];
```

<img src="E:\Typora file\SRP\assets\all-lights.png" alt="all" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/point-and-spot-lights/lights-per-object/max-8-per-object.png" alt="max 8" style="zoom:50%;" />

<center>Lights-per-object disabled and enabled.</center>

请注意，在启用每个对象的灯光的情况下，GPU实例化的效率较低，因为只有那些灯光数量和索引列表相匹配的对象才会被分组。SRP批处理程序不受影响，因为每个对象仍然得到它自己的优化绘制调用。

