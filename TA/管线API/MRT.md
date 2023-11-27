<font color=#4db8ff>Link：</font>https://zenn.dev/r_ngtm/articles/unity-mrt-urp10-urp14

<font color=#4db8ff>git：</font>https://github.com/rngtm/Unity_MRT_Sample

#### 一、MRT

##### 1.1 なぃMRT

通过使用MRT，可以将对象同时绘制到多个渲染目标

![img](https://res.cloudinary.com/zenn/image/fetch/s--QTgSV5Zi--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_1200/https://storage.googleapis.com/zenn-user-upload/deployed-images/bc2671e87158f9ad4434cd0e.png%3Fsha%3De90d6b4c2e6ba3d4b7af69884ff81778468a6687)

在 MRT 的着色器中，<font color=#4db8ff> Strut </font>在 frag 中返回。

```c++
struct BufferOutput
{
    half4 color : COLOR0; // RT0
    half3 normal : COLOR1; // RT1
};

BufferOutput frag (v2f i) : SV_Target
{
    BufferOutput o = (BufferOutput)0;
    o.color = tex2D(_MainTex, i.uv);
    o.normal = i.normal;
    return o;
}
```

shader

```c++
Shader "Unlit/NewUnlitShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
                float3 normal : TEXCOORD2;
            };
            
            struct BufferOutput
            {
                half4 color;
                half3 normal; 
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                o.normal = UnityObjectToWorldNormal(v.normal);
                return o;
            }

            BufferOutput frag (v2f i) : SV_Target
            {
                BufferOutput o = (BufferOutput)0;
                o.color = tex2D(_MainTex, i.uv);
                o.normal = i.normal;
                return o;
            }
            ENDCG
        }
    }
}
```

##### 1.2  version 

**URP12**

**在 URP12 及更早版本（Unity 2021.3 系列）中， CommandBuffer.GetTemporaryRT**用于创建临时渲染目标

```c++
var desc = renderingData.cameraData.cameraTargetDescriptor;
var colorTexture =  new RenderTargetIdentifier("_CustomPassHandle");
commandBuffer.GetTemporaryRT(colorTexture, desc);
```

**URP14**

RP14添加了<font color=#4db8ff>RTHandles</font>，这是一种管理渲染目标的机制。
使用RenderingUtils.ReAllocateIfNeeded或RTHandles.Alloc 创建 RTHandles 。

如果使用RenderingUtils.ReAllocateIfNeeded，如果 RTHandle 已分配纹理，纹理将自动释放。

```c++
var desc = renderingData.cameraData.cameraTargetDescriptor;
// Then using RTHandles, the color and the depth properties must be separate
desc.depthBufferBits = 0;
RenderingUtils.ReAllocateIfNeeded(ref m_Handle, desc, FilterMode.Point,
                                    TextureWrapMode.Clamp, name: "_CustomPassHandle");
```



```c++
var desc = renderingData.cameraData.cameraTargetDescriptor;
desc.depthBufferBits = 0;
m_Handle = RTHandles.Alloc(desc, name: "_CustomPassHandle");
```

当不再需要纹理时，可以通过调用**RTHandles.Release来释放它。**

参考：https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/upgrade-guide-2022-2.html

#### 二、MRTを実装してみる

##### 2.1 URP10.7.0 での MRT

创建一个 RendererFeature，将对象渲染为 Color、Normal 和 Depth，参考 URP10.7.0 的 RenderObjects.cs 实现。

```c++
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace UnityEngine.Experimental.Rendering.Universal
{
    static class ShaderPropertyId
    {
        public static readonly int MyColorTexture = Shader.PropertyToID("_MyColorTexture");
        public static readonly int MyDepthTexture = Shader.PropertyToID("_MyDepthTexture");
        public static readonly int MyNormalTexture = Shader.PropertyToID("_MyNormalTexture");
        public static readonly int ColorTex = Shader.PropertyToID("_ColorTex");
        public static readonly int NormalTex = Shader.PropertyToID("_NormalTex");
    }

    public static class MyRenderTargetBuffer
    {
        private static bool isInitialize = false;
        
        public static RenderTargetIdentifier[] ColorAttachments;
        public static RenderTargetIdentifier MyDepthTexture;
        public static RenderTargetIdentifier MyColorTexture;
        public static RenderTargetIdentifier MyNormalTexture;
  
        public static void Initialize()
        {
            if (isInitialize)
            {
                return;
            }

            isInitialize = true;

            MyColorTexture = new RenderTargetIdentifier(ShaderPropertyId.MyColorTexture);
            MyNormalTexture = new RenderTargetIdentifier(ShaderPropertyId.MyNormalTexture);
            MyDepthTexture = new RenderTargetIdentifier(ShaderPropertyId.MyDepthTexture);

            ColorAttachments = new RenderTargetIdentifier[]
            {
                MyColorTexture,
                MyNormalTexture,
            };

        }

        public static void Dispose()
        {
            ColorAttachments = null;
            isInitialize = false;
        }
    }
    
    public class SetupRenderPass : ScriptableRenderPass
    {
        public SetupRenderPass(RenderPassEvent renderPassEvent)
        {
            this.renderPassEvent = renderPassEvent;
        }

        public override void Configure(CommandBuffer cmd, RenderTextureDescriptor desc)
        {
            base.Configure(cmd, desc);

            var colorDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
            var depthDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.Depth, 8);
            var normalDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
            cmd.GetTemporaryRT(ShaderPropertyId.MyColorTexture, colorDesc);
            cmd.GetTemporaryRT(ShaderPropertyId.MyDepthTexture, depthDesc);
            cmd.GetTemporaryRT(ShaderPropertyId.MyNormalTexture, normalDesc);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
        }

        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            base.OnCameraCleanup(cmd);
            
            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyColorTexture);
            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyDepthTexture);
            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyNormalTexture);
        }
    }

    public class MyDrawObjectsFeature : ScriptableRendererFeature
    {
        public RenderObjects.RenderObjectsSettings settings = new RenderObjects.RenderObjectsSettings();
        private RenderObjectsPass renderObjectsPass;
        private SetupRenderPass setupRenderPass;
        
        public override void Create()
        {
            MyRenderTargetBuffer.Initialize();
            RenderObjects.FilterSettings filter = settings.filterSettings;

            // Render Objects pass doesn't support events before rendering prepasses.
            // The camera is not setup before this point and all rendering is monoscopic.
            // Events before BeforeRenderingPrepasses should be used for input texture passes (shadow map, LUT, etc) that doesn't depend on the camera.
            // These events are filtering in the UI, but we still should prevent users from changing it from code or
            // by changing the serialized data.
            if (settings.Event < RenderPassEvent.BeforeRenderingPrepasses)
                settings.Event = RenderPassEvent.BeforeRenderingPrepasses;

            renderObjectsPass = new RenderObjectsPass(settings.passTag, settings.Event, filter.PassNames,
                filter.RenderQueueType, filter.LayerMask, settings.cameraSettings);

            setupRenderPass = new SetupRenderPass(RenderPassEvent.BeforeRenderingPrepasses);

            renderObjectsPass.overrideMaterial = settings.overrideMaterial;
            renderObjectsPass.overrideMaterialPassIndex = settings.overrideMaterialPassIndex;

            if (settings.overrideDepthState)
                renderObjectsPass.SetDetphState(settings.enableWrite, settings.depthCompareFunction);

            if (settings.stencilSettings.overrideStencilState)
                renderObjectsPass.SetStencilState(settings.stencilSettings.stencilReference,
                    settings.stencilSettings.stencilCompareFunction, settings.stencilSettings.passOperation,
                    settings.stencilSettings.failOperation, settings.stencilSettings.zFailOperation);
        }
        
        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            renderObjectsPass.ConfigureTarget(MyRenderTargetBuffer.ColorAttachments, MyRenderTargetBuffer.MyDepthTexture);
            renderObjectsPass.ConfigureClear(ClearFlag.All, Color.clear);

            renderer.EnqueuePass(setupRenderPass);
            renderer.EnqueuePass(renderObjectsPass);
        }

        protected override void Dispose(bool disposing)
        {
            base.Dispose(disposing);
            MyRenderTargetBuffer.Dispose();
        }
    }
}
```

![img](https://res.cloudinary.com/zenn/image/fetch/s--231Km5OZ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_1200/https://storage.googleapis.com/zenn-user-upload/deployed-images/e203365c5e0e85ca3cdbe0fe.png%3Fsha%3Db8f99920a2df55f0f6ef7c0e21433e4f44123121)

##### 2.2 带有 URP12.1.8 的 MRT

创建一个 RendererFeature，将对象渲染为 Color、Normal 和 Depth，参考 URP12.1.8 的 .cs 实现。

```c++
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace UnityEngine.Experimental.Rendering.Universal
{
    static class ShaderPropertyId
    {
        public static readonly int MyColorTexture = Shader.PropertyToID("_MyColorTexture");
        public static readonly int MyDepthTexture = Shader.PropertyToID("_MyDepthTexture");
        public static readonly int MyNormalTexture = Shader.PropertyToID("_MyNormalTexture");
        public static readonly int ColorTex = Shader.PropertyToID("_ColorTex");
        public static readonly int NormalTex = Shader.PropertyToID("_NormalTex");
    }

    public static class MyRenderTargetBuffer
    {
        private static bool isInitialize = false;

        public static RenderTargetIdentifier[] ColorAttachments;
        public static RenderTargetIdentifier MyDepthTexture;
        public static RenderTargetIdentifier MyColorTexture;
        public static RenderTargetIdentifier MyNormalTexture;

        public static void Initialize()
        {
            if (isInitialize)
            {
                return;
            }

            isInitialize = true;

            MyColorTexture = new RenderTargetIdentifier(ShaderPropertyId.MyColorTexture);
            MyNormalTexture = new RenderTargetIdentifier(ShaderPropertyId.MyNormalTexture);
            MyDepthTexture = new RenderTargetIdentifier(ShaderPropertyId.MyDepthTexture);

            ColorAttachments = new RenderTargetIdentifier[]
            {
                MyColorTexture,
                MyNormalTexture,
            };

        }

        public static void Dispose()
        {
            ColorAttachments = null;
            isInitialize = false;
        }
    }

    public class SetupRenderPass : ScriptableRenderPass
    {
        public SetupRenderPass(RenderPassEvent renderPassEvent)
        {
            this.renderPassEvent = renderPassEvent;
        }

        public override void Configure(CommandBuffer cmd, RenderTextureDescriptor desc)
        {
            base.Configure(cmd, desc);

            var colorDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
            var depthDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.Depth, 8);
            var normalDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
            cmd.GetTemporaryRT(ShaderPropertyId.MyColorTexture, colorDesc);
            cmd.GetTemporaryRT(ShaderPropertyId.MyDepthTexture, depthDesc);
            cmd.GetTemporaryRT(ShaderPropertyId.MyNormalTexture, normalDesc);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
        }

        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            base.OnCameraCleanup(cmd);

            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyColorTexture);
            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyDepthTexture);
            cmd.ReleaseTemporaryRT(ShaderPropertyId.MyNormalTexture);
        }
    }

    public class MyDrawObjectsFeature : ScriptableRendererFeature
    {
        public RenderObjects.RenderObjectsSettings settings = new RenderObjects.RenderObjectsSettings();
        private RenderObjectsPass renderObjectsPass;
        private SetupRenderPass setupRenderPass;

        public override void Create()
        {
            MyRenderTargetBuffer.Initialize();
            RenderObjects.FilterSettings filter = settings.filterSettings;

            // Render Objects pass doesn't support events before rendering prepasses.
            // The camera is not setup before this point and all rendering is monoscopic.
            // Events before BeforeRenderingPrepasses should be used for input texture passes (shadow map, LUT, etc) that doesn't depend on the camera.
            // These events are filtering in the UI, but we still should prevent users from changing it from code or
            // by changing the serialized data.
            if (settings.Event < RenderPassEvent.BeforeRenderingPrePasses)
                settings.Event = RenderPassEvent.BeforeRenderingPrePasses;

            renderObjectsPass = new RenderObjectsPass(settings.passTag, settings.Event, filter.PassNames,
                                                      filter.RenderQueueType, filter.LayerMask, settings.cameraSettings);

            setupRenderPass = new SetupRenderPass(RenderPassEvent.BeforeRenderingPrePasses);

            renderObjectsPass.overrideMaterial = settings.overrideMaterial;
            renderObjectsPass.overrideMaterialPassIndex = settings.overrideMaterialPassIndex;

            if (settings.overrideDepthState)
                renderObjectsPass.SetDetphState(settings.enableWrite, settings.depthCompareFunction);

            if (settings.stencilSettings.overrideStencilState)
                renderObjectsPass.SetStencilState(settings.stencilSettings.stencilReference,
                                                  settings.stencilSettings.stencilCompareFunction, settings.stencilSettings.passOperation,
                                                  settings.stencilSettings.failOperation, settings.stencilSettings.zFailOperation);
        }

        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            renderObjectsPass.ConfigureTarget(MyRenderTargetBuffer.ColorAttachments, MyRenderTargetBuffer.MyDepthTexture);
            renderObjectsPass.ConfigureClear(ClearFlag.All, Color.clear);

            renderer.EnqueuePass(setupRenderPass);
            renderer.EnqueuePass(renderObjectsPass);
        }

        protected override void Dispose(bool disposing)
        {
            base.Dispose(disposing);
            MyRenderTargetBuffer.Dispose();
        }
    }
}
```

与URP10.7.0的差异

RenderPassEvent.BeforeRenderingPrepasses 已更改为 BeforeRenderingPrePasses

```c++
- if (settings.Event < RenderPassEvent.BeforeRenderingPrepasses)
+ if (settings.Event < RenderPassEvent.BeforeRenderingPrePasses)
-     settings.Event = RenderPassEvent.BeforeRenderingPrepasses;
+     settings.Event = RenderPassEvent.BeforeRenderingPrePasses;

...

- setupRenderPass = new SetupRenderPass(RenderPassEvent.BeforeRenderingPrepasses);
+ setupRenderPass = new SetupRenderPass(RenderPassEvent.BeforeRenderingPrePasses);
```

<font color=#4db8ff>RenderObjectsPass：</font>https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/api/UnityEngine.Experimental.Rendering.Universal.RenderObjectsPass.-ctor.html?q=RenderObjectsPass



##### 2.3 MRT 与 URP14.0.7

参考 URP14 的 RenderObjects.cs 实现，
创建一个 RendererFeature，将对象渲染为 Color、Normal 和 Depth 。

```c++
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

namespace UnityEngine.Experimental.Rendering.Universal
{
    public static class ShaderPropertyId
    {
        public static readonly int ColorTex = Shader.PropertyToID("_ColorTex");
        public static readonly int NormalTex = Shader.PropertyToID("_NormalTex");
    }
    
    public class MyRenderTargetBuffer
    {
        private static bool isInitialize = false;
        
        static RTHandle m_MyColorTexture;
        static RTHandle m_MyDepthTexture;
        static RTHandle m_MyNormalTexture;

        public static RTHandle MyColorTexture => m_MyColorTexture;
        public static RTHandle MyDepthTexture => m_MyDepthTexture;
        public static RTHandle MyNormalTexture => m_MyNormalTexture;
        
        public static RTHandle[] ColorAttachments { get; private set; }
        public static RTHandle DepthAttachment { get; private set; }

        public static void Setup(RenderTextureDescriptor desc)
        {
            if (isInitialize)
            {
                return;
            }
            isInitialize = true;
            
            Debug.Log("Alloc RTHandle");
            
            var colorDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
            var depthDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.Depth, 8);
            var normalDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);

            RenderingUtils.ReAllocateIfNeeded(ref m_MyColorTexture, colorDesc, FilterMode.Point, TextureWrapMode.Clamp,
                name: "_MyColorTexture");
            RenderingUtils.ReAllocateIfNeeded(ref m_MyDepthTexture, depthDesc, FilterMode.Point, TextureWrapMode.Clamp,
                name: "_MyDepthTexture");
            RenderingUtils.ReAllocateIfNeeded(ref m_MyNormalTexture, normalDesc, FilterMode.Point, TextureWrapMode.Clamp,
                name: "_MyNormalTexture");
            
            ColorAttachments = new[] { MyColorTexture, MyNormalTexture };
            DepthAttachment = MyDepthTexture;
        }

        public static void Dispose()
        {
            if (!isInitialize)
            {
                return;
            }
            Debug.Log("Release RTHandle");
            
            DepthAttachment = null;
            ColorAttachments = null;
            
            RTHandles.Release(m_MyNormalTexture);
            m_MyNormalTexture = null;

            RTHandles.Release(MyDepthTexture);
            m_MyDepthTexture = null;

            RTHandles.Release(MyColorTexture);
            m_MyColorTexture = null;

            isInitialize = false;
        }
    }
    
    public class MyRenderObjectsFeature : ScriptableRendererFeature
    {
        public RenderObjects.RenderObjectsSettings settings = new RenderObjects.RenderObjectsSettings();

        RenderObjectsPass renderObjectsPass;

        /// <inheritdoc/>
        public override void Create()
        {
            RenderObjects.FilterSettings filter = settings.filterSettings;

            if (settings.Event < RenderPassEvent.BeforeRenderingPrePasses)
                settings.Event = RenderPassEvent.BeforeRenderingPrePasses;

            renderObjectsPass = new RenderObjectsPass(settings.passTag, settings.Event, filter.PassNames,
                filter.RenderQueueType, filter.LayerMask, settings.cameraSettings);

            switch (settings.overrideMode)
            {
                case RenderObjects.RenderObjectsSettings.OverrideMaterialMode.None:
                    renderObjectsPass.overrideMaterial = null;
                    renderObjectsPass.overrideShader = null;
                    break;
                case RenderObjects.RenderObjectsSettings.OverrideMaterialMode.Material:
                    renderObjectsPass.overrideMaterial = settings.overrideMaterial;
                    renderObjectsPass.overrideMaterialPassIndex = settings.overrideMaterialPassIndex;
                    renderObjectsPass.overrideShader = null;
                    break;
                case RenderObjects.RenderObjectsSettings.OverrideMaterialMode.Shader:
                    renderObjectsPass.overrideMaterial = null;
                    renderObjectsPass.overrideShader = settings.overrideShader;
                    renderObjectsPass.overrideShaderPassIndex = settings.overrideShaderPassIndex;
                    break;
            }

            if (settings.overrideDepthState)
                renderObjectsPass.SetDetphState(settings.enableWrite, settings.depthCompareFunction);

            if (settings.stencilSettings.overrideStencilState)
                renderObjectsPass.SetStencilState(settings.stencilSettings.stencilReference,
                    settings.stencilSettings.stencilCompareFunction, settings.stencilSettings.passOperation,
                    settings.stencilSettings.failOperation, settings.stencilSettings.zFailOperation);
        }

        /// <inheritdoc/>
        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            MyRenderTargetBuffer.Setup(renderingData.cameraData.cameraTargetDescriptor);

            renderObjectsPass.ConfigureTarget(MyRenderTargetBuffer.ColorAttachments, MyRenderTargetBuffer.DepthAttachment);
            renderObjectsPass.ConfigureClear(ClearFlag.All, Color.clear);

            renderer.EnqueuePass(renderObjectsPass);
        }

        protected override void Dispose(bool disposing)
        {
            base.Dispose(disposing);
            
            MyRenderTargetBuffer.Dispose();
        }
    }
}
```

![img](https://res.cloudinary.com/zenn/image/fetch/s--os1GLlaO--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_1200/https://storage.googleapis.com/zenn-user-upload/deployed-images/1ddfddfa5fa7efee5a038a1b.png%3Fsha%3Da759dafbfdc6733da52e33f4df35ef5daaa099f1)

##### 2.4 差异

URP14与URP12.1.8的差异

###### レンダーターゲット指定方法の違い

在URP12之前，使用RenderTargetIdentifier，但 在URP14中，使用RTHandle 。



###### テクスチャ作成方法の違い

在 URP12 及更早版本中，纹理是在 SciriptableRenderPass 中创建的

```c++
// URP10.7.0, URP12.1.8
public class SetupRenderPass : ScriptableRenderPass
{
    ....
    public override void Configure(CommandBuffer cmd, RenderTextureDescriptor desc)
    {
        base.Configure(cmd, desc);

        var colorDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
        var depthDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.Depth, 8);
        var normalDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
        cmd.GetTemporaryRT(ShaderPropertyId.MyColorTexture, colorDesc);
        cmd.GetTemporaryRT(ShaderPropertyId.MyDepthTexture, depthDesc);
        cmd.GetTemporaryRT(ShaderPropertyId.MyNormalTexture, normalDesc);
```

在 URP14 中，纹理是在 SciriptableRendererFeature 中创建的

```c++
// URP14
public static class MyRenderTargetBuffer
{
    static RTHandle m_MyColorTexture;
    static RTHandle m_MyDepthTexture;
    static RTHandle m_MyNormalTexture;
    ....
    public static void Setup(RenderTextureDescriptor desc)
    {
        ....
        var colorDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);
        var depthDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.Depth, 8);
        var normalDesc = new RenderTextureDescriptor(desc.width, desc.height, RenderTextureFormat.ARGB32, 0);

        // URP14
        RenderingUtils.ReAllocateIfNeeded(ref m_MyColorTexture, colorDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_MyColorTexture");
        RenderingUtils.ReAllocateIfNeeded(ref m_MyDepthTexture, depthDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_MyDepthTexture");
        RenderingUtils.ReAllocateIfNeeded(ref m_MyNormalTexture, normalDesc, FilterMode.Point, TextureWrapMode.Clamp, name: "_MyNormalTexture");

```

##### 2.5 关于RTHandle

RTHandle 定义了 RenderTexture 和 RenderTargetIdentifier。RTHandle 抽象了渲染目标 ID 和渲染纹理实体，并且
可以将其视为单个实体。

```c++
public class RTHandle
{
    internal RTHandleSystem m_Owner;
    internal RenderTexture m_RT;
    internal Texture m_ExternalTexture;
    internal RenderTargetIdentifier m_NameID;
    ...
}
```

RTHandle 定义了隐式类型转换

```c++
public static implicit operator RenderTargetIdentifier(RTHandle handle) { ... }
public static implicit operator Texture(RTHandle handle) { ... }
```

由于隐式类型转换，现在可以在 CommandBuffer 的 SetGlobalTexture 中传递 RTHandle 而不是 RenderTargetIdentifier，从而更易于处理。

```c++
public void SetGlobalTexture(int nameID, RenderTargetIdentifier value) 
```

#### 三 、Deferred Lighting を実装してみる

这次，我将 _MyColorTexture 和 _MyNormalTexture 导出为纹理。让我们使用这些纹理来实现照明。

![img](https://res.cloudinary.com/zenn/image/fetch/s--Kb9ZxGHx--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_1200/https://storage.googleapis.com/zenn-user-upload/deployed-images/376ac16a30ba1953a197eaa3.png%3Fsha%3D9c6616eb12c2f3bbe06a936f1a38f8bd134f502b)



##### 3.1 URP 10.7.0

URP10.7.0中的RendererFeature如下。

```c++
using UnityEngine;
using UnityEngine.Experimental.Rendering.Universal;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MyDeferredLightingFeature : ScriptableRendererFeature
{
    class CustomRenderPass : ScriptableRenderPass
    {
        private Material _material;

        public void Setup(Material material)
        {
            _material = material;
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("My Deferred Lighting");
            var dest = renderingData.cameraData.renderer.cameraColorTarget;
            
            // シェーダーの _ColorTex へ、テクスチャ _MyColorTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.ColorTex, MyRenderTargetBuffer.MyColorTexture);
            
            // シェーダーの _NormalTex へ、テクスチャ _MyNormalTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.NormalTex, MyRenderTargetBuffer.MyNormalTexture);
                
            cmd.Blit(null, dest, _material);
            context.ExecuteCommandBuffer(cmd);
            
            CommandBufferPool.Release(cmd);
        }
    }

    Material _material;
    CustomRenderPass m_ScriptablePass;


    /// <inheritdoc/>
    public override void Create()
    {
        m_ScriptablePass = new CustomRenderPass();

        // Configures where the render pass should be injected.
        m_ScriptablePass.renderPassEvent = RenderPassEvent.AfterRenderingTransparents;

        if (_material == null)
        {
            _material = CoreUtils.CreateEngineMaterial("Hidden/MyDeferredLighting");
        }
    }

    // Here you can inject one or multiple render passes in the renderer.
    // This method is called when setting up the renderer once per-camera.
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        m_ScriptablePass.Setup(_material);
        renderer.EnqueuePass(m_ScriptablePass);
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
        if (_material != null)
        {
            CoreUtils.Destroy(_material);
            _material = null;
        }
    }
}

```





##### 3.2 URP12.1.8

URP12.1.8中的RendererFeature如下。

```c++
using UnityEngine;
using UnityEngine.Experimental.Rendering.Universal;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MyDeferredLightingFeature : ScriptableRendererFeature
{
    class CustomRenderPass : ScriptableRenderPass
    {
        private Material _material;

        public void Setup(Material material)
        {
            _material = material;
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("My Deferred Lighting");
            var dest = renderingData.cameraData.renderer.cameraColorTarget;
            
            // シェーダーの _ColorTex へ、テクスチャ _MyColorTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.ColorTex, MyRenderTargetBuffer.MyColorTexture);
            
            // シェーダーの _NormalTex へ、テクスチャ _MyNormalTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.NormalTex, MyRenderTargetBuffer.MyNormalTexture);
                
            cmd.Blit(null, dest, _material);
            context.ExecuteCommandBuffer(cmd);
            
            CommandBufferPool.Release(cmd);
        }
    }

    Material _material;
    CustomRenderPass m_ScriptablePass;


    /// <inheritdoc/>
    public override void Create()
    {
        m_ScriptablePass = new CustomRenderPass();

        // Configures where the render pass should be injected.
        m_ScriptablePass.renderPassEvent = RenderPassEvent.AfterRenderingTransparents;

        if (_material == null)
        {
            _material = CoreUtils.CreateEngineMaterial("Hidden/MyDeferredLighting");
        }
    }

    // Here you can inject one or multiple render passes in the renderer.
    // This method is called when setting up the renderer once per-camera.
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        m_ScriptablePass.Setup(_material);
        renderer.EnqueuePass(m_ScriptablePass);
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
        if (_material != null)
        {
            CoreUtils.Destroy(_material);
            _material = null;
        }
    }
}

```

##### 3.3 URP14.0.7

URP14.0.7中的RendererFeature如下。

```c++
using UnityEngine;
using UnityEngine.Experimental.Rendering.Universal;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class MyDeferredLightingFeature : ScriptableRendererFeature
{
    class CustomRenderPass : ScriptableRenderPass
    {
        private Material _material;

        public void Setup(Material material)
        {
            _material = material;
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            var cmd = CommandBufferPool.Get("My Deferred Lighting");
            var cameraColorTargetHandle = renderingData.cameraData.renderer.cameraColorTargetHandle;
            
            // シェーダーの _ColorTex へ、テクスチャ _MyColorTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.ColorTex, MyRenderTargetBuffer.MyColorTexture);
            
            // シェーダーの _NormalTex へ、テクスチャ _MyNormalTexture を設定する
            cmd.SetGlobalTexture(ShaderPropertyId.NormalTex, MyRenderTargetBuffer.MyNormalTexture);
                
            cmd.Blit(null, cameraColorTargetHandle, _material);
            context.ExecuteCommandBuffer(cmd);
            
            CommandBufferPool.Release(cmd);
        }
    }

    Material _material;
    CustomRenderPass m_ScriptablePass;


    /// <inheritdoc/>
    public override void Create()
    {
        m_ScriptablePass = new CustomRenderPass();

        // Configures where the render pass should be injected.
        m_ScriptablePass.renderPassEvent = RenderPassEvent.AfterRenderingTransparents;

        if (_material == null)
        {
            _material = CoreUtils.CreateEngineMaterial("Hidden/MyDeferredLighting");
        }
    }

    // Here you can inject one or multiple render passes in the renderer.
    // This method is called when setting up the renderer once per-camera.
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        m_ScriptablePass.Setup(_material);
        renderer.EnqueuePass(m_ScriptablePass);
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
        if (_material != null)
        {
            CoreUtils.Destroy(_material);
            _material = null;
        }
    }
}
```

#### 四、差分

##### 差分1: レンダーターゲットの作成

**URP12 之前**

**使用CommandBuffer.GetTemporaryRT()**创建渲染目标，并使用
**CommandBuffer.ReleaseTemporaryRT()**释放渲染目标。

```c++
public void GetTemporaryRT(int nameID, RenderTextureDescriptor desc) {...}
public extern void ReleaseTemporaryRT(int nameID);
```

**URP14**

**使用RTHandles.Alloc()**创建渲染纹理（渲染目标） ，并使用
**RTHandles.Release()**释放渲染纹理（渲染目标） 。

```c++
public static RTHandle Alloc(RenderTargetIdentifier tex, string name) { ... }
public static void Release(RTHandle rth) { ... }
```

提供了**RenderingUtils.ReAllocateIfNeeded**，它包装了**RTHandles.Alloc以供安全使用。**

```c++
public static bool ReAllocateIfNeeded(
    ref RTHandle handle,
    in RenderTextureDescriptor descriptor,
    FilterMode filterMode = FilterMode.Point,
    TextureWrapMode wrapMode = TextureWrapMode.Repeat,
    bool isShadowMap = false,
    int anisoLevel = 1,
    float mipMapBias = 0,
    string name = "")
{
    if (RTHandleNeedsReAlloc(handle, descriptor, filterMode, wrapMode, isShadowMap, anisoLevel, mipMapBias, name, false))
    {
        handle?.Release();
        handle = RTHandles.Alloc(descriptor, filterMode, wrapMode, isShadowMap, anisoLevel, mipMapBias, name);
        return true;
    }
    return false;
}
```

##### 差分2: レンダーターゲットの指定

**URP12 之前**

使用ConfigureTarget 指定渲染目标。

```c++
public void ConfigureTarget(RenderTargetIdentifier[] colorAttachments, RenderTargetIdentifier depthAttachment) { ... }
```

**URP14**

URP14 使用 RTHandle。

```c++
public void ConfigureTarget(RTHandle[] colorAttachments, RTHandle depthAttachment)
```



#### 九、API

##### RenderObjectsPass

```c++
if (shaderTags != null && shaderTags.Length > 0)
{
    foreach (var passName in shaderTags)
        m_ShaderTagIdList.Add(new ShaderTagId(passName));
}
else
{
    m_ShaderTagIdList.Add(new ShaderTagId("SRPDefaultUnlit"));
    m_ShaderTagIdList.Add(new ShaderTagId("UniversalForward"));
    m_ShaderTagIdList.Add(new ShaderTagId("UniversalForwardOnly"));
}
```

如果没自己定义shaderTags，就按照else里面初始定的3个名字，也就是官方初始的

##### DrawObjectsPass.cs

```c++
 public DrawObjectsPass(string profilerTag, bool opaque, RenderPassEvent evt, RenderQueueRange renderQueueRange, LayerMask layerMask, StencilState stencilState, int stencilReference)
            : this(profilerTag,
            new ShaderTagId[] { new ShaderTagId("SRPDefaultUnlit"), new ShaderTagId("UniversalForward"), new ShaderTagId("UniversalForwardOnly") },
            opaque, evt, renderQueueRange, layerMask, stencilState, stencilReference)
        { }
```

觉URP版本之间经常改源码，，比如之前版本SRPDefaultUnlit是在UniversalForward之后的，现在(12.1.7)变成之前了，，也就是说我们的正常着色Pass的LightMode要是SRPDefaultUnlit