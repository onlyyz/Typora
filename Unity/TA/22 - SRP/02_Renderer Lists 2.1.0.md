# Custom SRP 2.1.0

使用<font color=#66ff66>renderer lists</font>绘制几何图形。

删除没有绘制内容的通道。

将<font color=#66ff66> geometry rendering</font>拆分为多个通道。

废弃<font color="red">dynamic batching and GPU instancing</font>设置。

<font color=#4db8ff>Link：</font>https://catlikecoding.com/unity/custom-srp/2-1-0/

![img](https://catlikecoding.com/unity/custom-srp/2-1-0/tutorial-image.jpg)

<center>How it all started.</center>

This tutorial is made with Unity 2022.3.10f1 and follows [Custom SRP 2.0.0](https://catlikecoding.com/unity/custom-srp/2-0-0/).

### 1、Unsupported Shaders Pass

虽然我们已经调整了 SRP 以与<font color="red">Render Graph API </font>配合使用，但我们并没有改变渲染方式，因此还无法从<font color=#FFCE70>Render Graph</font>的功能中获益。在本教程中，我们将在绘制<font color=#66ff66>visible geometry</font>时改用<font color=#66ff66>renderer lists</font>。

我们首先调整 <font color=#66ff66>UnsupportedShadersPass</font>，因为它很简单。

#### 1.1 Moving Code

将<font color=#66ff66>shader tag ID </font> 数组和错误材质从 <font color=#66ff66>CameraRenderer</font> 复制到 <font color=#66ff66>UnsupportedShadersPass</font>，并在 **Record** 中初始化材质。同时删除 <font color=#66ff66>CameraRenderer </font>字段，不再调用其 <font color=#66ff66>DrawUnsupportedShaders </font>方法。暂时保留 **Record** 的渲染器参数。

```C#
static readonly ShaderTagId[] shaderTagIds = {
    new("Always"),
    new("ForwardBase"),
    new("PrepassBase"),
    new("Vertex"),
    new("VertexLMRGBM"),
    new("VertexLM")
};

static Material errorMaterial;

//CameraRenderer renderer;

void Render(RenderGraphContext context) { } //=> renderer.DrawUnsupportedShaders();

[Conditional("UNITY_EDITOR")]
public static void Record(RenderGraph renderGraph, CameraRenderer renderer)
{
    …

        if (errorMaterial == null)
        {
            errorMaterial = new(Shader.Find("Hidden/InternalErrorShader"));
        }
    //pass.renderer = renderer;

    …
}
```

现在可以从 中删除复制的代码<font color=#66ff66>CameraRenderer.editor</font>。

```c++
//static ShaderTagId[] legacyShaderTagIds = { … };

//static Material errorMaterial;

//public void DrawUnsupportedShaders() { … }
```

#### 1.2 Rendering via a List

要使用<font color=#bc8df9>renderer lists</font>，<font color=#66ff66>UnsupportedShadersPass</font>需要访问<font color=#66ff66>UnityEngine和UnityEngine.Rendering.RendererUtils</font>命名空间中的类型。

```c++
using System.Diagnostics;
using UnityEngine;
using UnityEngine.Experimental.Rendering.RenderGraphModule;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RendererUtils;
```

现在需要<font color=#4db8ff> camera and culling results</font>，而不是<font color=#66ff66>CameraRenderer</font>参数。**Record**它使用这些来创建一个<font color=#66ff66>RendererListDesc</font>结构体，并向其传递<font color=#4db8ff>shader tag ID array</font>、<font color=#4db8ff>the culling results, and the camera</font>。

```c++
public static void Record(
    RenderGraph renderGraph, Camera camera, CullingResults cullingResults)
{
    #if UNITY_EDITOR
    using RenderGraphBuilder builder = renderGraph.AddRenderPass(
        sampler.name, out UnsupportedShadersPass pass, sampler);

    if (errorMaterial == null)
    {
        errorMaterial = new(Shader.Find("Hidden/InternalErrorShader"));
    }

    new RendererListDesc(shaderTagIds, cullingResults, camera);

    builder.SetRenderFunc<UnsupportedShadersPass>(
        (pass, context) => pass.Render(context));
    #endif
}
```

Record调整传递参数给<font color=#66ff66>CameraRenderer.Render</font>以匹配。

```c++
UnsupportedShadersPass.Record(renderGraph, camera, cullingResults);
```

<font color=#66ff66>renderer list description</font>取代了我们之前用于<font color="red"> drawing, filtering, and sorting settings</font>（绘制的绘制、过滤和排序设置）。现在我们只需创建一个描述。我们必须设置其<font color=#66ff66> render queue range</font>，对于此通道来说，它包括所有内容。我们还必须使用错误材质作为覆盖材质，以使用不受支持的着色器绘制几何体。

```c++
new RendererListDesc(shaderTagIds, cullingResults, camera)
{
    overrideMaterial = errorMaterial,
    renderQueueRange = RenderQueueRange.all
};
```

要创建<font color=#66ff66> renderer list</font>，请<font color=#FFCE70>CreateRendererList</font>在<font color="red"> render graph</font>上调用，并将**description** 传递给它。

```C#
renderGraph.CreateRendererList(
    new RendererListDesc(shaderTagIds, cullingResults, camera)
    {
        overrideMaterial = errorMaterial,
        renderQueueRange = RenderQueueRange.all
    });
```

这并不直接为我们提供一个引用的<font color=#66ff66>RendererList</font>，而是 一个<font color=#66ff66>RendererListHandle</font>，它是由<font color=#bc8df9> renderer graph.</font>管理的列表资源的**handle** 。为它创建一个字段，以便我们可以在 **Render**中访问它。

```C#
RendererListHandle list;
```

我们获取的<font color="red">handle </font>在<font color=#66ff66>Record</font>此时无效。我们必须注册该<font color="red">handle </font>，以便<font color="red"> render graph</font>通过构建器的<font color=#66ff66>UseRendererList</font>方法传递它来知道我们的通道使用它。

这给了我们相同的<font color="red">handle </font>，我们将其分配给我们的字段。<font color="red"> render graph</font>将确保该列表在调用我们通道的渲染函数时存在且有效。

```c++
pass.list = builder.UseRendererList(renderGraph.CreateRendererList(
    new RendererListDesc(shaderTagIds, cullingResults, camera)
    {
        overrideMaterial = errorMaterial,
        renderQueueRange = RenderQueueRange.all
    }));
```

