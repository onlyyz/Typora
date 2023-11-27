# 13_Color Grading

执行颜色分级。
		复制多个URP/HDRP颜色分级工具。
		使用颜色LUT。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/tutorial-image.jpg)

<center>Tweaking colors to create a mood.</center>

### 1、Color Adjustments

目前，我们只应用色调映射到最终的图像，带来HDR颜色在可见的LDR范围内。但这并不是调整图像颜色的唯一原因。视频、照片和数字图像的颜色调整大致有三个步骤。

首先是色彩校正，它的目的是使图像与我们观察场景时看到的图像相匹配，以弥补介质的限制。

其次是颜色分级，这是关于实现与原始场景不匹配的理想外观或感觉，不需要逼真。这两个步骤通常合并为一个颜色分级步骤。

之后是色调映射，将HDR颜色映射到显示范围。

只有色调映射应用的图像往往变得不那么丰富多彩，除非它非常明亮。ACES增加了一些深色的对比度，但它不能代替颜色分级。本教程以中性色调映射为基础。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/without-adjustments.png" alt="img" style="zoom:50%;" />

<center>Image without color adjustments, neutral tone mapping.</center>

#### [1.1、Color Grading Before Tone Mapping](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.1)

<font color="red">Color grading</font>发生在<font color="DarkOrchid ">tone mapping</font>之前。在<font color="red">PostFXStackPasses</font>中添加一个函数，在色调映射之前。最初只让它将颜色成分限制在60。

```glsl
float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	return color;
}
```

在<font color="DarkOrchid "> tone mapping passes</font>中调用这个函数，而不是在那里限制颜色。还可以添加一个新的通道，用于没有色调映射，但有调色功能。

```glsl
float4 ToneMappingNonePassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ColorGrade(color.rgb);
	return color;
}

float4 ToneMappingACESPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ColorGrade(color.rgb);
	color.rgb = AcesTonemap(unity_to_ACES(color.rgb));
	return color;
}

float4 ToneMappingNeutralPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ColorGrade(color.rgb);
	color.rgb = NeutralTonemap(color.rgb);
	return color;
}

float4 ToneMappingReinhardPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ColorGrade(color.rgb);
	color.rgb /= color.rgb + 1.0;
	return color;
}
```

在着色器和<font color="green">PostFXStack.Pass</font>枚举中添加相同的传递，在其他色调映射传递之前。然后调整<font color="red">PostFXStack.DoToneMapping</font>，使<font color="green">None</font>模式使用它自己的通道而不是Copy。

```c#
	void DoToneMapping(int sourceId) {
		PostFXSettings.ToneMappingSettings.Mode mode = settings.ToneMapping.mode;
		Pass pass = Pass.ToneMappingNone + (int)mode;
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, pass);
	}
```

<font color="DarkOrchid ">ToneMappingSettings.Mode</font>枚举现在必须从零开始。

```c#
	public struct ToneMappingSettings {

		public enum Mode { None, ACES, Neutral, Reinhard }

		public Mode mode;
	}
```

#### 1.2、[Settings](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.2)

我们将复制URP和HDRP的<font color="red">*Color Adjustments* post-processing tool </font>的功能。第一步是在<font color="green">PostFXSettings</font>中为其添加一个配置结构。我添加了使用<font color="green">System</font>，因为我们还需要添加一堆Serializable属性。

```c#
using System;
using UnityEngine;

[CreateAssetMenu(menuName = "Rendering/Custom Post FX Settings")]
public class PostFXSettings : ScriptableObject {

	…

	[Serializable]
	public struct ColorAdjustmentsSettings {}

	[SerializeField]
	ColorAdjustmentsSettings colorAdjustments = default;

	public ColorAdjustmentsSettings ColorAdjustments => colorAdjustments;

	…
}
```

URP和HDRP的调色功能是相同的。我们将为调色添加相同的配置选项，顺序相同。首先是 "曝光后"，一个不受约束的浮点。之后是对比度，一个从-100到100的滑块。下一个选项是滤色器，这是一个没有阿尔法的HDR颜色。接下来是色相偏移，另一个滑块，但从-180°到+180°。最后一个选项是饱和度，同样是一个从-100到100的滑块。

```C#
public struct ColorAdjustmentsSettings {

		public float postExposure;

		[Range(-100f, 100f)]
		public float contrast;

		[ColorUsage(false, true)]
		public Color colorFilter;

		[Range(-180f, 180f)]
		public float hueShift;

		[Range(-100f, 100f)]
		public float saturation;
	}
```

默认值都是零，除了滤色镜应该是白色。这些设置不会改变图像。

```
	ColorAdjustmentsSettings colorAdjustments = new ColorAdjustmentsSettings {
		colorFilter = Color.white
	};
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/settings.png" alt="img" style="zoom:50%;" />

<center>Settings for color adjustments.</center>

我们同时在做<font color="DarkOrchid ">color grading and tone mapping</font>，所以重构后将<font color="green">PostFXStack.DoToneMapping</font>更名为<font color="red">DoColorGradingAndToneMapping</font>。

我们还将在这里经常访问<font color="red">PostFXSettings</font>的内部类型，所以让我们添加使用静态<font color="green">PostFXSettings</font>来保持代码的简短。

然后添加一个<font color="green">ConfigureColorAdjustments</font>方法，在这个方法中我们抓取颜色调整设置，并在<font color="RoyalBlue">DoColorGradingAndToneMapping</font>的开始调用它。

```c#
using UnityEngine;
using UnityEngine.Rendering;
using static PostFXSettings;

public partial class PostFXStack {
	
	…
	
