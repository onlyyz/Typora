# [02_Draw Calls](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/)

白一个HLSL<font color="green">  Shader</font>。
支持<font color="green">  SRP batcher</font>，<font color="green"> GPU instancing </font>，以及动态批处理。
为每个对象配置材质属性，并随机绘制许多。
创建透明和镂空的材质。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/tutorial-image.jpg)

<center>Many spheres, but only a few draw calls.</center>

### 1、Shaders 

为了绘制东西，CPU必须告诉GPU绘制什么和如何绘制。所画的东西通常是一个网格。如何绘制是由<font color="green">  Shader</font>定义的，它是一组给GPU的指令。除了网格，<font color="green">  Shader</font>还需要额外的信息来完成其工作，包括物体的<font color="red">transformation matrices and material properties</font>变换矩阵和材料属性。

Unity的<font color="RoyalBlue">LW/Universal和HD RP</font>允许你用<font color="DarkOrchid ">Shader Graph</font>包来设计<font color="green">  Shader</font>，它可以为你生成<font color="green">  Shader</font>代码。但是我们的定制RP不支持这个，所以我们必须自己编写<font color="green">  Shader</font>代码。这使我们能够完全控制和理解<font color="green">  Shader</font>的作用。

#### [1.1、Unlit Shader](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.1)

我们的第一个<font color="green">  Shader</font>将简单地绘制一个纯色的网格，没有任何照明。一个<font color="green">  Shader</font>资产可以通过<font color="green"> *Assets / Create / Shader* menu</font>菜单中的一个选项创建。<font color="green">Unlit Shader</font>是最合适的，但我们要从新开始，从创建的shader文件中删除所有的默认代码。将资产命名为<font color="green">Unlit</font>，并放在<font color="DarkOrchid ">*Custom RP*.</font>下的一个新的<font color="green">Shaders</font>文件夹中。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/shader-asset.png" alt="img" style="zoom:50%;" />

<center>Unlit shader asset.</center>

<font color="green">  Shader</font>代码在大多数情况下看起来像C#代码，但它由不同的方法组成，包括一些在过去有意义但现在没有意义的古老的旧位。

<font color="green">  Shader</font>的定义就像一个类，但是只有<font color="red">Shader keyword </font>，后面跟着一个字符串，用来在材质的<font color="green">Shader</font>下拉菜单中为它创建一个条目。

让我们使用<font color="green">*Custom RP/Unlit*</font>。接下来是一个代码块，它包含了更多前面有关键词的块。有一个Properties（属性）块来定义材质属性，接着是<font color="red">SubShader</font>（子<font color="green">  Shader</font>）块，里面需要有一个<font color="green">Pass</font>（通行证）块，它定义了渲染某种东西的一种方式。用其他的空块创建这个结构。

```glsl
Shader "Custom RP/Unlit" {
	
	Properties {}
	
	SubShader {
		
		Pass {}
	}
}
```

这定义了一个最小的Shader，它可以编译并允许我们创建一个使用它的材料。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-material.png" alt="img" style="zoom:50%;" />

<center>Custom unlit material.</center>

默认的<font color="green">  Shader</font>实现将网格渲染成纯白色。材质显示了渲染队列的<font color="red"> render queue</font>，它自动从<font color="green">  Shader</font>中获取该属性，并设置为<font color="green">2000</font>，这是不透明几何体的默认值。它也有一个切换按钮来启用<font color="green"> double-sided global illumination</font>，但这对我们来说并不重要。

#### [1.2、Shaders HLSL程序](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.2)

我们用来编写<font color="green">  Shader</font>代码的语言是高级着色语言，简称<font color="red">HLSL</font>。我们必须把它放在<font color="green">Pass</font>块中，在<font color="red">HLSLPROGRAM和ENDHLSL</font>关键字之间。我们必须这样做，因为在Pass块中也有可能放入其他非HLSL代码。

```glsl
	Pass {
			HLSLPROGRAM
			ENDHLSL
		}
```

>   那么CG程序呢？
>   Unity也支持编写CG而不是HLSL程序，但我们将完全使用HLSL，就像Unity的现代RP一样。

要绘制网格，GPU 必须光栅化所有三角形，将其转换为像素数据。它通过将<font color="green"> vertex coordinates</font>从 3D 空间转换到 2D 可视化空间，然后填充由生成的三角形覆盖的所有像素来实现。

这两个步骤由单独的<font color="green"> shader programs</font>控制，我们都必须定义它们。

第一个称为<font color="red"> vertex kernel/program/shader</font>

第二个称为<font color="red">fragment kernel/program/shader</font>。

一个片段对应于一个显示<font color="RoyalBlue">pixel or texture texel</font>，尽管它可能不代表最终结果，因为当稍后在其上绘制某些东西时它可能会被覆盖。

我们必须用一个名称来标识这两个程序，这是通过<font color="green"># pragma </font>指令完成的。

这些是以相关名称开头的单行语句，<font color="green">#pragma</font>后面跟有<font color="red">vertex或fragment</font>加上相关名称。我们将使用<font color="green">UnlitPassVertex和UnlitPassFragment</font>。

```glsl
				HLSLPROGRAM
                #pragma vertex UnlitPassVertex
                #pragma fragment UnlitPassFragment
                ENDHLSL
```

<font color="red">shader compiler</font>现在会抱怨说它找不到已声明的<font color="green">  Shader</font>内核(<font color="green">shader kernels</font>)。我们必须用同样的名字编写HLSL函数来定义其实现。我们可以直接在<font color="green">pragma</font>指令下面这样做，但我们将把所有的HLSL代码放在一个单独的文件中。

具体来说，我们将在同一个资产文件夹中使用一个<font color="green">UnlitPass.hlsl</font>文件。我们可以通过添加一个<font color="green">#include</font>指令和该文件的相对路径来指示<font color="green">  Shader</font>插入该文件的内容。

```glsl
	HLSLPROGRAM
			#pragma vertex UnlitPassVertex
			#pragma fragment UnlitPassFragment
			#include "UnlitPass.hlsl"
			ENDHLSL
```

Unity没有一个方便的菜单选项来创建<font color="green">HLSL</font>文件，所以你必须做的是复制<font color="green">  Shader</font>文件，将其重命名为<font color="green">UnlitPass</font>，在外部将其文件扩展名改为hlsl并清除其内容。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-pass-asset.png" alt="img" style="zoom:67%;" />

<center>*UnlitPass* HLSL asset file.</center>

#### [1.3、Include Guard](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.3)

HLSL文件用来分组代码，就像C#类一样，尽管HLSL没有类的概念。除了代码块的局部作用域<font color="red" >（local scopes ）</font>，只有一个全局作用域<font color="red" >（global scope）</font>。

所以所有的东西都可以在任何地方访问。包括一个文件也和使用一个命名空间不一样。

它在include指令的位置插入了文件的全部内容，<font color="red" >如果你多次包含同一个文件，你会得到重复的代码，这很可能会导致编译器错误。</font>为了防止这种情况，我们将在<font color="green">UnlitPass.hlsl</font>中添加一个<font color="RoyalBlue">include guard</font>。

可以使用<font color="green">#define</font>指令来定义任何标识符，这通常是以大写字母进行的。我们将用它来定义文件顶部的<font color="OrangeRed" ><b>CUSTOM_UNLIT_PASS_INCLUDED</b></font>。

```glsl
#define CUSTOM_UNLIT_PASS_INCLUDED
```

这是一个简单宏的例子，它只是定义了一个标识符<font color="green" >( identifier)</font>。如果它存在，那么就意味着我们的文件已经被包含了。

所以我们不想再包含它的内容。换句话说，<font color="red">我们只想在它还没有被定义时插入代码。我们可以用<font color="green" >#ifndef</font>指令来检查。</font>在定义宏之前要这样做。

```glsl
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED
```

如果宏已经被定义，<font color="DarkOrchid "><font color="green">#ifndef</font>后面的所有代码将被跳过，因此不会被编译。我们必须在文件末尾添加一个<font color="green">#endif</font>指令来终止其范围</font>。

```glsl
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED
#endif
```

现在我们可以确定文件的所有相关代码永远不会被多次插入，即使我们最终不止一次地包含它。

#### [1.4、Shader Functions](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.4)

