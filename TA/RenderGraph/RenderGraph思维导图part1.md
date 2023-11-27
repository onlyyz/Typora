# RenderGraph整理

## Class

* RenderGraphResourceRegistry[单例]
* RenderGraphResourcePool
* RenderGraphResources
    - enum
        - RenderGraphResourceType
            - Texture
            - ComputeBuffer
            - Count
        - TextureSizeMode
            - Explicit
            - Scale
            - Functor
    - struct
        - Handle
            - ResourceHandle
                - Data
                    - const
                        - kValidityMask = 0xFFFF0000
                        - kIndexMask = 0xFFFF
                    - static
                        - s_CurrentValidBit 当前帧executionIndex对应的HashID
                    - m_Value  当前资源句柄的ID
                    - type [RenderGraphResourceType]
                        - iType[Getter]
                - Func
                    - 构造 
                        - ResourceHandle(int value,RenderGraphResourceType type)
                            - value:资源在resourceArray中的index
                            - 将value的高16位屏蔽掉，然后再跟当前帧executionIndex的HashID(s_CurrentValidBit)
                            做按位或操作得到一个HashID，目的应该是要让资源在每帧的HashID都不一样

                    - 隐式转换 
                        - int(ResourceHandle handle) 
                            - 返回handle.index
                    - IsValid()
                        - 验证当前资源是否合法，当前资源的高16位HashID是否等于当前帧executionIndex的HashID
                    - static 
                        - NewFrame(int executionIndex)
                            - 算当前帧executionIndex对应的静态哈希码(当前是静态方法)
            - TextureHandle
                - Data
                    - static
                        - TextureHandle s_NullHandle
                    - internal 
                        - ResourceHandle handle
                - Func
                    - nullHandle Getter
                    - 隐式转换
                        - 转换之前先使用Isvalid检查合法性
                        - RTHandle(TextureHandle texture)
                        - RenderTargetIdentifier(TextureHandle texture)
                        - RenderTexture(TextureHandle texture)
                    - IsValid检查句柄合法性
            - ComputeBufferHandle
                - Data
                    - static
                        - ComputeBufferHandle s_NullHandle
                    - internal 
                        - ResourceHandle handle
                - Func
                    - nullHandle Getter
                    - 隐式转换
                        - 转换之前先使用Isvalid检查合法性
                        - ComputeBuffer(ComputeBufferHandle buffer)
                    - IsValid检查句柄合法性
            - RendererListHandle
                - Data
                    - m_IsValid
                    - handle
                - Func
                    - 构造 RendererListHandle
                    - 隐式转换 
                        - int(RendererListHandle handle)
                        - RendererList(RendererListHandle rendererList)
                    - IsValid检查句柄合法性
        - Descriptor
            - FastMemoryDesc
                - Data
                    - inFastMemory
                    - [enum]FastMemoryFlags flags
                        - None
                        - SpillTop
                        - SpillBottom
                    - residencyFraction
            - TextureDesc
                - Data
                    - TextureSizeMode sizeMode
                    - int width
                    - int height
                    - int slices
                    - Vector2 scale
                    - ScaleFunc func
                    - DepthBits depthBufferBits
                    - GraphicsFormat colorFormat
                    - FilterMode filterMode
                    - TextureWrapMode wrapMode
                    - TextureDimension dimension
                    - bool
                        - enableRandomWrite
                        - useMipMap
                        - autoGenerateMips
                        - isShadowMap
                        - anisoLevel
                        - mipMapBias
                        - enableMSAA
                        - bindTextureMS
                        - useDynamicScale
                        - fallBackToBlackTexture
                        - clearBuffer
                    - MSAASamples msaaSamples
                    - RenderTextureMemoryless memoryless
                    - string name
                    - FastMemoryDesc fastMemoryDesc
                    - Color clearColor
                - Func
                    - InitDefaultValues(bool dynamicResolution, bool xrReady)
                    - 构造
                        - TextureDesc
                            - (int width, int height, bool dynamicResolution = false, bool xrReady = false)
                            - (Vector2 scale, bool dynamicResolution = false, bool xrReady = false)
                            - (ScaleFunc func, bool dynamicResolution = false, bool xrReady = false)
                            - (TextureDesc input)
                    - int GetHashCode()
            - ComputeBufferDesc
                - Data
                    - int 
                        - count
                        - stride
                    - ComputeBufferType type
                    - string name
                - Func
                    - 构造 ComputeBufferDesc
                        - (int count, int stride)
                        - (int count, int stride, ComputeBufferType type)
                    - int GetHashCode() 
* RenderGraphPass
    - RenderGraphPass
        - Data
            - name
            - index
            - ProfilingSampler customSampler
            - bool 
                - enableAsyncCompute
                - allowPassCulling
                - generateDebugData
            - TextureHandle[] colorBuffers kMaxMRTCount = 8;
            - TextureHandle depthBuffer
            - int
                - colorBufferMaxIndex
                - refCount
            - List< ResourceHandle>[] 
                - resourceReadLists
                    - var resourceRead = passInfo.pass.resourceReadLists[type];
                    foreach (var resource in resourceRead)
                    {
                        ref CompiledResourceInfo info = ref m_CompiledResourcesInfos[type][resource];
                        info.consumers.Add(passIndex);
                        info.refCount++;
                    把Pass中resourceReadLists的TextureHandle/ComputeBufferHandle
                    全部都隐式转换成int然后取出对应唯一位置的
                - resourceWriteLists(ToDo)
                - transientResourceList(ToDo)
            - List< RendererListHandle> usedRendererListList(ToDo)
        - Func
            - GetExecuteDelegate
            返回Pass的执行回调函数

            - abstract
                - Execute
                - Release
                - HasRenderFunc
            - 构造
                - RenderGraphPass
                new出三个 List< ResourceHandle>[]
                resourceReadLists,resourceWriteLists,transientResourceList

            - Clear
                - 清空所有List,Buffer全部恢复到nullHandle
            - AddList
                - AddResourceWrite
                - AddResourceRead
                - AddTransientResource
                - UseRendererList
            - SetState
                - EnableAsyncCompute
                - AllowPassCulling
                - GenerateDebugData
            - SetBuffer[初始化设置好ColorBuffer和DepthBuffer]
                - SetColorBuffer
                - SetDepthBuffer
    - RenderGraphPass< PassData>
        - Data
            - PassData data
            - RenderFunc< PassData>renderFunc;
        - Func
            - Excute 执行回调delegate函数
            - Initialize 先执行Clear然后初始化index,passData,passName,sampler
            - Release 
            先执行Clear将render data/当前RenderGraphPass Release到RenderGraphObjectPool中
            - HasRenderFunc 返回当前renderFunc是否为空
    
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
