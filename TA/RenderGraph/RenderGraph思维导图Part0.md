# RenderGraph整理

## Class

* RenderGraphResourceRegistry[单例]
    - Data
        - static m_CurrentRegistry
        - DynamicArray<IRenderGraphResource>[]m_Resources【包含ComputeBufferResource,TextureResource】
        - TexturePool m_TexturePool
        - int m_TextureCreationIndex;
        - ComputeBufferPool m_ComputeBufferPool
        - DynamicArray< RendererListResource>  m_RendererListResources
        - RenderGraphDebugParams m_RenderGraphDebug
        - RenderGraphLogger m_Logger
        - int m_CurrentFrameIndex
        - RTHandle m_CurrentBackbuffer
    - Class
        - IRenderGraphResource
            - Data
                - bool imported
                - int cachedHash
                - int transientPassIndex
                - uint writeCount
                - bool wasReleased
                - bool requestFallBack
            - Func [virtual]
                - Reset()
                - GetName()
                - IsCreated()
            - Func
                - IncrementWriteCount()
                - NeedsFallBack()
            - 派生
                -  RenderGraphResource<DescType[struct], ResType[class]>
                    - DescType desc
                    - ResType resource
                    - Func
                        - 构造函数
                        - Reset()
                            - base.Reset()
                            - resource=null
                        - IsCreated()
                            - resource!=null
                    - 派生
                        - TextureResource: RenderGraphResource<TextureDesc, RTHandle>
                            - Func
                                - GetName()
                                    - 判断是否是import
                                        - [即由现有的RenderTargetIdentify转换到TextureHandle]
                                        - [即由现有的RTHandle转换到TextureHandle]
                                        //- [即由现有的ComputeBuffer转换到ComputeBufferHandle]

                                        - 否则返回通过正常手段创建的Resoure的name  (desc.name;)
                        - ComputeBufferResource: RenderGraphResource<ComputeBufferDesc, ComputeBuffer>
                            - Func
                                - GetName()
                                    - 判断是否是import
                                        - [即由现有的ComputeBuffer转换到ComputeBufferHandle]
                                            - 直接返回"ImportedComputeBuffer"
                                        - 否则返回通过正常手段创建的Resoure的name  (desc.name;)
    - struct 
        - RendererListResource
            - Data
                - RendererListDesc desc;
                - RendererList rendererList;
            - 构造
                - RendererListResource(in RendererListDesc desc)
                    - RenderListDesc传递[Renderer list creation descriptor.]
                        - RenderListDesc包含
                            - SortingCriteria
                            - PerObjectData
                            - RenderQueueRange
                            - RenderStateBlock
                            - overrideMaterial
                            - excludeObjectMotionVectors
                            - layerMask
                            - overrideMaterialPassIndex
                            - 强制包含
                                - CullingResults
                                - Camera
                                - passName
                                - passNames
                    - 新构造一个RendererList
                        - RenderList包含
                            - static data
                                - ShaderTagId s_EmptyName
                                - RendererList nullRendererList
                            - Data
                                - isValid
                                - cullingResult
                                - drawSettings
                                - filteringSettings
                                - stateBlock
                            - Func
                                - RendererList Create(in RendererListDesc desc)
                                    - newRenderList(如果参数的RendererListDesc不可用【即IsValid为false】就直接返回默认的newRenderList)
                                    - 如果RendererListDesc可用就给新建的newRenderList的成员(sortingSettings,drawSettings,filterSettings,stateBlock)赋值,然后返回
    - Func
        - Resource[Texture/ComputeBuffer/RenderList] Getter + TextureNeedsFallback
            -  RTHandle GetTexture
            返回TextureHandle对应m_Resources(DynamicArray< IRenderGraphResource>)的TextureResource的RTHandle

            - bool TextureNeedsFallback(in TextureHandle handle)
                返回判断该TextureHandle是否需要FallBack为Black(2D),blackTexture3DXR(3D)

            - GetRendererList(in RendererListHandle handle)
                返回m_RendererListResources中Handle对应的RendererList

            - GetComputeBuffer(in ComputeBufferHandle handle)
            返回ComputeBufferHandle对应m_Resources(DynamicArray< IRenderGraphResource>)的ComputeBufferResource的ComputeBuffer

        - Internal Interface
            - [构造]
                - RenderGraphResourceRegistry(RenderGraphDebugParams renderGraphDebug, RenderGraphLogger logger)
            - BeginRender(int currentFrameIndex, int executionCount)
                - currentFrameIndex赋值
                - ResourceHandle.NewFrame新建帧(Hash运算出对应Hash码)
                
            - EndRender()
                - 把当前的RenderGraphResourceRegistry单例设置回null
            -  返回资源状态成员变量[Validity,Name,Imported,Created,TransientIndex([仅供当前Pass中使用资源]Pass的Index)]
                - CheckHandleValidity(in ResourceHandle res)/(RenderGraphResourceType type, int index)
                - IncrementWriteCount(in ResourceHandle res)
                    资源对象写入时调用WriteComputeBuffer/ReadWriteTexture之后资源对应的writeCount++
                - GetResourceName
                - IsResourceImported
                - IsResourceCreated
                - GetResourceTransientIndex
                - IsRendererListCreated
            - 导入ComputeBuffer/Texture资源/
            导入BackBuffer[最后Blit的RenderTarget]/
            创建ComputeBuffer/Texture资源/
            返回Texture/ComputeBuffer资源的总数ResourceCount/
            ResourceHandle获取对应的Resource/ResourceDescriptor/ResourceCount
            RendererListDesc创建RendererListHandle
                - Create
                    - CreateTexture
                    - CreateAndClearTexture
                    - CreateComputeBuffer
                    - CreateRendererList
                    - CreateAndClearTexture(RenderGraphContext rgContext, int index)
                    [真正创建RenderTexture的地方]
                    - CreateComputeBuffer(RenderGraphContext rgContext, int index)
                    [真正创建ComputeBuffer的地方]
                - Import 
                    - ImportTexture(RTHandle rt)
                    - ImportBackbuffer(RenderTargetIdentifier rt)
                    - ImportComputeBuffer
                - Release
                    - ReleaseTexture
                    - ReleaseComputeBuffer
                - Getter
                    - Resource
                        - GetComputeBufferResource
                        - GetTextureResource
                    - ResourceDesc
                        - GetTextureResourceDesc
                        - GetComputeBufferResourceDesc
                    - ResourceCount
                        - GetTextureResourceCount
                        - GetComputeBufferResourceCount
                - AddNewResource 返回资源句柄的ID

            - Validate TextureDesc/RendererListDesc/ComputeBufferDesc可用性
                - ValidateTextureDesc
                - ValidateRendererListDesc
                - ValidateComputeBufferDesc
            
            - 创建RendererLists
                - CreateRendererLists(List<RendererListHandle> rendererLists)
            - 清理资源相关
                - clear清理资源【Excute的Final的时候】
                    - Clear(bool onException)
                        - m_Resources[0]/[1]
                        - m_RendererListResources
                        - m_TexturePool.CheckFrameAllocation
                            - 把当前帧中在m_FrameAllocatedResources里所有申请过的资源
                            放置到Pool中，然后让PurgeUnusedResources清理
                        - m_ComputeBufferPool.CheckFrameAllocation
                            - 同上
                - PurgeUnusedResources清理10帧之内的无效资源【EndFrame的时候(当前帧的最后)】
                - Cleanup 完全清理所有资源切换是否启用RenderGraph时使用
            - Log相关函数
                - LogTextureCreation
                - LogTextureRelease
                - LogComputeBufferCreation
                - LogComputeBufferRelease
                - LogResources
