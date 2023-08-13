## 环境配置

-   检查显卡支持：http://vulkan.gpuinfo.org/
-   Vulkan SDK：https://vulkan.lunarg.com/

完成后打开<font color="green">Vulkan SDK\Bin\VKCube</font> 查看是否支持Vulkan

![image-20230713204623372](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230713204623372.png)

### Visual Studio

#### 1、属性

```CPP
//解决方案\bin\win32 or 64...\调试
Output Directory	$(SolutionDir)\bin\$(Platform)\$(Configuration)
Intermediate Directory	$(SolutionDir)\bin\intermediates\$(Platform)\$(Configuration)
```

![image-20230715124844116](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715124844116.png)

#### 2、glfw Library link

```C++
GLFW			$(SolutionDir)Dependencies\GLFW\include
LINK    			$(SolutionDir)Dependencies\GLFW\lib-vc2022
```

![image-20230715224203232](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715224203232.png)

![image-20230715130107019](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715130107019.png)

![image-20230715130208971](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715130208971.png)

#### 3、GLM

```C++
$(SolutionDir)Dependencies\GLM\glm
```

![image-20230715225130652](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715225130652.png)

#### 4、Vulkan

```cpp
$(SolutionDir)Dependencies\VulkanSDK\Include
$(SolutionDir)Dependencies\VulkanSDK\Lib
glfw3.lib;vulkan-1.lib
```

![image-20230715231303877](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715231303877.png)

![image-20230715231409323](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715231409323.png)

![image-20230715231457234](E:/dev/Typora-Note/Vulkan/00_Environment.assets/image-20230715231457234.png)