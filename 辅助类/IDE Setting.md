## Visual Studio

#### 1、属性

```CPP
//解决方案\bin\win32 or 64...\调试
Output Directory	$(SolutionDir)\bin\$(Platform)\$(Configuration)
Intermediate Directory	$(SolutionDir)\bin\intermediates\$(Platform)\$(Configuration)
```

![image-20230715124844116](./assets/image-20230715124844116.png)

#### 2、glfw Library link

```C++
GLFW			$(SolutionDir)Dependencies\GLFW\include
LINK    			$(SolutionDir)Dependencies\GLFW\lib-vc2022
```

![image-20230715224203232](./assets/image-20230715224203232.png)

![image-20230715130107019](./assets/image-20230715130107019.png)

![image-20230715130208971](./assets/image-20230715130208971.png)

#### 3、GLM

```C++
$(SolutionDir)Dependencies\GLM\glm
```

![image-20230715225130652](./assets/image-20230715225130652.png)

#### 4、dll 32位

修改启动项

![image-20230728094820184](./assets/image-20230728094820184.png)

#### 5、断点

1. 设置调试信息格式为：用于“编辑并继续”的程序数据库（/ZI）
操作：
项目->属性->配置属性->C/C++ ->常规 ->调试信息格式

2. 设置生成调试信息为：是（/DEBUG）
操作：
项目->属性->配置属性->链接器->调试->生成调试信息

3. 设置优化为：已禁用（/Od)
操作：
项目->属性->配置属性->C/C++ ->优化

4. 删除解决方案下的.ncb文件
如果有的话

5.工具->选项->调试->要求与原始版本完成匹配 去掉勾选



## Visual Studio Code