	void ConfigureColorAdjustments () {
		ColorAdjustmentsSettings colorAdjustments = settings.ColorAdjustments;
	}

	void DoColorGradingAndToneMapping (int sourceId) {
		ConfigureColorAdjustments();

		ToneMappingSettings.Mode mode = settings.ToneMapping.mode;
		Pass pass = Pass.ToneMappingNone + (int)mode;
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, pass);
	}
	
	…
}
```

我们可以通过设置着色器矢量和颜色来进行<font color="red">color adjustments</font>就足够了。

颜色调整向量的组成部分是<font color="DarkOrchid ">exposure, contrast, hue shift, and saturation曝光、对比度、色调偏移和饱和度</font>。曝光是以档为单位的，这意味着我们必须把2提高到配置的曝光值的幂数。同时将对比度和饱和度转换为0-2范围，将色相偏移转换为-1-1。滤镜必须是在线性线性色彩空间中。

我不会展示添加附带的着色器属性标识符。

```c#
	ColorAdjustmentsSettings colorAdjustments = settings.ColorAdjustments;
		buffer.SetGlobalVector(colorAdjustmentsId, new Vector4(
			Mathf.Pow(2f, colorAdjustments.postExposure),
			colorAdjustments.contrast * 0.01f + 1f,
			colorAdjustments.hueShift * (1f / 360f),
			colorAdjustments.saturation * 0.01f + 1f
		));
		buffer.SetGlobalColor(colorFilterId, colorAdjustments.colorFilter.linear);
```

#### 1.3、[Post Exposure](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.3)

在着色器方面，添加矢量和颜色。我们将把每个调整放在自己的函数中，从曝光后开始。创建一个ColorGradePostExposure函数，将颜色与曝光值相乘。然后在ColorGrade中限制颜色后应用曝光。

```glsl
float4 _ColorAdjustments;
float4 _ColorFilter;

float3 ColorGradePostExposure (float3 color) {
	return color * _ColorAdjustments.x;
}

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	return color;
}
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/post-exposure-minus-2.png" alt="minus 2" style="zoom: 50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/post-exposure-plus-2.png" alt="plus 2" style="zoom:50%;" /></center>

<center>Post exposure −2 and 2.</center>

#### 1.4、[Contrast](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.4)

第二个调整是对比度。我们通过从颜色中减去统一的中间灰，然后通过对比度进行缩放，再加上中间灰来应用它。使用<font color="DarkOrchid ">ACEScc_MIDGRAY</font>作为灰色。

```glsl
float3 ColorGradingContrast (float3 color) {
	return (color - ACEScc_MIDGRAY) * _ColorAdjustments.y + ACEScc_MIDGRAY;
}

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradingContrast(color);
	return color;
}
```

为了获得最佳效果，这种覆盖是在<font color="green">Log C</font>而不是线性色彩空间中进行的。我们可以用色彩核心库文件中的<font color="green">LinearToLogC</font>函数从线性转换到Log C，然后用<font color="red">LogCToLinear</font>函数转换回来。

```glsl
float3 ColorGradingContrast (float3 color) {
	color = LinearToLogC(color);
	color = (color - ACEScc_MIDGRAY) * _ColorAdjustments.y + ACEScc_MIDGRAY;
	return LogCToLinear(color);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/logc.png" alt="minus 50" style="zoom:50%;" />

<center>Linear and Log C.</center>

当对比度增加时，这可能会导致负的颜色成分，这可能会扰乱后面的调整。因此，在<font color="green">ColorGrade</font>中调整对比度后要消除负值。

```glsl
color = ColorGradingContrast(color);
	color = max(color, 0.0);
```

<center><img src=".\..\..\\Typora-Note\assets\contrast-minus-50.png" alt="minus 50" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/contrast-plus-50.png" alt="plus 50" style="zoom:50%;" /></center>

<center>Contrast −50 and 50.</center>

#### 1.5、[Color Filter](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.5)

接下来是滤色器，简单地将其与颜色相乘。它对负值的效果很好，所以我们可以在消除它们之前应用它。

```glsl
float3 ColorGradeColorFilter (float3 color) {
	return color * _ColorFilter.rgb;
}

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradingContrast(color);
	color = ColorGradeColorFilter(color);
	color = max(color, 0.0);
	return color;
}
```

<img src=".\..\..\\Typora-Note\assets\color-filter.png" alt="img" style="zoom:50%;" />

<center>Light cyan color filter, eliminating most red light.</center>

#### 1.6、[Hue Shift](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.6)

URP和HDRP在滤色片之后进行色相偏移，我们将使用同样的调整顺序。颜色的色调是通过<font color="green">RgbToHsv</font>将颜色格式从RGB转换为HSV来调整的，再加上H的色调偏移，然后通过<font color="DarkOrchid ">HsvToRgb</font>转换回来。因为色调是在0-1色轮上定义的，如果它超出了范围，我们就得把它包起来。我们可以用<font color="red">RotateHue</font>来做这个，把调整后的色相、0和1作为参数传给它。这必须发生在负值被消除之后。

```
float3 ColorGradingHueShift (float3 color) {
	color = RgbToHsv(color);
	float hue = color.x + _ColorAdjustments.z;
	color.x = RotateHue(hue, 0.0, 1.0);
	return HsvToRgb(color);
}

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradingContrast(color);
	color = ColorGradeColorFilter(color);
	color = max(color, 0.0);
	color = ColorGradingHueShift(color);
	return color;
}
```

<img src=".\..\..\\Typora-Note\assets\hue-shift.png" alt="img" style="zoom:50%;" />

<center>180° hue shift.</center>

#### 1.7、[Saturation](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#1.7)

最后的调整是饱和度。首先借助<font color="red"> color's luminance 亮度函数</font>获得颜色的亮度。然后像对比度一样计算结果，只是用亮度而不是中间灰，也不是用对数C。这又会产生负值，所以要从<font color="DarkOrchid ">ColorGrade</font>的最终结果中删除这些负值。

```glsl
float3 ColorGradingSaturation (float3 color) {
	float luminance = Luminance(color);
	return (color - luminance) * _ColorAdjustments.w + luminance;
}

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradingContrast(color);
	color = ColorGradeColorFilter(color);
	color = max(color, 0.0);
	color = ColorGradingHueShift(color);
	color = ColorGradingSaturation(color);
	return max(color, 0.0);
}
```

<center><img src=".\..\..\\Typora-Note\assets\saturation-minus-100.png" alt="minus 100" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/saturation-plus-100.png" alt="plus 100" style="zoom:50%;" /></center>

<center>Saturation −100 and 100.</center>

### 2、More Controls

颜色调整工具并不是URP和HDRP提供的唯一调色选项。我们将增加对另外几个的支持，再次复制Unity的方法。

#### 2.1、White [Balance](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#2.1)

- [ ] 白平衡工具使得调整图像的感知温度成为可能。它有两个滑块，用于-100-100的范围。第一个是<font color="red">温度</font>，用于使图像更冷或更暖。第二个是<font color="DarkOrchid "> *Tint*色调</font>，用于调整温度偏移的颜色。在<font color="red">PostFXSettings</font>中为它添加一个设置结构，默认值为零。


```c#
	[Serializable]
	public struct WhiteBalanceSettings {

		[Range(-100f, 100f)]
		public float temperature, tint;
	}

	[SerializeField]
	WhiteBalanceSettings whiteBalance = default;

	public WhiteBalanceSettings WhiteBalance => whiteBalance;