* RenderGraphResourcePool
    - RenderGraphResourcePool< Type>[抽象类]
        - Dictionary通过哈希值跟踪资源，并将具有相同哈希值的资源存储在List中
        （用List代替堆栈，因为我们需要能够删除陈旧无用资源，这些资源可能位于堆栈的中间）。

        - Data
            - Dictionary<int, List<(Type resource, int frameIndex)>> m_ResourcePool
            - List<(int, Type)> m_FrameAllocatedResources
            - int s_CurrentFrameIndex
            - const int kStaleResourceLifetime = 10;
            - ResourceLogInfo
                - string name
                - long size;
        - Abstract Func
            - protected
                - void ReleaseInternalResource(Type res)
                - string GetResourceName(Type res)
                - long GetResourceSize(Type res)
                - string GetResourceTypeName()
            - public
                - void PurgeUnusedResources(int currentFrameIndex);
        - Func
            - ReleaseResource(int hash, Type resource, int currentFrameIndex)
            如果在pool中找不到hash对应的list，就new一个出来，存放到pool中
            这个list主要用于记录当前资源的引用和其使用的当前帧数，如果超过10帧(常数)没有用过才会销毁
            [PurgeUnusedResources(由对应资源池(TexturePool,ComputeBuffer)类实现释放)]

            - TryGetResource(int hashCode, out Type resource)
            从Pool里的Dictionary用hash id获取对应的list且如果list的count大于0的话就通过Out关键词返回对应资源并从list中去除掉位于最后的元素

            - CleanUp 清理pool中所有资源池对应的资源(Release)
            - RegisterFrameAllocation(int hash, Type value) 
            往资源池(Texture resource pool/Compute Buffer pool)添加申请的资源引用(注册)存放于对应资源池的m_FrameAllocatedResources链表

            - UnregisterFrameAllocation(int hash, Type value)
            从m_FrameAllocatedResources链表中移除掉资源引用

            - CheckFrameAllocation(bool onException, int frameIndex)
            需要将所有的资源释放到池中，避免泄露(ReleaseResource)

            - LogResources(RenderGraphLogger logger)
            - ShouldReleaseResource(int lastUsedFrameIndex, int currentFrameIndex)
            判断资源十帧之前是否依旧有用到过。

        - 派生
            - TexturePool : RenderGraphResourcePool<RTHandle>
                - Func
                    - ReleaseInternalResource(RTHandle res)释放资源
                    - GetResourceName(RTHandle res) 直接返回RTHandle对应的RenderTexture的name
                    - GetResourceSize(RTHandle res) 
                    Profiling.Profiler.GetRuntimeMemorySizeLong返回RT的资源大小

                    - GetResourceTypeName() 
                    返回资源名称"Texture"

                    - void PurgeUnusedResources(int currentFrameIndex)
                    移除在资源池m_ResourcePool中应该被移除的资源

            - ComputeBufferPool : RenderGraphResourcePool<ComputeBuffer>
                - Func
                    - ReleaseInternalResource(RTHandle res)释放资源
                    - GetResourceName(RTHandle res) 
                    直接返回默认字段"ComputeBufferNameNotAvailable" :(

                    - GetResourceSize(RTHandle res)]
                    返回buffer中元素数量*每个元素的步长

                    - GetResourceTypeName() 
                    返回"ComputeBuffer"

                    - void PurgeUnusedResources(int currentFrameIndex)
                    移除在资源池m_ResourcePool中应该被移除的资源
* RenderGraphResources
* RenderGraphPass
* RenderGraphObjectPool
* RenderGraphBuilder
* RenderGraph
* ...

## Struct

* RenderGraphParameters[RenderGraph]

    - currentFrameIndex
    - scriptableRenderContext
    - commandBuffer

* ...
