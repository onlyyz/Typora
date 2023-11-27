[RenderObjects](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.unity3d.com%2FPackages%2Fcom.unity.render-pipelines.universal@15.0%2Fmanual%2Fcontainers%2Fhow-to-custom-effect-render-objects.html) 是 ScriptableRendererFeature 的一种实现，它通过 RenderObjectsPass 渲染指定图层的对象。在 Assets 窗口选中 Universal Renderer Data 文件，在 Inspector 窗口依次点击【Add Renderer Feature→Render Objects】，如下。

```c++
public enum RenderPassEvent {
    BeforeRendering = 0, // 渲染前
    BeforeRenderingShadows = 50, // 渲染阴影前
    AfterRenderingShadows = 100, // 渲染阴影后
    BeforeRenderingPrePasses = 150, // 渲染PrePasses前
    [EditorBrowsable(EditorBrowsableState.Never)]
    [Obsolete("Obsolete, to match the capital from 'Prepass' to 'PrePass' (UnityUpgradable) -> BeforeRenderingPrePasses")]
    BeforeRenderingPrepasses = 151, // 渲染PrePasses前
    AfterRenderingPrePasses = 200, // 渲染PrePasses后
    BeforeRenderingGbuffer = 210, // 渲染Gbuffer前
    AfterRenderingGbuffer = 220, // 渲染Gbuffer后
    BeforeRenderingDeferredLights = 230, // 渲染延时光照前
    AfterRenderingDeferredLights = 240, // 渲染延时光照后
    BeforeRenderingOpaques = 250, // 渲染不透明物体前
    AfterRenderingOpaques = 300, // 渲染不透明物体后
    BeforeRenderingSkybox = 350, // 渲染天空盒子前
    AfterRenderingSkybox = 400, // 渲染天空盒子后
    BeforeRenderingTransparents = 450, // 渲染透明物体前
    AfterRenderingTransparents = 500, // 渲染透明物体后
    BeforeRenderingPostProcessing = 550, // 屏幕后处理前
    AfterRenderingPostProcessing = 600, // 屏幕后处理后
    AfterRendering = 1000 // 渲染前
}
```