我们在<font color="green"> include guard </font>的范围内定义我们的<font color="green">  Shader</font>函数。它们就像没有任何访问修饰符的 C# 方法一样编写。从什么都不做的简单 void 函数开始。

```glsl
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

void UnlitPassVertex () {}

void UnlitPassFragment () {}

#endif
```

这足以让我们的<font color="green">  Shader</font>编译。结果可能是默认的青色<font color="green">  Shader</font>，如果有任何显示的话。

![img](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/cyan-sphere.png)

<center>青色球体。</center>

为了产生有效的输出，我们必须使我们的片段函数返回一个颜色。颜色是用一个包含其红、绿、蓝和阿尔法分量的四分量float4向量定义的。

我们可以通过float4(0.0, 0.0, 0.0, 0.0)来定义纯黑色，但我们也可以写一个单一的0，因为单一的值会自动扩展为一个完整的向量。alpha值并不重要，因为我们要创建一个不透明的<font color="green">  Shader</font>，所以0就可以了。

```c#
float4 UnlitPassFragment () {
	return 0.0;
}
```

##### 1.4.2、SV_TARGET and SV_POSITION

此时<font color="green">  Shader</font>编译器将失败，shader编译器会失败，因为我们的函数缺少语义。

我们必须说明我们返回的值是什么意思，因为我们有可能产生很多具有不同含义的数据。在这种情况下，我们为渲染目标提供<font color="red">默认的系统值</font>，通过在<font color="green">UnlitPassFragment</font>的参数列表后面写上冒号和<font color="red">SV_TARGET</font>来表示。

>   [SV_TARGET](https://docs.unity3d.com/cn/2021.1/Manual/SL-ShaderSemantics.html)   [论坛](https://forum.unity.com/threads/very-simple-question-about-fragment-function.482942/)
>
>   [SV_TARGET and SV_POSITION](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics)

```glsl
float4 UnlitPassFragment () : SV_TARGET {
	return 0.0;
}
```

<font color="green">UnlitPassVertex</font>负责转换顶点位置，所以应该返回一个位置。这也是一个float4的向量，因为它必须被定义为一个齐次裁剪空间位置<font color="green">（homogeneous clip space position）</font>，但我们稍后会讨论这个问题。

我们再次从零向量开始，在这种情况下，我们必须指出它的含义是<font color="red">SV_POSITION</font>。

>   SV_POSITION
>
>   简单地说POSITION语意用于顶点[<font color="green">  Shader</font>](https://so.csdn.net/so/search?q=<font color="green">  Shader</font>&spm=1001.2101.3001.7020)，用来指定这些个位置坐标值，是在变换前的顶点的object space坐标。SV_POSITION语意则用于像素<font color="green">  Shader</font>，用来标识经过顶点<font color="green">  Shader</font>变换之后的顶点坐标。
>
>   SV是Systems Value的简写。在SV_Position的情况下，如果它是绑定在一个从VS输出的数据结构上的话，意味着这个输出的数据结构包含了最终的转换过的，将用于光栅器的顶点坐标。或者，如果你将这个标志绑定到一个输入给PS的数据数据的话，它会包含一个基于屏幕空间的像素的坐标。所有的这些SV语义都在DirectX SDK文档中有描述。
>
>
>   VS将会输出顶点的坐标到裁剪空间中，并且这些坐标是使用齐次坐标系的。一般滴，这些基于裁剪空间的齐次坐标，是通过对输入的object space坐标位置乘以一个WVP矩阵而得到的。然后在光栅器中，这些齐次坐标将会进行透视除法，即对这些坐标的每个分量除以w，做完透视除法后，这些坐标值将会限定在一个左下角是-1，右上角是1的视口范围中。然后做完“把Y坐标翻转，把取值范围从[-1,1]限制到[0,1]，接着xy方向分别乘以视口的宽高”的视口变换之后，你便获得了“以SV_Position为标识的，传递给PS的，取值范围是[0,视口高宽]，视口左上角为坐标原点的”像素位置坐标。

```glsl
float4 UnlitPassVertex () : SV_POSITION {
	return 0.0;
}
```

#### [1.5、Space Transformation](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.5)

当所有的顶点都被设置为零时，网格会坍缩成一个点，没有任何东西被渲染出来。

<font color="red">顶点函数<font color="green">（vertex function）</font>的主要工作是将原始顶点位置转换到正确的空间。当被调用时，如果我们要求的话，该函数会提供可用的顶点数据。</font>

我们通过向<font color="green">UnlitPassVertex</font>添加参数来做到这一点。

我们需要顶点位置，它是在<font color="DarkOrchid ">object space</font>中定义的，所以我们将它命名为<font color="green">positionOS</font>，使用与<font color="red">Unity的new RPs</font>相同的约定。

这个位置的类型是<font color="green">float3</font>，因为它是一个三维点。让我们首先返回它，通过<font color="green">float4(positionOS, 1.0)****</font>添加1作为第四个需要的分量。

```glsl
float4 UnlitPassVertex (float3 positionOS) : SV_POSITION {
	return float4(positionOS, 1.0);
}
```

我们还必须向输入添加语义，因为<font color="green">vertex data</font>以包含的不仅仅是一个位置。在这种情况下，我们需要<font color="red">POSITION</font>在参数名称后直接添加颜色。

```glsl
float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
	return float4(positionOS, 1.0);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/object-space-position.png" alt="img" style="zoom:50%;" />

​																			使用object space位置。

网格又显示出来了，但不正确，因为我们输出的位置是在错误的空间。

空间转换需要矩阵，当有东西被画出来时，这些矩阵会被发送到GPU。

我们必须将这些矩阵添加到我们的<font color="green">  Shader</font>中，但由于它们总是相同的，我们将把Unity提供的标准输入放在一个单独的<font color="green">HLSL</font>文件中，这既是为了保持代码的结构化，也是为了能够将代码包含在其他<font color="green">  Shader</font>中。

添加一个<font color="green">UnityInput.hlsl</font>文件，把它放在<font color="green">ShaderLibrary</font>文件夹中，直接放在<font color="red">Custom RP</font>下面，以反映Unity的RP的文件夹结构。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/shader-library.png" alt="img" style="zoom:50%;" />

​													ShaderLibrary*文件夹与*UnityInput*文件。

以<font color="green"> include guard </font>开始文件，<font color="red">CUSTOM_UNITY_INPUT_INCLUDED</font>然后定义一个在全局范围内<font color="green">float4x4</font>命名的矩阵。

<font color="red">unity_ObjectToWorld</font>在 C# 类中，这将定义一个字段，但在这里它被称为统一值。它在每次绘制时由 GPU 设置一次，在绘制期间对顶点和片段函数的所有调用保持不变（统一）。

```glsl
#ifndef CUSTOM_UNITY_INPUT_INCLUDED
#define CUSTOM_UNITY_INPUT_INCLUDED

float4x4 unity_ObjectToWorld;

#endif
```

我们可以使用矩阵从<font color="green">object space</font>转换到<font color="green">world space</font>。

因为这是常见的功能，所以让我们为它创建一个函数并将其放入另一个文件中，这次<font color="green">Common.hlsl</font>在相同的<font color="green">ShaderLibrary</font>文件夹。我们包括<font color="green">UnityInput</font>

在那里然后声明一个TransformObjectToWorld函数 float3作为输入和输出。

```glsl
#ifndef CUSTOM_COMMON_INCLUDED
#define CUSTOM_COMMON_INCLUDED

#include "UnityInput.hlsl"

float3 TransformObjectToWorld (float3 positionOS) {
	return 0.0;
}
	
#endif
```

空间转换是通过调用mul函数与一个矩阵和一个矢量来完成的。

在这种情况下，我们确实需要一个4D向量，但由于它的第四个分量总是1，我们可以通过使用<font color="green">float4(positionOS, 1.0)</font>来自己添加它。结果还是一个4D向量，其第四个分量总是1。我们可以通过访问该向量的xyz属性来提取前三个分量，这就是所谓的<font color="red">swizzle</font>操作。

```glsl
float3 TransformObjectToWorld (float3 positionOS) {
	return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
}
```

我们现在可以在<font color="green">UnlitPassVertex</font>. 首先包括<font color="green">Common.hlsl</font>函数的正上方。由于它存在于不同的文件夹中，我们可以通过相对路径访问它<font color="green">../ShaderLibrary/Common.hlsl</font>. 

然后用于<font color="green">TransformObjectToWorld</font>计算<font color="green">positionWS</font>变量并返回它而不是<font color="red">object space</font>位置。

```glsl
#include "../ShaderLibrary/Common.hlsl"

float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
	float3 positionWS = TransformObjectToWorld(positionOS.xyz);
	return float4(positionWS, 1.0);
}
```

结果仍然是错误的

因为我们需要在<font color="red">homogeneous clip space</font>中的位置。这个空间定义了一个立方体，包含相机视野中的所有东西，在透视相机的情况下扭曲成梯形。

从世界空间到这个空间的转换可以通过乘以<font color="green">view-projection matrix</font>来完成，该矩阵考虑了相机的位置、方向、投影、视野和远近裁剪平面。

它提供了<font color="RoyalBlue">unity_ObjectToWorld</font>矩阵，所以将它添加到<font color="green">UnityInput.hlsl.</font>

```glsl
float4x4 unity_ObjectToWorld;

float4x4 unity_MatrixVP;
```

在<font color="green">Common.hlsl</font>中添加一个<font color="RoyalBlue">TransformWorldToHClip</font>，其工作原理与<font color="RoyalBlue">TransformObjectToWorld</font>相同，只是它的输入是在世界空间，使用其他矩阵，并产生一个<font color="green">float4</font>。

```c#
float3 TransformObjectToWorld (float3 positionOS) {
	return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
}

float4 TransformWorldToHClip (float3 positionWS) {
	return mul(unity_MatrixVP, float4(positionWS, 1.0));
}
```

让<font color="green">UnlitPassVertex</font>使用该函数来返回正确空间中的位置。

```c#
float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
	float3 positionWS = TransformObjectToWorld(positionOS.xyz);
	return TransformWorldToHClip(positionWS);
}
```



![img](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/black-sphere.png)

<center>Correct black sphere.</center>

#### [1.6、Core Library](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.6)

我们刚刚定义的两个函数非常常见，它们也包含在<font color="blue">Core RP Pipeline</font>包裹。核心库定义了许多更有用和必要的东西，所以让我们安装那个包，删除我们自己的定义并包含相关文件，在这种情况下

<font color="red">*Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl*.</font>

```C#
//float3 TransformObjectToWorld (float3 positionOS) {
//	return mul(unity_ObjectToWorld, float4(positionOS, 1.0)).xyz;
//}

