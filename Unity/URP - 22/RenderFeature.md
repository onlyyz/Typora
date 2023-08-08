### 1、不透明纹理

#### 1.1 <font color="red">ConfigureInput </font>

使用 ScriptableRenderPassInput.Color 参数调用 ConfigureInput 时
确保渲染传递可以使用<font color=#4db8ff>不透明纹理</font>.

#### 1.2 <font color=#FFCE70>ConfigureTarget</font>

配置本次渲染传递的渲染目标。调用此方法可替代 CommandBuffer.SetRenderTarget 方法。
此方法应在 Configure.SetRenderTarget 中调用。