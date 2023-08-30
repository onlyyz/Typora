```C#
using System;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

/*
Create  => the Pass.cs
创建材质，以及设置渲染事件，会在Unity OnEnable 和 OnValidate时调用，
创建pass, 传递Shader和渲染事件

OnCameraSetup =>ConfigureTarget
Execute->InternalStartRendering->
 在渲染摄像机之前，渲染器会调用此方法。
 如果需要配置渲染目标及其清除状态，并创建临时渲染目标纹理，则重载此方法。
 如果一个渲染传递没有覆盖此方法，则此渲染传递会渲染到活动的摄像机的渲染目标。
 千万不要调用 CommandBuffer.SetRenderTarget。而应调用 <c>ConfigureTarget</c> 和 <c>ConfigureClear</c>。
在 ScriptableRenderPass API 中添加了 OnCameraSetup() 函数，渲染器在渲染每个摄像头前都会调用该函数

ConfigureTarget
 为本次渲染传递配置渲染目标。调用此方法代替 CommandBuffer.SetRenderTarget 方法。
 此方法应在 Configure.SetRenderTarget 中调用。


AddRenderPasses => EnqueuePass
RenderSingleCamera->->AddRenderPasses->EnqueuePass
 注入创建的渲染Pass 存在多注入，重复注入的Bug

SetupRenderPasses
配置渲染Pass 会在 AddRenderPasses中被调用，可以在这里init参数
// 使用脚本化渲染通行证输入的颜色参数调用配置输入（ConfigureInput）。
            // 确保渲染传递可以使用不透明纹理。

Execute
有点像onRenderImage，似乎每帧渲染，Drew
判断材质是否丢失， 以及Volume 组件
SetupRenderPasses
InternalStartRendering->InternalStartRendering->OnCameraSetup-	>ConfigureTarget

Dispose
GC dispos 和释放

Volume
Inherits from SRP class
Volume只是插值用，可以通过获取component去添加参数







//TODO:
RenderPass 是计算渲染的内容
RenderFeature模块是输入Shader，调用Pass去执行结果。

RenderFeature执行顺序：
主要是有 设置模块(Settings)  初始化模块(Create) 执行逻辑模块(AddRenderPasses) 
Setting: 设置

Create() : render feature初始化的时候调用

AddRenderPasses() : 每帧调用|渲染过程



CustomRenderPass执行顺序：

RenderPassEvent:渲染事件

OnCameraSetUp() : 每帧执行，在里面申请RenderTexture、设置RenderTarget、和ClearRenderTarget。记得用ConfigureRenderTarget()和ConfigureClear()，不要用cmd.SetRenderTarget()这些方法，用urp推荐的方法。
// 在 ScriptableRenderPass API 中添加了 OnCameraSetup() 函数，渲染器在渲染每个摄像头前都会调用该函数。


Execute()： 每帧执行，在里面做DrawMesh或者Blit之类的操作。可以用CommandBufferPool.Get("String")来申请CommandBuffer，之后要记得通过Context.ExecuteCommandBuffer来提交命令。CommandBuffer用完之后记得释放

OnCameraCleanUp():每帧执行，在这里面释放申请的RT。



Volume and Shader

Shader和Volume组件的属性是在Render 渲染里关联起来的，使用SetFloat方法 




获取当前相机的目标RT：

因为主相机是直接渲染到视口，没用绑定RenderTexture，所以用renderingData.CameraData.Camera.RenderTarget获取到的可能是空值

实际上应该用renderingData.CameraData.renderer.cameraColorTarget来获取颜色RT。 



//Todo
OnCameraSetup()
DepthOnlyPass、CopyDepthPass 和 CopyColorPass 现在使用 OnCameraSetup() 而不是 Configure() 在执行前设置其通道，因为它们只需在每个摄像头上获取一次渲染纹理，而不是在每个眼睛上获取一次。
https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/changelog/CHANGELOG.html?q=OnCameraSetup

//分配RTHandle 纹理的分配使用
RenderingUtils.ReAllocateIfNeeded(ref mTempRT0, descriptor, name: mTempRT0Name);

*/
namespace UnityEngine.Experiemntal.Rendering.Universal
{
    /// <summary>
    /// First set Volume parameters 
    /// </summary>
    
    /// <summary>
    /// secondly  Bloom BloomFeature 
    /// </summary>
    
    public class BloomFeature : ScriptableRendererFeature
    {
        // 后处理材质
        Material passMaterial;
        
        [System.Serializable]
        public class BloomSettings
        {
            public RenderPassEvent renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
            public Shader shader;
        }
        public BloomSettings settings = new BloomSettings();
        //RenderPass
        BloomPass bloomPass;
        
        //创建Pass  传递Shader和渲染事件
        public override void Create()
        {
            this.name = "Create Test Bloom";
            passMaterial = CoreUtils.CreateEngineMaterial(settings.shader);
            bloomPass = new BloomPass(settings.renderPassEvent, passMaterial);
        }

        //初始化参数，被AddRenderPasses调用
        public override void SetupRenderPasses(ScriptableRenderer renderer,
            in RenderingData renderingData)
        {
            if (renderingData.cameraData.cameraType == CameraType.Game)
            {
                // 配置此渲染通道的输入要求
                // 确保渲染传递可以使用不透明纹理。
                bloomPass.ConfigureInput(ScriptableRenderPassInput.Color);
                
                //为渲染命令缓冲区（Rendering.CommandBuffer）识别渲染纹理（RenderTexture）。
                bloomPass.SetTarget(renderer.cameraColorTarget);
                
                // 一个 RTHandle 是一个渲染纹理（RenderTexture），它可以根据相机尺寸自动缩放。
                // 这样在渲染过程中使用不同尺寸的相机时，就可以适当地重新利用渲染纹理内存。
                bloomPass.SetTarget(renderer.cameraColorTargetHandle);
            }
        }
        
        //注入Pass  //每帧都会调用，渲染摄像机内容。 
        public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            if (renderingData.cameraData.cameraType == CameraType.Game)
               
                //update input the pass
                renderer.EnqueuePass(bloomPass);
        }
        
        //Dispose
        protected override void Dispose(bool disposing)
        {
            CoreUtils.Destroy(passMaterial);
            bloomPass.Dispose();
        }
    }

    /// <summary>
    /// Third  Bloom Pass 
    /// </summary>

    public class BloomPass : ScriptableRenderPass
    {
        ProfilingSampler m_ProfilingSampler = new ProfilingSampler("Test Bloom");
        // 后处理材质
        Material passMaterial;  
        //Render Target
        RTHandle CameraColorTarget;
        // 用来获取相机rt的id 设置当前渲染目标
        RenderTargetIdentifier currentTarget;    
        // 传递到volume
        BloomVolume _bloomVolune;
        // 设置储存图像信息
        static readonly int TempTargetId = Shader.PropertyToID("_TempleteTex");
        const int MAXITERATION = 16;
        
        
        //RT array of Down Sample and Up Sample
        RTHandle[] mipDownRT;
        RTHandle[] mipUpRT;
        
        //Up and Down
        struct Level 
        {
            internal int down;
            internal int up;
        }
        
        Level[] Dwon_Up_Struct;
        
        
        static class ShaderIDs
        {
            internal static readonly int BlurOffset = Shader.PropertyToID("_BlurOffset");
            internal static readonly int Threshold = Shader.PropertyToID("_Threshold");
            internal static readonly int ThresholdKnee = Shader.PropertyToID("_ThresholdKnee");
            internal static readonly int Intensity = Shader.PropertyToID("_Intensity");
            internal static readonly int BloomColor = Shader.PropertyToID("_BloomColor");
        }
        
        //Create => 接受材质和渲染事件
        public BloomPass(RenderPassEvent evt,Material PassMat)
        {
            passMaterial = PassMat;
            //渲染传递执行时的事件。
            renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;

            //set the id
            Dwon_Up_Struct = new Level[MAXITERATION];
            for (int i = 0; i < MAXITERATION; i++)
            {
                Dwon_Up_Struct[i] = new Level
                {
                    down = Shader.PropertyToID("_BlurMipDown" + i),
                    up = Shader.PropertyToID("_BlurMipUp" + i)
                };
            }
        }
        
        public void SetTarget(RTHandle colorHandle)
        {
            CameraColorTarget = colorHandle;
        }
        public void SetTarget(RenderTargetIdentifier colorHandle)
        {
            currentTarget = colorHandle;
        }
        
        
        
        
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            //传入volume数据
            var stack = VolumeManager.instance.stack; 
            //拿到我们的Volume
            _bloomVolune = stack.GetComponent<BloomVolume>(); 
            
            //获得相机的cameraTarget的描述
            RenderTextureDescriptor colorCopyDescriptor = renderingData.cameraData.cameraTargetDescriptor;
            colorCopyDescriptor.depthBufferBits = (int) DepthBits.None;
            
            
            
            //如果缺少条件就不执行后处理
            if (renderingData.cameraData.camera.cameraType != CameraType.Game||!renderingData.cameraData.postProcessEnabled ||
                renderingData.cameraData.isPreviewCamera )
                return;
            if (passMaterial == null)
                return;
            
            
            
            
            CommandBuffer cmd = CommandBufferPool.Get("cmd Test Bloom");
            using (new ProfilingScope(cmd, m_ProfilingSampler))
            {
                DoBloom(cmd, ref renderingData);
                //Blitter.BlitCameraTexture(cmd, m_CameraColorTarget, m_CameraColorTarget, m_Material, 0);
            }
            
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();

            CommandBufferPool.Release(cmd);
        }

        void ConfigureBloom(CommandBuffer cmd)
        {
            return;
        }
        
        void DoBloom(CommandBuffer cmd, ref RenderingData renderingData)
        {
         
            // Threshold Prefiltered
            Vector4 threshold;
            // threshold.x = Mathf.GammaToLinearSpace(Threshold.value);
            // threshold.y = threshold.x * ThresholdKnee.value;
            // threshold.z = 2.0f * threshold.y;
            // threshold.w = 0.25f / (threshold.y + 0.00001f);
            // threshold.y -= threshold.x;
            // cmd.SetGlobalVector(mBloomThresholdId, threshold);
            
            //to GPU
            // ConfigureBloom(cmd);
            var cameraData = renderingData.cameraData;
            // 传入摄像机
            var camera = cameraData.camera; 
            //获得当前相机渲染目标
            RTHandle sourceRT = cameraData.renderer.cameraColorTargetHandle; 
            //渲染结果图片
            int buffer0 = TempTargetId; 
            
            int width, height;
            width = (int)(camera.scaledPixelWidth / 2);
            height = (int)(camera.scaledPixelHeight / 2);
            
            
            // 降采样
            RenderTargetIdentifier lastDown = currentTarget; //备份

            //升采样和降采样
            for (int i = 0; i < _bloomVolune.maxIterations.value; i++)
            {
                if (height < 1 || width < 1) {
                    break;
                }
                
                int mipDown = Dwon_Up_Struct[i].down;
                int mipUp = Dwon_Up_Struct[i].up;
               
              
                //ToDo:创建Tex，cmd.GetTemporaryRT但是似乎有bug,待定
                cmd.GetTemporaryRT(mipDown, width, height, 0, FilterMode.Bilinear);
                cmd.GetTemporaryRT(mipUp, width, height, 0, FilterMode.Bilinear);
                
                //Gaussian filter
                cmd.Blit(lastDown, mipDown,passMaterial,0);
                cmd.Blit(mipDown, mipUp,passMaterial,1);

                lastDown = mipUp;    
                //降半径
                width /= 2;
                height /= 2;
            }
            
            //渲染到相机
            // cmd.Blit(lastDown, sourceRT, passMaterial,2);
            cmd.Blit(lastDown, sourceRT);
            for (int i = 0; i < _bloomVolune.maxIterations.value; i++)
            {
                cmd.ReleaseTemporaryRT(Dwon_Up_Struct[i].down);
                cmd.ReleaseTemporaryRT(Dwon_Up_Struct[i].up);
            }
            return;
        }
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        //Release
        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            return;
        }
        
        public void Dispose()
        {
            return;
        }
    }
}
```





```C#
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;
using System;

namespace UnityEngine.Experiemntal.Rendering.Universal
{
    //add the struct to the Volume Component
    [Serializable, VolumeComponentMenu("URP14/TestBloom")]
    public class BloomVolume : VolumeComponent,IPostProcessComponent
    {
        //set the parameters UI
        [Range(0f, 10f), Tooltip("模糊的迭代次数")]
        public IntParameter maxIterations = new ClampedIntParameter(1, 0, 10);
        [Range(0f, 10f), Tooltip("模糊半径")]
        public FloatParameter   BlurRange = new ClampedFloatParameter(1.0f, 0.0f, 10.0f);
        [Range(0f, 10f), Tooltip("降采样次数")]
        public IntParameter RTDownSampling = new ClampedIntParameter(1, 1, 10); 
        
        public bool IsActive()
        {
            return active;
        }

        public bool IsTileCompatible()=> false;
    }
}

```