//float4 TransformWorldToHClip (float3 positionWS) {
//	return mul(unity_MatrixVP, float4(positionWS, 1.0));
//}

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```

编译失败，因为代码在<font color="green">SpaceTransforms.hlsl</font>不假设<font color="red">unity_ObjectToWorld</font>的存在。相反，它期望相关矩阵是<font color="green">UNITY_MATRIX_M</font>由宏定义的

所以让我们在包含文件之前通过<font color="green">#define</font> <font color="blue">UNITY_MATRIX_M unity_ObjectToWorld</font>在单独的行上写入来做到这一点。之后所有出现的<font color="green">UNITY_MATRIX_M</font>都会被替换<font color="green">unity_ObjectToWorld</font>。我们稍后会发现这是有原因的。

```c#
#define UNITY_MATRIX_M unity_ObjectToWorld

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```

对于逆矩阵 也是如此<font color="green">unity_WorldToObject</font>，它应该通过 定义<font color="red">UNITY_MATRIX_I_M</font>，

<font color="green">unity_MatrixV</font>矩阵通过<font color="DarkOrchid ">UNITY_MATRIX_V</font>和<font color="DarkOrchid ">unity_MatrixVP</font>通过<font color="green">UNITY_MATRIX_VP</font>。

最后，还有通过 定义的投影矩阵<font color="green">UNITY_MATRIX_P</font>可用<font color="green">glstate_matrix_projection</font>. 我们不需要这些额外的矩阵，但如果我们不包含它们，代码将无法编译。

```glsl
#define UNITY_MATRIX_M unity_ObjectToWorld
#define UNITY_MATRIX_I_M unity_WorldToObject
#define UNITY_MATRIX_V unity_MatrixV
#define UNITY_MATRIX_VP unity_MatrixVP
#define UNITY_MATRIX_P glstate_matrix_projection
```

将额外的矩阵添加到*<font color="green">UnityInput</font>*以及。

```glsl
float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;

float4x4 unity_MatrixVP;
float4x4 unity_MatrixV;
float4x4 glstate_matrix_projection;
```

最后缺少的是一个矩阵以外的东西。它是<font color="green">unity_WorldTransformParams</font>，它包含了一些我们在这里不需要的变换信息。它是一个定义为<font color="red">real4</font>的向量，它本身不是一个有效的类型，而是对<font color="green">float4或half4</font>的别名，取决于目标平台。

```glsl
float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;
real4 unity_WorldTransformParams;
```

该别名和许多其他基本宏是根据图形 API 定义的，我们可以通过包含*<font color="green">Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl</font>*. 

在我们的<font color="green">Common.hlsl</font>包括之前的文件<font color="green">UnityInput.hlsl.</font> 如果您对它们的内容感到好奇，可以检查导入包中的这些文件。

```c#
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "UnityInput.hlsl"
```

#### [1.7、Color](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#1.7)

为了能够配置每种材质的颜色，我们必须将其定义为统一值。在 <font color="green">include </font>指令下方，<font color="green">UnlitPassVertex</font>函数之前执行此操作。我们需要一个<font color="green">float4</font>，我们将命名它<font color="red">_BaseColor</font>。前导下划线是表示它代表材料属性的标准方式。返回此值而不是 中的硬编码颜色<font color="green">UnlitPassFragment</font>。

```glsl
#include "../ShaderLibrary/Common.hlsl"

float4 _BaseColor;

float4 UnlitPassVertex (float3 positionOS : POSITION) : SV_POSITION {
	float3 positionWS = TransformObjectToWorld(positionOS);
	return TransformWorldToHClip(positionWS);
}

float4 UnlitPassFragment () : SV_TARGET {
	return _BaseColor;
}
```

我们回到黑色，因为默认值为零。要将其链接到我们必须添加<font color="green">_BaseColor</font>到<font color="green">Properties</font>块中的材料<font color="green">*Unlit* shader file.</font>。

```glsl
	Properties {
		_BaseColor
	}
```

属性名后面必须跟一个在检查器中使用的字符串和一个<font color="green">Color</font>类型标识符，就像为方法提供参数一样。

```glsl
		_BaseColor("Color", Color)
```

最后，我们必须提供一个默认值，在本例中是通过为其分配一个包含四个数字的列表。让我们使用白色。

```glsl
		_BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/shaders/unlit-material-color.png" alt="img" style="zoom:50%;" />

<center>未点亮的红色材料。</center>

现在可以使用我们的<font color="green">  Shader</font>创建多种材质，每种材质都有不同的颜色。

### [2、Batching](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2)

每个绘图调用都需要 CPU 和 GPU 之间的通信。如果必须将大量数据发送到 GPU，那么等待可能会浪费时间。当 CPU 忙于发送数据时，它不能做其他事情。这两个问题都会降低帧速率。

目前我们的方法很简单：每个对象都有自己的绘制调用。这是最糟糕的方法，尽管我们最终发送的数据很少，所以现在没问题。

例如，我制作了一个包含 76 个球体的场景，每个球体使用四种材质中的一种：红色、绿色、黄色和蓝色。它需要 <font color="red">draw calls </font>用来渲染，76 次用于球体，1 次用于天空盒，1 次用于清除渲染目标。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/76-spheres.png" alt="img" style="zoom: 80%;" />

<center>*76 spheres, 78 draw calls.*</center>

如果你打开*Stats*面板的*Game*窗口然后您可以看到渲染框架所需的概述。这里有趣的事实是，它显示了  <font color="green">77 batches</font>（忽略清除部分），其中零是通过批处理保存的。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/statistics.png" alt="img" style="zoom:67%;" />

<center>游戏窗口统计</center>

## [2.1、SRP Batcher](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.1)

<font color="green">Batching</font>是组合绘制调用的过程，减少CPU和GPU之间的通信时间。最简单的方法是启用<font color="red">SRP batcher</font>。然而，这只适用于兼容的<font color="green">  Shader</font>，而我们的<font color="green">Unlit</font><font color="green">  Shader</font>不是这样的。