```

<img src=".\..\..\\Typora-Note\assets\white-balance-settings.png" alt="img" style="zoom:50%;" />

<center>White balance settings.</center>

我们可以通过调用<font color="red">Core Library</font>中的<font color="green">ColorUtils.ColorBalanceToLMSCoeffs</font>来获得一个单一的矢量着色器属性，将<font color="RoyalBlue">温度和色调</font>传递给它。在<font color="red">PostFXStack</font>的专用配置方法中设置它，并在<font color="green">DoColorGradingAndToneMapping</font>的<font color="RoyalBlue">ConfigureColorAdjustments</font>之后调用。

```glsl
	void ConfigureWhiteBalance () {
		WhiteBalanceSettings whiteBalance = settings.WhiteBalance;
		buffer.SetGlobalVector(whiteBalanceId, ColorUtils.ColorBalanceToLMSCoeffs(
			whiteBalance.temperature, whiteBalance.tint
		));
	}
	
	void DoColorGradingAndToneMapping (int sourceId) {
		ConfigureColorAdjustments();
		ConfigureWhiteBalance();

		…
	}
```

在着色器方面，我们通过将颜色与LMS色彩空间的矢量相乘来应用白平衡。我们可以使用<font color="red">LinearToLMS</font>和<font color="RoyalBlue">LMSToLinear</font>函数转换为LMS并返回。在后期曝光之后和对比度之前应用它。

```c#
float4 _ColorAdjustments;
float4 _ColorFilter;
float4 _WhiteBalance;

float3 ColorGradePostExposure (float3 color) { … }

float3 ColorGradeWhiteBalance (float3 color) {
	color = LinearToLMS(color);
	color *= _WhiteBalance.rgb;
	return LMSToLinear(color);
}

…

float3 ColorGrade (float3 color) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradeWhiteBalance(color);
	color = ColorGradingContrast(color);
	…
}
```

冷的温度使图像变蓝，而暖的温度使图像变黄。通常使用小的调整，但我显示的是极端值，以使效果明显。

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/temperature-minus-100.png" alt="minus 100" style="zoom:50%;" /> <img src=".\..\..\\Typora-Note\assets\temperature-plus-100.png" alt="plus 100" style="zoom:50%;" /></center>

<center>Temperature −100 and 100.</center>

色调可以用来补偿不理想的色彩平衡，将图像推向绿色或洋红色。

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/tint-minus-100.png" alt="minus 100" style="zoom: 50%;" /> <img src=".\..\..\\Typora-Note\assets\tint-plus-100.png" alt="plus 100" style="zoom:50%;" /></center>

<center>Tint −100 and 100.</center>

#### 2.2、[Split Toning](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#2.2)

分割色调工具是用来分别对图像的阴影和高光部分进行着色。一个典型的例子是将阴影推向冷蓝色，将高光推向暖橙色。

为它创建一个设置结构，为阴影和高光创建两个不带alpha的LDR颜色。它们的默认值是灰色。还包括一个平衡-100-100的滑块，默认为0。

```c#
[Serializable]
	public struct SplitToningSettings {

		[ColorUsage(false)]
		public Color shadows, highlights;

		[Range(-100f, 100f)]
		public float balance;
	}

	[SerializeField]
	SplitToningSettings splitToning = new SplitToningSettings {
		shadows = Color.gray,
		highlights = Color.gray
	};

	public SplitToningSettings SplitToning => splitToning;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/split-toning-settings.png" alt="img" style="zoom:50%;" />

<center>Split toning settings.</center>

在<font color="green">PostFXStack</font>中向着色器发送两种颜色，保持它们在<font color="red">伽玛空间</font>中。平衡值可以存储在其中一个颜色的第四个分量中，缩放到-1-1范围。

