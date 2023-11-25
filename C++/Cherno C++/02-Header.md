<font color=#4db8ff>Variables</font>的本质区别其实就是<font color=#4db8ff>内存</font>的差距



<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=9RJTQmK0YPI&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=10

#### 一、Memory

#### 1.1 int

内存大小本质也取决于编译器的选择

byte = 8 bit

```c++
char 1 byte
short 2 byte    
int 4 byte = long = float
long long = 8byte = double
```

#### 1.2 #pragma

以“ **#** ” 开头的任何内容都是预处理命令，下面预处理指令的含义是只包含一次，防止多次包含 <font color=#4db8ff>.h</font> 头文件，多次Copy 代码

```c++
#pragma once
```

但是更为实际的方式是如下

```c++
#ifndef _LOG_H
#define _LOG_H
#endif
```

其中 “ **ifndef** ” 的本意是 <font color=#4db8ff>” if no define the “</font>

#### 1.3 <>

```c++
#include <XXX>
```

用于包含整个文件夹，其中 “ **<>** ” 仅适用于编译器以及包含所有内容的引号

#### 二、Debug in VS

<font color=#4db8ff>link：</font>https://www.youtube.com/watch?v=0ebzPwixrJA&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=11

#### 2.1 Break Point

```c++
#include <iostream>
#include "Log.h"

int main() 
{
	Log("Hello World");
	std::cin.get();
}
```

![image-20231121221343676](./assets/image-20231121221343676.png)

<font color=#4db8ff> F9 </font>断点

![image-20231121222951493](./assets/image-20231121222951493.png)

内存

![image-20231121223333974](./assets/image-20231121223333974.png)

查看内存位置

初始内存为，意思是未初始化内存值，CC为十进制的 <font color=#4db8ff>204</font>,，16位，刚好对其四个bit。下面是八个十六进制，刚好是四个字节的内存

```c++
CC CC CC CC 
```

<font color=#4db8ff>Ctrl + Alt + W</font> 打开“监视”窗口

<font color=#4db8ff>ctrl+ alt+ C</font>  调用堆栈