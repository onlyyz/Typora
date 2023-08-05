#### 1、strcpy

'strcpy': This function or variable may be unsafe. Consider using strcpy_s instead

找到【项目属性】，点击【C++】里的【预处理器】，对【预处理器】进行编辑，在里面加入一段代码：

```CPP
_CRT_SECURE_NO_WARNINGS
```