```c#
	void ConfigureSplitToning () {
		SplitToningSettings splitToning = settings.SplitToning;
		Color splitColor = splitToning.shadows;
		splitColor.a = splitToning.balance * 0.01f;
		buffer.SetGlobalColor(splitToningShadowsId, splitColor);
		buffer.SetGlobalColor(splitToningHighlightsId, splitToning.highlights);
	}
	
	void DoColorGradingAndToneMapping (int sourceId) {
		ConfigureColorAdjustments();
		ConfigureWhiteBalance();
		ConfigureSplitToning();

		…
	}
```

在着色器方面，我们将在近似的<font color="green">伽马空间</font>中进行<font color="red">分色</font>，事先将颜色提高到2.2的逆值，之后再提高到2.2。这样做是为了配合Adobe产品的分割色调。

这个调整是在<font color="green">滤色器之后</font>进行的，在负值被消除之后。

```glsl
float4 _WhiteBalance;
float4 _SplitToningShadows, _SplitToningHighlights;

…

float3 ColorGradeSplitToning (float3 color) {
	color = PositivePow(color, 1.0 / 2.2);
	return PositivePow(color, 2.2);
}

…

float3 ColorGrade (float3 color) {
	…
	color = ColorGradeColorFilter(color);
	color = max(color, 0.0);
	color = ColorGradeSplitToning(color);
	…
}
```

我们通过在<font color="red">color and the shadows tint</font>之间进行柔光混合来应用色调，然后是高光色调。我们可以使用<font color="red">SoftLight</font>函数来做这个，两次。

```glsl
float3 ColorGradeSplitToning (float3 color) {
	color = PositivePow(color, 1.0 / 2.2);
	float3 shadows = _SplitToningShadows.rgb;
	float3 highlights = _SplitToningHighlights.rgb;
	color = SoftLight(color, shadows);
	color = SoftLight(color, highlights);
	return PositivePow(color, 2.2);
}
```

在混合之前，我们通过在<font color="red">中性0.5</font>和自己之间插值将色调限制在各自的区域。

对于高光部分，我们根据<font color="red">饱和的亮度</font>加上平衡，同样是饱和的。对于阴影，我们采用相反的方法。

```gls
float t = saturate(Luminance(saturate(color)) + _SplitToningShadows.w);
	float3 shadows = lerp(0.5, _SplitToningShadows.rgb, 1.0 - t);
	float3 highlights = lerp(0.5, _SplitToningHighlights.rgb, t);
	color = SoftLight(color, shadows);
	color = SoftLight(color, highlights);
```

<center><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/split-toning-blue-orange.png" alt="split toning" style="zoom:50%;" /><img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/color-adjustments/without-adjustments.png" alt="neutral" style="zoom:50%;" /></center>

<center>Split toning with blue and orange, and without adjustments for comparison.</center>

#### 2.3、[Channel Mixer](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#2.3)

我们将支持的另一个工具是通道混合器。它允许你结合输入的RGB值来创建一个新的RGB值。例如，你可以交换R和G，从G中减去B，或将G加到R中，将绿色推向黄色。

混合器本质上是一个3×3的转换矩阵，默认为<font color="red"> identity matrix </font>。我们可以使用三个Vector3值，用于红、绿、蓝的配置。Unity的控件为每种颜色显示一个单独的标签，每个输入通道有-100-100的滑块，但我们将简单地直接显示向量。行是输出的颜色，XYZ列是RGB输入。

```c#
	[Serializable]
	public struct ChannelMixerSettings {

		public Vector3 red, green, blue;
	}
	
	[SerializeField]
	ChannelMixerSettings channelMixer = new ChannelMixerSettings {
		red = Vector3.right,
		green = Vector3.up,
		blue = Vector3.forward
	};

	public ChannelMixerSettings ChannelMixer => channelMixer;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/channel-mixer-settings.png" alt="img" style="zoom:50%;" />

<center>Channel mixer set to identity matrix.</center>

将这三个向量发送到GPU。

```c#
	void ConfigureChannelMixer () {
		ChannelMixerSettings channelMixer = settings.ChannelMixer;
		buffer.SetGlobalVector(channelMixerRedId, channelMixer.red);
		buffer.SetGlobalVector(channelMixerGreenId, channelMixer.green);
		buffer.SetGlobalVector(channelMixerBlueId, channelMixer.blue);
	}

	void DoColorGradingAndToneMapping (int sourceId) {
		…
		ConfigureSplitToning();
		ConfigureChannelMixer();
		…
	}
```

并在着色器中执行<font color="red">矩阵乘法</font>。在<font color="green">分割调色</font>之后再做这个。之后再来消除负数，因为负数的权重可能会产生负的颜色通道。

```glsl
float4 _SplitToningShadows, _SplitToningHighlights;
float4 _ChannelMixerRed, _ChannelMixerGreen, _ChannelMixerBlue;

…

float3 ColorGradingChannelMixer (float3 color) {
	return mul(
		float3x3(_ChannelMixerRed.rgb, _ChannelMixerGreen.rgb, _ChannelMixerBlue.rgb),
		color
	);
}