你可以通过在<font color="green">inspector</font>中选择它来验证这一点。有一个<font color="green">*SRP Batcher* </font>程序的行表示不兼容，在它下面给出了一个原因

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/not-compatible.png" alt="img" style="zoom:67%;" />

​																不兼容。

与其说<font color="green">SRP batches</font>减少了<font color="red">draw calls</font>的数量，不如说是使其更加精简。

<font color="red">它把材料属性缓存在GPU上，所以它们不必在每次绘制调用时都被发送</font>这既减少了必须传达的数据量，也减少了CPU在每次绘制调用时必须做的工作。

但是，这只有在<font color="green">shader </font>遵守统一数据的严格结构时才有效。

<font color="green">所有的材质属性都必须被定义在一个具体的内存缓冲区内，而不是在全局层面</font>。

<font color="red">通过将_BaseColor声明包裹在一个带有<font color="blue">UnityPerMaterial名称的cbuffer块</font>中来实现的</font>

_这就像一个结构声明，但必须用分号来结束。它通过将_BaseColor放在一个特定的<font color="*MediumBlue">常量内存缓冲区<ins>( constant memory buffer)</ins></font>中来隔离<font color="green">_BaseColor</font>，尽管它仍然可以在全局级别上访问。

```glsl
cbuffer UnityPerMaterial {
	float _BaseColor;
};
```

Unity没有直接提供MVP矩阵，而是拆开成提供两个矩阵M和VP是因为VP矩阵在一帧中不会改变，可以重复利用。

