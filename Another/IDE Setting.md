## Visual Studio	

1、属性

```CPP
//解决方案\bin\win32 or 64...\调试
Output Directory	$(SolutionDir)\bin\$(Platform)\$(Configuration)
Intermediate Directory	$(SolutionDir)\bin\intermediates\$(Platform)\$(Configuration)
```

![image-20230715124844116](E:/dev/Typora-Note/Another/IDE%20Setting.assets/image-20230715124844116.png)

2、glfw Library link

```C++
GLFW			$(SolutionDir)Dependencies\GLFW\include
LINK    			$(SolutionDir)Dependencies\GLFW\lib-vc2022
```

![image-20230715224203232](E:/dev/Typora-Note/Another/IDE%20Setting.assets/image-20230715224203232.png)

![image-20230715130107019](E:/dev/Typora-Note/Another/IDE%20Setting.assets/image-20230715130107019.png)

![image-20230715130208971](E:/dev/Typora-Note/Another/IDE%20Setting.assets/image-20230715130208971.png)

GLM

```C++
$(SolutionDir)Dependencies\GLM\glm
```

![image-20230715225130652](E:/dev/Typora-Note/Another/IDE%20Setting.assets/image-20230715225130652.png)





## Visual Studio Code