float3 ColorGrade (float3 color) {
	…
	ColorGradeSplitToning(color);
	color = ColorGradingChannelMixer(color);
	color = max(color, 0.0);
	color = ColorGradingHueShift(color);
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/mixed-channels-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/mixed-channels-scene.png" alt="scene" style="zoom:50%;" />

<center>Green split between GB and blue split between RGB.</center>

#### 2.4、[Shadows Midtones Highlights](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#2.4)

我们将支持的最后一个工具是<font color="red">Shadows Midtones Highlights</font>。它的工作原理与分色法类似，只是它还允许调整中间调子，并将阴影和高光区域解耦，使其可配置。

Unity的控件显示了色轮和可视化的区域权重，但我们将使用三个HDR色域和四个滑块，用于阴影和高光过渡区的开始和结束。阴影强度从开始到结束会减少，而高光强度从开始到结束会增加。我们将使用0-2的范围，所以我们可以进入HDR一下。颜色默认为白色，我们将使用与Unity相同的区域默认值，即阴影为0-0.3，高光为0.55-1。

```c#
	[Serializable]
	public struct ShadowsMidtonesHighlightsSettings {

		[ColorUsage(false, true)]
		public Color shadows, midtones, highlights;

		[Range(0f, 2f)]
		public float shadowsStart, shadowsEnd, highlightsStart, highLightsEnd;
	}

	[SerializeField]
	ShadowsMidtonesHighlightsSettings
		shadowsMidtonesHighlights = new ShadowsMidtonesHighlightsSettings {
			shadows = Color.white,
			midtones = Color.white,
			highlights = Color.white,
			shadowsEnd = 0.3f,
			highlightsStart = 0.55f,
			highLightsEnd = 1f
		};

	public ShadowsMidtonesHighlightsSettings ShadowsMidtonesHighlights =>
		shadowsMidtonesHighlights;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/shadows-midtones-highlights-settings.png" alt="img" style="zoom:50%;" />

<center>Shadows midtones highlights settings.</center>

将三种颜色发送到GPU，转换为线性空间。区域范围可以打包在一个矢量中。

```c#
	void ConfigureShadowsMidtonesHighlights () {
		ShadowsMidtonesHighlightsSettings smh = settings.ShadowsMidtonesHighlights;
		buffer.SetGlobalColor(smhShadowsId, smh.shadows.linear);
		buffer.SetGlobalColor(smhMidtonesId, smh.midtones.linear);
		buffer.SetGlobalColor(smhHighlightsId, smh.highlights.linear);
		buffer.SetGlobalVector(smhRangeId, new Vector4(
			smh.shadowsStart, smh.shadowsEnd, smh.highlightsStart, smh.highLightsEnd
		));
	}

	void DoColorGradingAndToneMapping (int sourceId) {
		ConfigureColorAdjustments();
		ConfigureWhiteBalance();
		ConfigureSplitToning();
		ConfigureChannelMixer();
		ConfigureShadowsMidtonesHighlights();

		…
	}
```

在着色器中，我们将颜色分别与三种颜色相乘，每种颜色都按自己的<font color="DarkOrchid ">权重</font>进行缩放，然后将结果相加。这些权重是基于亮度的。阴影的权重从1开始，在其开始和结束之间减少到0，使用<font color="green">smoothstep</font>函数。高光部分的权重从0增加到1。而中间调的权重等于1减去其他两个权重。我们的想法是，阴影和高光区域不会重叠，或者只是一点点，所以中间调的权重永远不会变成负数。

然而，我们并没有在检查器中强制执行这一点，就像我们没有强制执行开始在结束之前一样。

```glsl
float4 _ChannelMixerRed, _ChannelMixerGreen, _ChannelMixerBlue;
float4 _SMHShadows, _SMHMidtones, _SMHHighlights, _SMHRange;

…

float3 ColorGradingShadowsMidtonesHighlights (float3 color) {
	float luminance = Luminance(color);
	float shadowsWeight = 1.0 - smoothstep(_SMHRange.x, _SMHRange.y, luminance);
	float highlightsWeight = smoothstep(_SMHRange.z, _SMHRange.w, luminance);
	float midtonesWeight = 1.0 - shadowsWeight - highlightsWeight;
	return
		color * _SMHShadows.rgb * shadowsWeight +
		color * _SMHMidtones.rgb * midtonesWeight +
		color * _SMHHighlights.rgb * highlightsWeight;
}

float3 ColorGrade (float3 color) {
	…
	color = ColorGradingChannelMixer(color);
	color = max(color, 0.0);
	color = ColorGradingShadowsMidtonesHighlights(color);
	…
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/shadows-midtones-highlights-adjusted.png" alt="img" style="zoom:50%;" />

<center>Blue shadows, pink midtones, and yellow highlights.</center>

Unity控件的色轮的工作原理是一样的，只是它们限制了输入的颜色，允许更精确的拖动。用HVS的取色器模式调整颜色，可以在一定程度上模仿这种功能，但没有约束。

#### 2.5、[ACES Color Spaces](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#2.5)

当使用ACES色调映射时，Unity在<font color="green">ACES色彩空间</font>而不是线性色彩空间中执行大部分调色，以产生更好的效果。我们也这样做吧。

后期曝光和白平衡总是在线性空间中应用。对比度是它的分歧之处。给<font color="green">ColorGradingContrast</font>添加一个布尔值<font color="RoyalBlue">useACES</font>参数。

如果使用ACES，首先从线性空间转换到ACES，然后转换到<font color="red">ACEScc</font>色彩空间，而不是对数C。在调整对比度后，通过<font color="RoyalBlue">ACEScc_to_ACES和ACES_to_ACEScg转换为ACEScg</font>，而不是回到线性空间。

ACEScg是ACES颜色空间的线性子集。

```glsl
float3 ColorGradingContrast (float3 color, bool useACES) {
	color = useACES ? ACES_to_ACEScc(unity_to_ACES(color)) : LinearToLogC(color);
	color = (color - ACEScc_MIDGRAY) * _ColorAdjustments.y + ACEScc_MIDGRAY;
	return useACES ? ACES_to_ACEScg(ACEScc_to_ACES(color)) : LogCToLinear(color);
}
```

从现在开始，我们在<font color="RoyalBlue">调颜色对比</font>步骤之后，要么在<font color="red">线性</font>空间，要么在<font color="red">ACEScg色</font>彩空间。

一切都还是一样的，只是亮度应该用<font color="DarkOrchid ">ACEScg</font>空间的<font color="green">AcesLuminance</font>来计算。引入一个<font color="green">Luminance</font>函数变量，根据是否使用ACES来调用正确的函数。

```glsl
float Luminance (float3 color, bool useACES) {
	return useACES ? AcesLuminance(color) : Luminance(color);
}
```

<font color="green">ColorGradeSplitToning</font>使用亮度，要给它一个<font color="red">useACES</font>参数并把它传给<font color="green">Luminance</font>。

```glsl
float3 ColorGradeSplitToning (float3 color, bool useACES) {
	color = PositivePow(color, 1.0 / 2.2);
	float t = saturate(Luminance(saturate(color), useACES) + _SplitToningShadows.w);
	…
}
```

对调色阴影中调高光做同样的处理。

```glsl
float3 ColorGradingShadowsMidtonesHighlights (float3 color, bool useACES) {
	float luminance = Luminance(color, useACES);
	…
}
```

而对于<font color="green">ColorGradingSaturation</font>。

```glsl
float3 ColorGradingSaturation (float3 color, bool useACES) {
	float luminance = Luminance(color, useACES);
	return luminance + _ColorAdjustments.w * (color - luminance);
}
```

然后把这个参数也添加到<font color="green">ColorGrade</font>中，这次默认设置为<font color="red">false</font>。把它传递给需要它的函数。最终的颜色应该在适当的时候通过<font color="red">ACEScg_to_ACES</font>转换为<font color="green">ACES</font>颜色空间。

```glsl
float3 ColorGrade (float3 color, bool useACES = false) {
	color = min(color, 60.0);
	color = ColorGradePostExposure(color);
	color = ColorGradeWhiteBalance(color);
	color = ColorGradingContrast(color, useACES);
	color = ColorGradeColorFilter(color);
	color = max(color, 0.0);
	ColorGradeSplitToning(color, useACES);
	color = ColorGradingChannelMixer(color);
	color = max(color, 0.0);
	color = ColorGradingShadowsMidtonesHighlights(color, useACES);
	color = ColorGradingHueShift(color);
	color = ColorGradingSaturation(color, useACES);
	return max(useACES ? ACEScg_to_ACES(color) : color, 0.0);
}
```

现在调整<font color="green">ToneMappingACESPassFragment</font>，使其表示使用ACES。由于<font color="red">ColorGrade</font>的结果将是ACES颜色空间，所以可以直接传递给<font color="RoyalBlue">ACESTonemap</font>。

```glsl
float4 ToneMappingACESPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ColorGrade(color.rgb, true);
	color.rgb = AcesTonemap(color.rgb);
	return color;
}
```

为了说明差异，这里是使用ACES色调映射与增加对比度和调整阴影、中间色调和高光的比较。

<center><img src=".\..\..\\Typora-Note\assets\aces-aces.png" alt="ACES" style="zoom:50%;" /> <img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/more-controls/aces-linear.png" alt="linear" style="zoom:50%;" /></center>

<center>ACES tone mapping with color grading in ACES and linear color spaces.</center>

### 3、[LUT](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#3)

对每个像素执行所有的调色步骤是一个很大的工作。我们可以制作一些变体，只应用改变某些东西的步骤，但这需要大量的关键词或通道。我们可以做的是将调色工作纳入一个<font color="red"> lookup table</font>--简称LUT--并对其进行采样以转换颜色。LUT是一个3D纹理，通常是32×32×32。

填充该纹理并在之后对其进行采样，比直接对整个图像进行调色要省事得多。URP和HDRP使用同样的方法。

#### 3.1、LUT Resolution

通常情况下，颜色LUT的分辨率为32就足够了，但让我们把它变成可配置的。这是一个质量设置，我们将添加到<font color="red">CustomRenderPipelineAsset</font>中，然后用于所有的调色。我们将使用一个枚举来提供16、32和64的选项，然后将其作为一个整数传递给管道构造函数。

```c#
	public enum ColorLUTResolution { _16 = 16, _32 = 32, _64 = 64 }

	[SerializeField]
	ColorLUTResolution colorLUTResolution = ColorLUTResolution._32;

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			allowHDR, useDynamicBatching, useGPUInstancing, useSRPBatcher,
			useLightsPerObject, shadows, postFXSettings, (int)colorLUTResolution
		);
	}
```

在<font color="green">CustomRenderPipeline</font>中保持对颜色LUT分辨率的跟踪，并将其传递给CameraRenderer.<font color="red">Render</font>方法。

```c#
	int colorLUTResolution;

	public CustomRenderPipeline (
		…
		PostFXSettings postFXSettings, int colorLUTResolution
	) {
		this.colorLUTResolution = colorLUTResolution;
		…
	}

	protected override void Render (ScriptableRenderContext context, Camera[] cameras) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera, allowHDR,
				useDynamicBatching, useGPUInstancing, useLightsPerObject,
				shadowSettings, postFXSettings, colorLUTResolution
			);
		}
	}
```

它将其传递给<font color="red">PostFXStack.Setup</font>。

```
public void Render (
		ScriptableRenderContext context, Camera camera, bool allowHDR,
		bool useDynamicBatching, bool useGPUInstancing, bool useLightsPerObject,
		ShadowSettings shadowSettings, PostFXSettings postFXSettings,
		int colorLUTResolution
	) {
		…
		postFXStack.Setup(context, camera, postFXSettings, useHDR, colorLUTResolution);
		…
	}
```

而<font color="red">PostFXStack</font>会对其进行跟踪。

```c#
int colorLUTResolution;

	…

	public void Setup (
		ScriptableRenderContext context, Camera camera, PostFXSettings settings,
		bool useHDR, int colorLUTResolution
	) {
		this.colorLUTResolution = colorLUTResolution;
		…
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/lut/color-lut-resolution.png" alt="img" style="zoom:50%;" />

<center>Color LUT resolution.</center>

#### 3.2、[Rendering to a 2D LUT Texture](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#3.2)

LUT是3D的，但普通的着色器无法渲染到3D纹理。所以我们将使用一个宽的2D纹理来代替模拟3D纹理，通过将2D切片放在一排。

因此，<font color="red">LUT纹理的高度等于配置的分辨率</font>，宽度等于分辨率的平方。使用默认的HDR格式，获得一个具有该尺寸的临时渲染纹理。在<font color="green">DoColorGradingAndToneMapping</font>中配置了调色之后再做这个。

```c#
	ConfigureShadowsMidtonesHighlights();

		int lutHeight = colorLUTResolution;
		int lutWidth = lutHeight * lutHeight;
		buffer.GetTemporaryRT(
			colorGradingLUTId, lutWidth, lutHeight, 0,
			FilterMode.Bilinear, RenderTextureFormat.DefaultHDR
		);
```

从现在开始，我们将把<font color="RoyalBlue">color grading and tone mapping调色和色调映射</font>都渲染到LUT中。

相应地重命名现有的<font color="red">tone-mapping passes - 色调映射通道</font>，因此<font color="green">ToneMappingNone</font>变成<font color="RoyalBlue">ColorGradingNone</font>，以此类推。然后使用适当的通道绘制到LUT，而不是摄像机目标。

然后将源图像复制到摄像机目标上，得到未经调整的图像作为最终结果，并释放LUT。

```c#
ToneMappingSettings.Mode mode = settings.ToneMapping.mode;
		Pass pass = Pass.ColorGradingNone + (int)mode;
		Draw(sourceId, colorGradingLUTId, pass);
		
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Copy);
		buffer.ReleaseTemporaryRT(colorGradingLUTId);
```

我们现在绕过了<font color="RoyalBlue">color grading and tone mapping 调色和色调映射</font>，但帧调试器显示，我们在最终拷贝前画了一个扁平化的图像版本。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/lut/flattened-image.png)

<center>Flattened image.</center>

#### 3.3、[LUT Color Matrix](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#3.3)

为了创建一个合适的LUT，我们需要用一个颜色转换矩阵来填充它。

我们通过调整<font color="green"> color grading pass functions  - 调色通道函数</font>来做到这一点，以使用从UV坐标导出的颜色，而不是对源纹理进行采样。添加一个<font color="green">GetColorGradedLUT</font>，它可以获得颜色并立即执行调色。

然后通过函数只需要在此基础上应用色调映射。

```glsl
float3 GetColorGradedLUT (float2 uv, bool useACES = false) {
	float3 color = float3(uv, 0.0);
	return ColorGrade(color, useACES);
}

float4 ColorGradingNonePassFragment (Varyings input) : SV_TARGET {
	float3 color = GetColorGradedLUT(input.screenUV);
	return float4(color, 1.0);
}

float4 ColorGradingACESPassFragment (Varyings input) : SV_TARGET {
	float3 color = GetColorGradedLUT(input.screenUV, true);
	color = AcesTonemap(color);
	return float4(color, 1.0);
}

float4 ColorGradingNeutralPassFragment (Varyings input) : SV_TARGET {
	float3 color = GetColorGradedLUT(input.screenUV);
	color = NeutralTonemap(color);
	return float4(color, 1.0);
}

float4 ColorGradingReinhardPassFragment (Varyings input) : SV_TARGET {
	float3 color = GetColorGradedLUT(input.screenUV);
	color /= color + 1.0;
	return float4(color, 1.0);
}
```

我们可以通过<font color="green">GetLutStripValue</font>函数找到LUT的输入颜色。它需要UV坐标和一个调色的LUT参数向量，我们需要将其发送给<font color="green">GPU。</font>

```glsl
float4 _ColorGradingLUTParameters;

float3 GetColorGradedLUT (float2 uv, bool useACES = false) {
	float3 color = GetLutStripValue(uv, _ColorGradingLUTParameters);
	return ColorGrade(color, useACES);
}
```

四个矢量参数值是LUT高度，0.5除以宽度，0.5除以高度，高度除以自身减去1。

```c#
		buffer.GetTemporaryRT(
			colorGradingLUTId, lutWidth, lutHeight, 0,
			FilterMode.Bilinear, RenderTextureFormat.DefaultHDR
		);
		buffer.SetGlobalVector(colorGradingLUTParametersId, new Vector4(
			lutHeight, 0.5f / lutWidth, 0.5f / lutHeight, lutHeight / (lutHeight - 1f)
		));
```

![none](.\..\..\\Typora-Note\assets\lut-none.png)

![ACES](.\..\..\\Typora-Note\assets\lut-aces.png)

![Reinhard](.\..\..\\Typora-Note\assets\lut-reinhard.png)

<center>LUTs without color grading, with no, ACES, and Reinhard tone mapping.</center>

#### 3.4、[Log C LUT](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#3.4)

我们得到的LUT矩阵是<font color="green">线性色彩空间</font>，只涵盖0-1范围。为了支持HDR，我们必须扩展这个范围。我们可以通过将输入的颜色解释为对数C空间来做到这一点。这就把范围扩大到了59以下。

<img src=".\..\..\\Typora-Note\assets\linear-logc.png" alt="img" style="zoom:50%;" />

<center>Stored linear and Log C intensities.</center>

```glsl
float3 GetColorGradedLUT (float2 uv, bool useACES = false) {
	float3 color = GetLutStripValue(uv, _ColorGradingLUTParameters);
	return ColorGrade(LogCToLinear(color), useACES);
}
```

![none](.\..\..\\Typora-Note\assets\lut-logc-none.png)

![ACES](.\..\..\\Typora-Note\assets\lut-logc-aces.png)

![Reinhard](.\..\..\\Typora-Note\assets\lut-logc-reinhard.png)

<center>LogC colors with no, ACES, and Reinhard tone mapping.</center>

与线性空间相比，Log C对最暗的数值增加了一点分辨率。它在大约0.5时超过了线性值。在那之后，强度迅速上升，所以矩阵的分辨率下降了很多。这对于覆盖HDR值是需要的，但如果我们不需要这些，最好坚持使用线性空间，否则几乎一半的分辨率都被浪费了。在着色器中添加一个布尔值来控制这个。

```c#
bool _ColorGradingLUTInLogC;

float3 GetColorGradedLUT (float2 uv, bool useACES = false) {
	float3 color = GetLutStripValue(uv, _ColorGradingLUTParameters);
	return ColorGrade(_ColorGradingLUTInLogC ? LogCToLinear(color) : color, useACES);
}
```

只有在使用<font color="green">HDR并应用色调映射</font>的情况下才启用对数C模式。

```c#
	ToneMappingSettings.Mode mode = settings.ToneMapping.mode;
		Pass pass = Pass.ColorGradingNone + (int)mode;
		buffer.SetGlobalFloat(
			colorGradingLUTInLogId, useHDR && pass != Pass.ColorGradingNone ? 1f : 0f
		);
		Draw(sourceId, colorGradingLUTId, pass);
```

因为我们不再依赖渲染后的图像，我们不再需要将范围限制在60。它已经被LUT的范围所限制。

```glsl
float3 ColorGrade (float3 color, bool useACES = false) {
	//color = min(color, 60.0);
	…
}
```

#### 3.5、[Final Pass](https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/#3.5)

为了应用LUT，我们引入了一个新的最终通道。它所需要做的就是获得源颜色并对其应用调色LUT。在一个单独的<font color="DarkOrchid ">ApplyColorGradingLUT</font>函数中进行。

```glsl
float3 ApplyColorGradingLUT (float3 color) {
	return color;
}

float4 FinalPassFragment (Varyings input) : SV_TARGET {
	float4 color = GetSource(input.screenUV);
	color.rgb = ApplyColorGradingLUT(color.rgb);
	return color;
}
```

我们可以通过<font color="red">ApplyLut2D</font>函数来应用LUT，它负责将<font color="RoyalBlue">2D LUT条带解释为3D纹理</font>。它需要LUT纹理和采样器状态作为参数，然后是饱和的输入颜色--在线性或对数C空间中都可以，最后是一个参数向量，尽管这次只有三个分量。

```glsl
TEXTURE2D(_ColorGradingLUT);

float3 ApplyColorGradingLUT (float3 color) {
	return ApplyLut2D(
		TEXTURE2D_ARGS(_ColorGradingLUT, sampler_linear_clamp),
		saturate(_ColorGradingLUTInLogC ? LinearToLogC(color) : color),
		_ColorGradingLUTParameters.xyz
	);
}
```

在这种情况下，参数值是1除以LUT宽度，1除以高度，以及高度减去1。在最后绘制之前设置这些，现在使用最后的通道。

```c#
buffer.SetGlobalVector(colorGradingLUTParametersId,
			new Vector4(1f / lutWidth, 1f / lutHeight, lutHeight - 1f)
		);
		Draw(sourceId, BuiltinRenderTextureType.CameraTarget, Pass.Final);
		buffer.ReleaseTemporaryRT(colorGradingLUTId);
```

#### 3.6、LUT Banding

虽然我们现在使用LUT进行<font color="RoyalBlue">color grading and tone mapping - 调色和色调映射</font>，但结果应该和以前一样。然而，由于LUT的分辨率是有限的，而且我们用双线性插值对其进行采样，它将原本平滑的色彩过渡转换为线性带。

对于分辨率为32的LUT来说，这通常是不明显的，但在具有极端HDR色彩梯度的区域，带状现象会变得明显。一个例子是前面教程的色调映射场景中强度为200的聚光灯的衰减，它照亮了一个均匀的白色表面。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/lut/banding-16.png" alt="16" style="zoom:50%;" />

<img src=".\..\..\\Typora-Note\assets\banding-32.png" alt="32" style="zoom:50%;" />

<center>Color banding, LUT resolution 16 and 32.</center>

通过暂时切换到<font color="RoyalBlue">sampler_point_clamp</font>采样器状态，可以使带状现象非常明显。这将关闭LUT的2D片内的插值。由于<font color="RoyalBlue">ApplyLut2D</font>通过对两个片断进行采样并在它们之间进行混合来模拟3D纹理，所以相邻片断之间仍然存在插值。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/color-grading/lut/banding-16.png" alt="16" style="zoom:50%;" />

<img src=".\..\..\\Typora-Note\assets\banding-clamp-32.png" alt="32" style="zoom:50%;" />

<center>Point sampling, LUT resolution 16 and 32.</center>

如果带状现象太明显，你可以把分辨率提高到64，但颜色的一点点变化通常足以掩盖它。如果你在非常微妙的色彩转换中去寻找带状伪影，你更有可能发现由于8位帧缓冲器的限制而产生的带状，这不是由LUT引起的，可以通过抖动来缓解，但这是另一个话题。