Unity将M矩阵和VP矩阵存入Constant [Buffer](https://so.csdn.net/so/search?q=Buffer&spm=1001.2101.3001.7020)中以提高运算效率

>   M矩阵存入的buffer为<font color="RoyalBlue">UnityPerDraw</font> buffer, 也就是针对每个物体的绘制不会改变。
>
>   VP矩阵则存入的是<font color="blue">UnityPerFrame </font>buffer，即每一帧VP矩阵并不会改变。
>
>   使用cbuffer keyworld来引入Constant buffer，constant buffer中还有很多其他的数据，暂时我们用不到，只使用这两个矩阵
>

![img](https://img-blog.csdnimg.cn/2019060417385086.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpbmZvdXJldmVy,size_16,color_FFFFFF,t_70)

#### 2.1.2、Core Buffer

[**Constant buffers**](https://docs.unity3d.com/ScriptReference/ShaderData.ConstantBufferInfo.html)并不是在所有的平台上都支持的，比如OpenGL ES 2.0

所以我们可以使用Core RP库中的<font color="*MidnightBlue">CBUFFER_START</font>和<font color="*MidnightBlue">CBUFFER_END</font>宏，而不是直接使用<font color="green">cbuffer</font>。

<font color="Royalblue">第一个宏把缓冲区名称作为参数，就像它是一个函数一样。在这种情况下，我们最终得到的结果和之前的完全一样，只是在不支持<font color="DarkOrchid ">cbuffer</font>的平台上，<font color="green">cbuffer</font>代码将不存在。</font>

```glsl
CBUFFER_START(UnityPerMaterial)
	float4 _BaseColor;
CBUFFER_END
```

我们也必须对<font color="green">unity_ObjectToWorld、unity_WorldToObject和unity_WorldTransformParams</font>、这样做，只不过它们必须被分组在<font color="red">**UnityPerDraw**</font>缓冲区中。

```glsl
CBUFFER_START(UnityPerDraw)
	float4x4 unity_ObjectToWorld;
	float4x4 unity_WorldToObject;
	real4 unity_WorldTransformParams;
CBUFFER_END
```

在这种情况下，如果我们使用其中的一个，我们就需要定义特定的值组。<font color="red">对于变换组，我们还需要包括float4 unity_LODFade，尽管我们并没有使用它。</font>

<font color="green">具体的顺序并不重要，但是Unity把它直接放在unity_WorldToObject之后</font>，所以我们也要这样做。

```glsl
CBUFFER_START(UnityPerDraw)
	float4x4 unity_ObjectToWorld;
	float4x4 unity_WorldToObject;
	float4 unity_LODFade;
	real4 unity_WorldTransformParams;
CBUFFER_END
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/compatible.png" alt="img" style="zoom:50%;" />

<center>与 SRP 批处理程序兼容</center>

在我们的<font color="green">  Shader</font>兼容的情况下，下一步是启用 <font color="green">SRP batcher</font>，这可以通过设置<font color="green">GraphicsSettings</font>.<font color="DarkOrchid ">useScriptableRenderPipelineBatching</font>为<font color="RoyalBlue">true</font>来完成。我们只需要做一次，所以让我们在创建管道实例时进行，为<font color="green">CustomRenderPipeline</font>添加一个构造方法。

```c#
	public CustomRenderPipeline () {
		GraphicsSettings.useScriptableRenderPipelineBatching = true;
	}
```

[GraphicsSettings](https://docs.unity3d.com/ScriptReference/Rendering.GraphicsSettings.html) <font color="green">useScriptableRenderPipelineBatching </font>在运行时启用/禁用 SRP 批处理程序（实验性）。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/statistics-srp-batching.png" alt="img" style="zoom: 50%;" />

<center>*Negative batches saved.*</center>		

统计面板显示有76个批次被保存，尽管它显示的是一个负数。帧调试器现在在<font color="green">RenderLoopNewBatcher</font>.<font color="RoyalBlue">Draw</font>下显示了一个单一的<font color="red">*SRP Batch*</font>条目，不过请记住，这不是一个单一的绘制调用，而是一个优化的序列。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/srp-batch.png" alt="img" style="zoom:67%;" />

<center>*SRP Batch*</center>

#### [2.1.2、Many Colors](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.2)

尽管我们使用了四个材料，但我们得到了一个批次。这是因为它们的所有数据都被缓存在GPU上，每个绘制调用只需要包含一个偏移到正确的内存位置。

唯一的限制是每个材质的内存布局必须是相同的，这是因为我们对所有的材质使用相同的<font color="green">  Shader</font>，每个材质只包含一个颜色属性。Unity不会比较材质的确切内存布局，它只是将使用完全相同的<font color="green"> shader variant.</font>的绘制调用进行分组。

如果我们想要一些不同的颜色，这样做很好，但是如果我们想要给每个球体提供自己的颜色，那么我们就必须创建更多的材质。如果我们可以为每个对象设置颜色，那就更方便了。

这在默认情况下是不可能的，但我们可以通过创建一个<font color="**Royal**Blue">自定义组件类型（ custom component type）</font>来支持它。把它命名为<font color="**Royal**Blue">PerObjectMaterialProperties</font>。因为它是一个例子，我把它放在自定义RP下的Examples文件夹里。

我们的想法是，一个游戏对象可以有一个<font color="red">PerObjectMaterialProperties</font>组件附在它身上，它有一个<font color="green">Base Color</font>配置选项，这将被用来为它设置<font color="green">_BaseColor</font>材质属性。

它需要知道<font color="green">  Shader</font>属性的标识符，我们可以通过<font color="red">Shader.PropertyToID</font>检索并存储在一个静态变量中，就像我们在<font color="green">CameraRenderer</font>中对<font color="green">  Shader</font>通道标识符所做的那样，尽管在这种情况下它是一个整数。

```c#
using UnityEngine;

[DisallowMultipleComponent]
public class PerObjectMaterialProperties : MonoBehaviour {
	
	static int baseColorId = Shader.PropertyToID("_BaseColor");
	
	[SerializeField]
	Color baseColor = Color.white;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/per-object-material-properties.png" alt="img" style="zoom: 50%;" />

<font color="green">PerObjectMaterialProperties component.</font>

设置每个对象的材料属性是通过一个<font color="RoyalBlue">MaterialPropertyBlock对象完成的</font>。我们只需要一个所有<font color="green">PerObjectMaterialProperties</font>实例都能重用的对象，所以为它声明一个静态字段。

```c#
	static MaterialPropertyBlock block;
```

>   [MaterialPropertyBlock](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.html)
>
>   MaterialPropertyBlock 由[Graphics.DrawMesh](https://docs.unity3d.com/ScriptReference/Graphics.DrawMesh.html)和[Renderer.SetPropertyBlock](https://docs.unity3d.com/ScriptReference/Renderer.SetPropertyBlock.html)使用。在您想要绘制具有相同材质但属性略有不同的多个对象的情况下使用它。例如，如果你想稍微改变绘制的每个网格的颜色。不支持更改渲染状态。
>

如果还没有，请创建一个新块，然后<font color="green">SetColor</font>使用属性标识符和颜色调用它，然后通过 将该块应用于游戏对象的<font color="green">Renderer</font>组件<font color="green">SetPropertyBlock</font>，这将复制其设置。执行此操作，<font color="green">OnValidate</font>以便结果立即显示在编辑器中。

```c#
	void OnValidate () {
		if (block == null) {
			block = new MaterialPropertyBlock();
		}
		block.SetColor(baseColorId, baseColor);
		GetComponent<Renderer>().SetPropertyBlock(block);
	}
```

>   什么时候调用OnValidate？
>   当组件被加载或改变时，OnValidate会在Unity编辑器中被调用。所以每次加载场景和我们编辑组件时都会调用OnValidate。因此，各个颜色会立即出现并对编辑作出反应。
>
>   [OnValidate](https://docs.unity3d.com/ScriptReference/MonoBehaviour.OnValidate.html)在加载或更改组件时在 Unity 编辑器中调用。所以每次加载场景和编辑组件时。因此，各个颜色会立即出现并响应编辑。

将组件添加到 24 个任意球体并给它们不同的颜色。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/many-colors.png" alt="img" style="zoom:50%;" />

<center>很多颜色。</center>

<font color="red">不幸的是，SRP 批处理程序无法处理每个对象的<font color="green"> material properties</font>。</font>

因此，这 24 个球体每个都退回到一个常规绘制调用，由于排序，也可能将其他球体分成多个批次。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/24-non-batched.png" alt="img" style="zoom:50%;" />

<center>24 non-batched draw calls.</center>

此外，<font color="green">OnValidate</font>不会在构建中调用。为了使单独的颜色出现在那里，我们还必须在 中应用它们<font color="green">Awake</font>，我们可以通过简单地<font color="green">OnValidate</font>在那里调用来完成。

```c#
	void Awake () {
		OnValidate();
	}
```

## [2.3、GPU Instancing](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.3)

还有另一种方法可以整合绘制调用，它确实适用于每个对象的<font color="red">材质属性（per-object material properties）</font>。

它被称为<font color="green"> GPU instancing </font>

<font color="red">其工作方式是对具有<font color="green">相同网格的多个对象</font>同时发出一个绘制调用。<font color="green">CPU收集所有每个对象的变换和材质属性，并将其放入数组</font>，然后发送给GPU。然后，GPU遍历所有条目，并按照所提供的顺序进行渲染。</font>

因为<font color="red">**GPU实例需要通过数组提供数据**</font>，所以我们的<font color="green">  Shader</font>目前不支持它。要实现这一点，首先要在Shader Pass块中的<font color="green">pragmas vertex和pragmas fragment </font>之上添加<font color="RoyalBlue">#pragma multi_compile_instancing</font>指令。

```glsl
	#pragma multi_compile_instancing
	#pragma vertex UnlitPassVertex
	#pragma fragment UnlitPassFragment
```

这将使 Unity 生成我们<font color="green">  Shader</font>的两种变体，一种具有 GPU 实例化支持，一种不具有 GPU 实例化支持。材质检查器中还出现了一个切换选项，允许我们选择每种材质使用哪个版本。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/gpu-instancing-material.png" alt="img" style="zoom:67%;" />

<center>启用了 GPU 实例化的材质。</center>

支持<font color="green"> GPU instancing </font>需要改变方法，为此我们必须在<font color="DarkOrchid ">core shader library</font>中加入<font color="green">UnityInstancing.hlsl</font>文件。这必须在定义<font color="RoyalBlue">UNITY_MATRIX_M</font>和其他宏之后，在包含<font color="green">SpaceTransforms.hlsl</font>之前完成。

```c#
#define UNITY_MATRIX_P glstate_matrix_projection

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```

#### 2.3.1、Shader struct

<font color="red">**UnityInstancing.hlsl所做的是重新定义这些宏，以代替访问实例数据数组。**</font>

但是为了使其发挥作用，它需要知道当前正在渲染的对象的索引<font color="green">Index</font>。<font color="red">这个索引是通过顶点数据提供的</font>，所以我们必须让它可用。<font color="green">UnityInstancing.hlsl</font>定义了一些宏来使之变得简单，但它们假定我们的顶点函数有一个结构参数。

我们可以声明一个<font color="green">**struct**--就像**cbuffer**</font>一样--并将其作为一个函数的输入参数。

我们还可以在结构内部定义语义。这种方法的好处是，它比长长的参数列表更容易辨认。因此，<font color="red">将UnlitPassVertex的positionOS参数包裹在一个Attributes **struct**中，代表顶点输入数据</font>。

```c#
struct Attributes {
	float3 positionOS : POSITION;
};

float4 UnlitPassVertex (Attributes input) : SV_POSITION {
	float3 positionWS = TransformObjectToWorld(input.positionOS);
	return TransformWorldToHClip(positionWS);
}
```

当使用<font color="green"> GPU instancing </font>时，<font color="red">**对象索引也可以作为一个顶点属性**</font>使用。我们可以在适当的时候添加它，只需将<font color="RoyalBlue">**UNITY_VERTEX_INPUT_INSTANCE_ID**</font>放在Attributes里面。

```glsl
struct Attributes {
	float3 positionOS : POSITION;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

接下来，在UnlitPassVertex的开头添加<font color="RoyalBlue">UNITY_SETUP_INSTANCE_ID(input); </font>。

<font color="red">这将从输入中提取索引，并将其存储在一个全局静态变量中，其他实例化宏都依赖于此</font>。

```c#
float4 UnlitPassVertex (Attributes input) : SV_POSITION {
	UNITY_SETUP_INSTANCE_ID(input);
	float3 positionWS = TransformObjectToWorld(input.positionOS);
	return TransformWorldToHClip(positionWS);
}
```

<font color="DarkOrchid ">这足以让<font color="green">GPU instancing</font>工作，尽管<font color="green"> SRP batcher</font>优先</font>

所以我们现在不会得到不同的结果。但是我们还不支持每个实例的材质数据。

为了增加这一点，<font color="red">我们必须在需要时用一个数组引用替换_BaseColor</font>。

这是通过用<font color="RoyalBlue">**UNITY_INSTANCING_BUFFER_START**</font>替换<font color="RoyalBlue">CBUFFER_START</font>，用<font color="RoyalBlue">**UNITY_INSTANCING_BUFFER_END**</font>替换<font color="RoyalBlue">CBUFFER_END</font>来实现的，现在它也需要一个参数。这不需要和开始时一样，但没有令人信服的理由让它们不同。

```glsl
//CBUFFER_START(UnityPerMaterial)
//	float4 _BaseColor;
//CBUFFER_END

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	float4 _BaseColor;
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

然后用<font color="RoyalBlue">UNITY_DEFINE_INSTANCED_PROP(float4, BaseColor)</font>替换<font color="green">BaseColor</font>的定义。

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	//	float4 _BaseColor;
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

当使用实例化时，我们现在还必须使实例索引在<font color="green">UnlitPassFragment</font>中可用。为了方便起见，我们将使用一个 <font color="green">struct </font>让<font color="green">UnlitPassVertex</font>同时输出位置和索引，使用<font color="RoyalBlue">**UNITY_TRANSFER_INSTANCE_ID**(input, output); </font><font color="red">在索引存在时复制它</font>。

我们像Unity那样将这个 <font color="green">struct</font> 命名为<font color="RoyalBlue">**Varyings**</font>，因为它包含的数据可以在同一个三角形的片段之间变化。

```c#
struct Attributes {
    float3 positionOS : POSITION;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
struct Varyings {
	float4 positionCS : SV_POSITION;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex (Attributes input) { //: SV_POSITION {
	Varyings output;
	UNITY_SETUP_INSTANCE_ID(input);
	UNITY_TRANSFER_INSTANCE_ID(input, output);
	float3 positionWS = TransformObjectToWorld(input.positionOS);
	output.positionCS = TransformWorldToHClip(positionWS);
	return output;
}
```

将此结构作为参数添加到 <font color="green">UnlitPassFragment </font>中。然后像以前一样使用<font color="RoyalBlue">UNITY_SETUP_INSTANCE_ID</font>来使索引可用。现在必须通过<font color="RoyalBlue">**UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor)**</font>访问材质属性。

```glsl
float4 UnlitPassFragment (Varyings input) : SV_TARGET {
	UNITY_SETUP_INSTANCE_ID(input);
	return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/instanced-draw-calls.png" alt="img" style="zoom: 50%;" />

<center><font color="green">实例化绘制调用。</font></center>

Unity 现在能够将 24 个球体与每个对象的颜色相结合，从而减少绘制调用的数量。我最终得到了四个实例化的绘制调用，因为这些球体仍然使用其中的四种材质。

<font color="red">GPU 实例化仅适用于共享相同材质的对象。</font>

当它们覆盖<font color="green"> material color</font>时，它们都可以使用相同的<font color="green">material</font>，这样就可以在一个批次中绘制它们。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/one-instanced-material.png" alt="img" style="zoom: 50%;" />

<center><font color="green">一种实例材料。</font></center>

请注意，根据目标平台和每个实例必须提供的数据量，批处理大小有限制。如果你超过这个限制，那么你最终会得到不止一批。此外，如果有多种材料在使用中，排序仍然可以拆分批次。

#### [2.4、Drawing Many Instanced Meshes](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.4)

当数百个对象可以在一次绘制调用中组合起来时，<font color="green"> GPU instancing </font>就成为一个显著的优势。但是用手编辑场景中的这么多对象并不实际。

所以让我们随机生成一堆。创建一个<font color="green">MeshBall</font>的例子组件，当它被唤醒的时候会产生大量的对象。让它缓存<font color="green">_BaseColor</font><font color="green">  Shader</font>属性，并为网格和材质添加配置选项，这些选项必须支持实例化。

```c#
using UnityEngine;

public class MeshBall : MonoBehaviour {

	static int baseColorId = Shader.PropertyToID("_BaseColor");

	[SerializeField]
	Mesh mesh = default;

	[SerializeField]
	Material material = default;
}
```

用这个组件创建一个游戏对象。我给了它默认的球体网格来绘制。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/mesh-ball-component.png" alt="img" style="zoom:67%;" />

<center>Mesh ball component for spheres.</center>

我们可以生成许多新的游戏对象，但我们没有必要这样做。相反，我们将填充一个变换矩阵和颜色的数组，并告诉GPU用这些来渲染一个网格,这就是<font color="green"> GPU instancing </font>最有用的地方。

我们可以一次性提供多达1023个实例，所以让我们添加具有这个长度的数组的字段，以及一个我们需要传递颜色数据的<font color="green">MaterialPropertyBlock</font>。在这种情况下，颜色数组的元素类型必须是Vector4。

创建一个<font color="green">Awake</font>方法，用半径为 10 的球体和随机 RGB 颜色数据中的随机位置填充数组。

```c#
	void Awake () {
		for (int i = 0; i < matrices.Length; i++) {
			matrices[i] = Matrix4x4.TRS(
				Random.insideUnitSphere * 10f, Quaternion.identity, Vector3.one
			);
			baseColors[i] =
				new Vector4(Random.value, Random.value, Random.value, 1f);
		}
	}
```

在Update我们创建一个**[new block](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.html)（**如果它不存在）并调用<font color="green">SetVectorArray</font>它来配置颜色。

之后<font color="RoyalBlue">Graphics.DrawMeshInstanced</font>以网格、子网格索引零、材料、矩阵数组、元素数量和属性块作为参数调用。我们在这里设置块，以便网格球在热重载中幸存下来。

```c#
	void Update () {
		if (block == null) {
			block = new MaterialPropertyBlock();
			block.SetVectorArray(baseColorId, baseColors);
		}
		Graphics.DrawMeshInstanced(mesh, 0, material, matrices, 1023, block);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/mesh-ball.png" alt="img" style="zoom:50%;" />

<center>1023 spheres, 3 draw calls.</center>

进入游戏模式现在会产生一个密集的球体。它需要多少次绘制调用取决于平台，因为每个绘制调用的最大缓冲区大小不同。在我的例子中，渲染需要三个绘制调用。

请注意，各个网格的绘制顺序与我们提供数据的顺序相同。除此之外没有任何类型的排序或剔除，尽管整个批次一旦超出视锥体就会消失。

## [2.5、Dynamic Batching](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.5)

还有一种减少绘制调用的方法，被称为<font color="green">dynamic batching</font>。

这是一种古老的技术，它将共享相同材质的多个小网格组合成一个较大的网格，然后被绘制。当使用<font color="RoyalBlue">per-object material properties</font>时，这也是不可行的。

较大的网格是按需生成的，所以它只对小网格可行。球体太大，但它对立方体是有效的。要看到它的作用，请禁用<font color="green"> GPU instancing </font>，并在<font color="green">CameraRenderer</font>.<font color="red">DrawVisibleGeometry</font>中设置<font color="green">enableDynamicBatching</font>为<font color="RoyalBlue">true</font>

```c#
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		) {
			enableDynamicBatching = true,
			enableInstancing = false
		};
```

同时在<font color="green">CustomRenderPipeline</font>中禁用 <font color="RoyalBlue"> SRP batcher</font>，因为它具有优先权。

```c#
GraphicsSettings.useScriptableRenderPipelineBatching = false;
```

![](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/cubes.png)

<center>Drawing cubes instead.</center>

一般来说，<font color="green"> GPU instancing </font>比<font color="green">dynamic batching.</font>效果更好。

这种方法也有一些注意事项，例如，当涉及到不同比例时，较大的网格的法向量不能保证是单位长度的。另外，绘制顺序也会发生变化，因为现在是单个网格而不是多个。

还有一种<font color="green"> static batching</font>，它的工作原理与此类似，但对于被标记为批处理-静态的对象，它是提前进行批处理的。

除了需要更多的内存和存储，它没有任何注意事项。RP没有意识到这一点，所以我们不必担心这个问题。

#### [2.6、Configuring Batching](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#2.6)

哪种方法是最好的，可能会有所不同，所以让我们让它变得可配置。首先，添加布尔参数来控制<font color="red">dynamic batching and GUI instancing</font>是否被用于<font color="green">DrawVisibleGeometry</font>，而不是硬编码。

```c#
void DrawVisibleGeometry (bool useDynamicBatching, bool useGPUInstancing) {
		var sortingSettings = new SortingSettings(camera) {
			criteria = SortingCriteria.CommonOpaque
		};
		var drawingSettings = new DrawingSettings(
			unlitShaderTagId, sortingSettings
		) {
			enableDynamicBatching = useDynamicBatching,
			enableInstancing = useGPUInstancing
		};
		…
	}
```

<font color="green">Render </font>现在必须提供这种配置，反过来又要依靠RP来提供。

```c#
	public void Render (
		ScriptableRenderContext context, Camera camera,
		bool useDynamicBatching, bool useGPUInstancing
	) {
		…
		DrawVisibleGeometry(useDynamicBatching, useGPUInstancing);
		…
	}
```

<font color="green">CustomRenderPipeline</font>将通过字段跟踪选项，在它的构造方法中设置，并在<font color="green">Render</font>中传递它们。同时在构造函数中为<font color="green">  SRP batcher</font>添加一个bool参数，而不是总是启用它。

```c#
	bool useDynamicBatching, useGPUInstancing;

	public CustomRenderPipeline (
		bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher
	) {
		this.useDynamicBatching = useDynamicBatching;
		this.useGPUInstancing = useGPUInstancing;
		GraphicsSettings.useScriptableRenderPipelineBatching = useSRPBatcher;
	}

	protected override void Render (
		ScriptableRenderContext context, Camera[] cameras
	) {
		foreach (Camera camera in cameras) {
			renderer.Render(
				context, camera, useDynamicBatching, useGPUInstancing
			);
		}
	}
```

最后，将这三个选项作为配置字段添加到<font color="green">CustomRenderPipelineAsset</font>中，将它们传递给<font color="red">CreatePipeline</font>的构造函数调用。

```c#
	[SerializeField]
	bool useDynamicBatching = true, useGPUInstancing = true, useSRPBatcher = true;

	protected override RenderPipeline CreatePipeline () {
		return new CustomRenderPipeline(
			useDynamicBatching, useGPUInstancing, useSRPBatcher
		);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/batching/rp-configuration.png" alt="img" style="zoom:50%;" />

<center>	RP配置。</center>

现在可以更改我们的 RP 使用的方法。切换选项会立即生效，因为 Unity 编辑器会在检测到资产发生更改时创建一个新的 RP 实例。

### [3、Transparency](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3)

我们的<font color="green">  Shader</font>可用于创建<font color="green"> unlit opaque materials</font>。可以更改颜色的 <font color="green">alpha </font>分量，这通常表示透明度，但目前没有效果。我们还可以将渲染队列设置为<font color="RoyalBlue">Transparent</font>，但这只会在绘制对象时以及以什么顺序而不是如何绘制时发生变化。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/reduced-alpha.png" alt="img" style="zoom:67%;" />

<center>减少 alpha 并使用透明渲染队列。</center>

#### [3.1、Blend Modes](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.1)

不透明和透明渲染的主要区别在于，我们是替换之前绘制的任何东西，还是与之前的结果结合起来产生透视效果。我们可以通过设置<font color="DarkOrchid ">source和destination</font>的<font color="RoyalBlue">Blend Modes</font>来控制这一点。

这里的<font color="green">source</font>指的是现在被绘制的东西，而<font color="red">destination</font>指的是之前被绘制的东西，以及结果最终会在哪里出现。为此添加两个<font color="green">  Shader</font>属性。<font color="green">_SrcBlend和_DstBlend</font>。它们是混合模式的枚举，但我们可以使用的最佳类型是Float，默认情况下，源点设置为1，终点设置为0。

```glsl
	Properties {
		_BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
		_SrcBlend ("Src Blend", Float) = 1
		_DstBlend ("Dst Blend", Float) = 0
	}
```

为了使编辑更容易，我们可以将<font color="blue">Enum</font>属性添加到属性中，并将完全限定的枚举类型作为参数。<font color="green">UnityEngine.Rendering.BlendMode</font>

```glsl
[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend ("Src Blend", Float) = 1
[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend ("Dst Blend", Float) = 0
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/opaque-blend-modes.png" alt="img" style="zoom:50%;" />

<center>不透明混合模式。</center>

默认值表示我们已经使用的不透明混合配置。源设置为 1，意味着它被完全添加，而目标设置为零，意味着它被忽略。

标准透明度的源混合模式是<font color="green">SrcAlpha</font>，这意味着渲染颜色的 RGB 分量乘以它的 <font color="blue">alpha </font>分量。所以 alpha 越低，它就越弱。然后将目标混合模式设置为反向：<font color="RoyalBlue">OneMinusSrcAlpha</font>，以达到总权重 1。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/transparent-blend-modes.png" alt="img" style="zoom:50%;" />

​																				透明混合模式。

混合模式可以在<font color="green">Pass</font>块中定义，<font color="blue">Blend</font>语句后跟两种模式。我们想要使用<font color="blue"> shader properties</font>，我们可以通过将它们放在方括号内来访问它们。这是可编程<font color="green">  Shader</font>出现之前的旧语法。

```glsl
	Pass {
			Blend [_SrcBlend] [_DstBlend]

			HLSLPROGRAM
			…
			ENDHLSL
		}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/semitransparent-spheres.png" alt="img" style="zoom:50%;" />

<center>*Semitransparent yellow spheres.*</center>

#### [3.2、Write Depth and Texturing](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.2)

透明渲染通常不会写入<font color="RoyalBlue"> depth buffer </font>，因为它不会从中受益，甚至可能产生不希望的结果。

<font color="green">ZWrite</font>我们可以通过语句来控制是否写入深度。我们可以再次使用<font color="green">  Shader</font>属性，这次使用<font color="blue">_ZWrite</font>.

```glsl
			Blend [_SrcBlend] [_DstBlend]
			ZWrite [_ZWrite]
```

使用自定义属性定义<font color="green">  Shader</font>属性，以创建默认打开的值 0 和 1 的开关切换<font color="DarkOrchid "> Enum(Off, 0, On, 1)</font>

```glsl
		[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend ("Src Blend", Float) = 1
		[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend ("Dst Blend", Float) = 0
		[Enum(Off, 0, On, 1)] _ZWrite ("Z Write", Float) = 1
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/z-write-off.png" alt="img" style="zoom:50%;" />

<center>	书写深度关闭。</center>

#### 3.3、Texturing

之前我们使用<font color="green">alpha Texture</font>来创建一个非均匀的半透明材质。让我们通过给<font color="green">  Shader</font>添加一个<font color="RoyalBlue">_BaseMap</font>纹理属性来支持它

。在这种情况下，类型是2D，我们将使用Unity的标准白色纹理作为默认，用白色字符串表示。另外，我们必须用一个空的代码块来结束纹理属性。

很久以前，它是用来控制纹理设置的，但今天仍应包括在内，以防止在某些情况下出现奇怪的错误。

```glsl
	_BaseMap("Texture", 2D) = "white" {}
	_BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
```

纹理必须被上传到GPU内存中，Unity为我们做了这个。

<font color="green">  Shader</font>需要一个相关纹理的<font color="green">handle</font> ，我们可以像定义一个统一的值一样定义这个<font color="red">handle </font>，只不过我们使用<font color="blue">TEXTURE2D</font>宏，把名称作为一个参数。我们还需要为纹理定义一个采样器状态，考虑到它的<font color="blue">wrap and filter </font>模式，控制它应该如何采样。

这是用<font color="green">SAMPLER</font> 宏来完成的，就像<font color="blue">TEXTURE2D</font>一样，但名字前加了<font color="green">sampler</font>。这与Unity自动提供的采样器状态的名称一致。

<font color="red">**纹理和采样器状态**(Textures and sampler )是shader资源。**它们不能为每个实例提供**，必须在全局范围内声明</font>。在<font color="green">UnlitPass.hlsl</font>的<font color="green">  Shader</font>属性 <font color="RoyalBlue">shader properties</font>之前做这个。

```glsl
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

除此之外，Unity还通过一个<font color="green">float4</font>来提供纹理的平铺和偏移，这个<font color="green">float4</font>的名字与纹理属性相同，但附加了_ST，它代表缩放和平移或类似的东西。

这个属性应该是<font color="RoyalBlue">UnityPerMaterial</font>缓冲区的一部分，因此<font color="red">可以对每个实例进行设置。</font>

```glsl
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseMap_ST)
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

为了对纹理进行采样，我们需要纹理坐标<font color="red">（ texture coordinates）</font>，它是顶点属性的一部分。

具体来说，我们需要第一对坐标，因为还可能有更多。这可以通过向<font color="green">Attributes</font>添加一个具有<font color="blue">TEXCOORD0</font>含义的<font color="green">float2</font>字段来实现。由于这是我们的基础贴图，而纹理空间的尺寸普遍被命名为U和V，我们将其命名为<font color="green">baseUV</font>。

```c#
struct Attributes {
	float3 positionOS : POSITION;
	float2 baseUV : TEXCOORD0;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

我们需要把坐标传给<font color="RoyalBlue">fragment</font>函数，因为纹理就是在那里被采样的。所以也要在<font color="green">Varyings</font>中添加<font color="green">float2 baseUV</font>。这一次我们不需要添加特殊的含义，它只是我们传递的数据，不需要GPU的特别关注。

然而，我们仍然要给它附加一些意义。我们可以应用任何不用的标识符，让我们简单地使用<font color="RoyalBlue">**VAR_BASE_UV**</font>。

```glsl
struct Varyings {
	float4 positionCS : SV_POSITION;
	float2 baseUV : VAR_BASE_UV;
	UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

当我们复制<font color="green">UnlitPassVertex</font>中的坐标时，我们也可以应用存储在<font color="blue">_BaseMap_ST</font>中的比例和偏移。这样我们就可以按<font color="blue">顶点（Vertex）</font>而不是按<font color="blue">片断(fragment)</font>来做。scale存储在XY，偏移存储在ZW，我们可以通过swizzle属性访问。

```glsl
Varyings UnlitPassVertex (Attributes input) {
	…

	float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseMap_ST);
	output.baseUV = input.baseUV * baseST.xy + baseST.zw;
	return output;
}
```

<font color="red">现在UV坐标可供<font color="blue">UnlitPassFragment</font>使用，在整个三角形内插。</font>

在这里对纹理进行采样，使用<font color="blue">SAMPLE_TEXTURE2D</font>宏，将<font color="green">纹理、采样器状态和坐标作为参数（ texture, sampler state, and coordinates as arguments</font>）。

最终的颜色是通过乘法将纹理和统一颜色结合起来。两个相同大小的向量相乘的结果是所有匹配的分量都被相乘，所以在这种情况下，红色乘以红色，绿色乘以绿色，依此类推。

```c#
float4 UnlitPassFragment (Varyings input) : SV_TARGET {
	UNITY_SETUP_INSTANCE_ID(input);
	float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
	float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
	return baseMap * baseColor;
}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/textured.png" alt="img" style="zoom:50%;" />

<center>*Textured yellow spheres.*</center>

因为我们纹理的 RGB 数据是均匀的白色，所以颜色不受影响。但 alpha 通道不同，因此透明度不再统一。

#### [3.4、Alpha Clipping](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.4)

另一种透视表面的方法是在表面上开孔。<font color="green">  Shader</font>也可以通过丢弃一些它们通常会渲染的片段来做到这一点。

这会产生硬边，而不是我们目前看到的平滑过渡。这种技术称为<font color="green"> Alpha Clip</font>。完成此操作的常用方法是定义<font color="red">裁剪阈值**cutoff threshold**</font>。

alpha 值低于此阈值的片段将被丢弃，而所有其他片段将被保留。

添加<font color="red">_Cutoff</font>默认设置为 0.5 的属性。

由于 alpha 总是介于 0 和 1 之间，我们可以使用它作为它的类型。<font color="green">Range</font>(<font color="Pink">0.0, 1.0</font>)

```glsl
		_BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
		_Cutoff ("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
```

将其添加到材料属性中<font color="RoyalBlue">UnlitPass.hlsl</font>以及。

```glsl
	UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
	UNITY_DEFINE_INSTANCED_PROP(float, _Cutoff)
```

我们可以通过调用<font color="green">**UnlitPassFragment**</font> 中的<font color="red">clip</font>函数来丢弃片段。

如果我们传递给它的值为零或更小，它将中止并丢弃该片段。因此，将最终的 alpha 值（可通过a或w属性访问）传递给它减去截止阈值。

```c#
	float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, input.baseUV);
	float4 baseColor = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
	float4 base = baseMap * baseColor;
	clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
	return base;
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-inspector.png" alt="检查员" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-scene.png" alt="场景" style="zoom:50%;" />

<center>Alpha 截止设置为 0.2。</center>

<font color="red">一个材质通常使用<font color="blue">transparency blending</font> 或者是 <font color="blue">alpha clipping,</font>，而不是同时使用两者。</font>

一个典型的<font color="green"> clip materia</font>是完全不透明的，除了被丢弃的片段之外，它的确会向<font color="red">depth buffer</font>写入数据。

它使用<font color="blue">AlphaTest</font>渲染队列，这意味着它在所有完全不透明的对象之后被渲染。

之所以这样做，是因为丢弃碎片会使一些GPU优化变得不可能，因为不能再假设三角形完全覆盖它们后面的东西。通过先绘制完全不透明的物体，它们最终可能会覆盖部分<font color="green">alpha-clipped</font>物体，这样就不需要处理它们的隐藏片段了。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/clipped-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/clipped-scene.png" alt="scene" style="zoom:50%;" />

<center>Alpha-clipped material.</center>

但是，为了使这种优化发挥作用，我们必须确保剪辑只在需要的时候使用。我们将通过添加一个<font color="green">toggle</font>切换<font color="RoyalBlue">shader property</font>来做到这一点。

这是一个默认设置为0的Float属性，有一个控制<font color="green">  Shader</font>关键字的Toggle属性，我们将使用<font color="RoyalBlue">_CLIPPING</font>。属性本身的名字并不重要，所以简单地使用<font color="RoyalBlue">_Clipping</font>。

```c#
	_Cutoff ("Alpha Cutoff", Range(0.0, 1.0)) = 0.5
	[Toggle(_CLIPPING)] _Clipping ("Alpha Clipping", Float) = 0
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/alpha-clipping-off.png" alt="img" style="zoom:50%;" />

<center>Alpha clipping turned off, supposedly.</center>

### [3.5、Shader Features](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.5)

<font color="red">无论是否开启会生成两个变体</font>

<font color="green">the toggle</font>的切换将把<font color="green">_CLIPPING</font>关键字添加到材料的<font color="RoyalBlue">keywords</font>列表中，当禁用则会remove它。

但这本身并没有什么作用。我们必须告诉Unity根据关键字是否被定义来编译不同版本的<font color="green">  Shader</font>。我们通过在其Pass中的指令中添加<font color="OrangeRed">#pragma shader_feature _CLIPPING</font>来做到这一点。

```glsl
	// Shader Features
	#pragma shader_feature _CLIPPING
	//GPU Instancing 
	#pragma multi_compile_instancing
```

现在，Unity将编译我们的<font color="green">  Shader</font>代码，不管是否定义了<font color="green">_CLIPPING</font>。它将生成一个或两个变体，这取决于我们如何配置我们的材质。_

所以我们可以让我们的代码以定义为条件，就像<font color="RoyalBlue">include guards</font>一样，但在这种情况下，我们只想在定义了<font color="DarkOrchid ">CLIPPING</font>的情况下包含 <font color="green">clipping line</font>。

我们可以为此使用#<font color="DarkOrchid ">ifdef CLIPPING</font>，但我更喜欢<font color="RoyalBlue">#if defined(CLIPPING)。</font>

```c#
#if defined(_CLIPPING)
		clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
	#endif
```

#### 3.6、[Cutoff Per Object](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.6)

由于<font color="red">cutoff </font>是<font color="green">UnityPerMaterial buffer</font>的一部分，它可以按实例进行配置。所以让我们把这个功能添加到<font color="green">PerObjectMaterialProperties</font>中。

它与颜色的工作原理相同，只是我们需要在<font color="green">property block</font>上调用SetFloat而不是SetColor。

```c#
static int baseColorId = Shader.PropertyToID("_BaseColor");
	static int cutoffId = Shader.PropertyToID("_Cutoff");

	static MaterialPropertyBlock block;

	[SerializeField]
	Color baseColor = Color.white;

	[SerializeField, Range(0f, 1f)]
	float cutoff = 0.5f;

	…

	void OnValidate () {
		…
		block.SetColor(baseColorId, baseColor);
		block.SetFloat(cutoffId, cutoff);
		GetComponent<Renderer>().SetPropertyBlock(block);
	}
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-per-object-inspector.png" alt="inspector" style="zoom:50%;" />

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/cutoff-per-object-scene.png" alt="scene" style="zoom:50%;" />

<center>Alpha cutoff per instanced object.</center>

#### [3.7、Ball of Alpha-Clipped Spheres](https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/#3.7)

同样的情况也适用于**MeshBall**。现在，我们可以使用 <font color="RoyalBlue">clip material</font>，但所有实例最终都有完全相同的孔。

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/alpha-clipped-ball.png" alt="img" style="zoom:50%;" />

<center>Alpha-clipped mesh ball up close.</center>

让我们通过给每个实例一个随机的旋转，加上0.5-1.5范围内的随机统一比例来增加一些变化。但我们不是为每个实例设置截止点，而是在0.5-1范围内改变它们的颜色的alpha通道。这给了我们不太精确的控制，但无论如何，这是一个随机的例子。

```c#
	matrices[i] = Matrix4x4.TRS(
				Random.insideUnitSphere * 10f,
				Quaternion.Euler(
					Random.value * 360f, Random.value * 360f, Random.value * 360f
				),
				Vector3.one * Random.Range(0.5f, 1.5f)
			);
			baseColors[i] =
				new Vector4(
					Random.value, Random.value, Random.value,
					Random.Range(0.5f, 1f)
				);
```

<img src="https://catlikecoding.com/unity/tutorials/custom-srp/draw-calls/transparency/more-varied-ball.png" alt="img" style="zoom:50%;" />

<center>	More varied mesh ball.</center>

请注意，Unity 最终仍会向 GPU 发送一组截止值，每个实例一个，即使它们完全相同。该值是材质的副本，因此通过改变它可以一次更改所有球体的孔，即使它们仍然不同。

## Last Code Strct

<font color="green">CustomRenderPipelineAsset.CreatePipeline () </font> =>

<font color="green">CustomRenderPipeline. Render</font> =><font color="red"> foreach Camera.render </font>=>

<font color="red">CameraRenderer.Render()</font>  =>

{

}