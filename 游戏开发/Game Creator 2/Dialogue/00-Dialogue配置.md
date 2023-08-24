### 一、配置

#### 1.1 G C 2

在安装玩<font color=#4db8ff>Game Creator v2 </font>之后，在Unity <font color="red">Hierarchy</font>面板中可以创建Dialogue对话物体

顺序为<font color="red">Game Creator</font> ->Dialogur ->Dialogue

![image-20230823163434663](assets/image-20230823163434663.png)

创建完之后，会出现<font color=#66ff66>Dialogue</font>物体，点击之后，可以看到关于Dialogue的<font color=#4db8ff>Inspector</font>面板信息。

#### 1.2 对话框

为了可以看见对话，因此我们需要安装对话框，可以在<font color="red">顶部菜单栏</font>上找到<font color=#66ff66>Game creator</font>菜单，随后点击<font color=#4db8ff>Install</font>选项

![image-20230823163844086](assets/image-20230823163844086.png)

其中<font color=#FFCE70>Dialogue</font>下的都是对话框，可以根据选择去安装自己喜欢的对话框。选择对话框顺序如下：

![image-20230823164127263](assets/image-20230823164127263.png)

### 二、创建对话

#### 2.1 普通对话

![image-20230823164735547](assets/image-20230823164735547.png)

其中对话的类型有<font color="red">选择 Choice</font>、<font color=#66ff66>随机 Random</font>、<font color=#4db8ff>文本 Text</font>，利用这个来进行逻辑判断

![image-20230823164903136](assets/image-20230823164903136.png)

#### 2.2 角色

对话角色的创建：<font color="red">Create -> Game Creator -> Dialogue -> Actor</font>

![image-20230823175059069](assets/image-20230823175059069.png)

点击<font color="red">Add Expression</font>添加表情

![image-20230823190453933](assets/image-20230823190453933.png)

随后回<font color=#66ff66>Dialogue</font>，将创建的角色加入<font color="red">Actor</font>

![image-20230823193819535](assets/image-20230823193819535.png)

点击<font color="red">设置 齿轮</font>可以让场景的模型<font color=#4db8ff>GameObject</font>与人物<font color=#4db8ff>Actor</font>绑定

![image-20230823194342810](assets/image-20230823194342810.png)

随后是输入对话内容

![image-20230823194613863](assets/image-20230823194613863.png)

添加动态变量，如：<font color="red">Play Name</font>、<font color=#66ff66>Game Time</font>，而不是写死对话

![image-20230823195901475](assets/image-20230823195901475.png)

![image-20230823195959144](assets/image-20230823195959144.png)

![image-20230823200033375](assets/image-20230823200033375.png)

如图操作即可。

### 三、全局变量

3.1 全局变量数据

![image-20230823200647165](assets/image-20230823200647165.png)

利用全局变量就可以动态设置他们，并且列表参数基本相同

![image-20230823200840573](assets/image-20230823200840573.png)

其中<font color="red">Key</font>是一个可以被替代，搜索的自定义<font color=#66ff66>ID</font>

![image-20230823201047207](assets/image-20230823201047207.png)

随后进行替换

![image-20230823201208647](assets/image-20230823201208647.png)

### 四、音频旁白

![image-20230824100559950](assets/image-20230824100559950.png)

可以添加音频功能，根据<font color=#66ff66>Audio</font>进行选择

### 五、动作时间

在界面中，我们可以通关点击<font color="red">Configure</font>，进入动画选择界面

![image-20230824100819916](assets/image-20230824100819916.png)

当把动画添加进去之后，不仅可以通过调节节点来预览动画，也可以在动画的播放时间线上，添加事件

可以先去Mixamo中获得动画进行调试

link：https://www.mixamo.com/#/?page=2&type=Character

![image-20230824101723785](assets/image-20230824101723785.png)

![image-20230824101954321](assets/image-20230824101954321.png)

在时间线上添加事件之后，我们可以给事件添加条件

![image-20230824102312939](assets/image-20230824102312939.png)

### 六、跳转

#### 6.1 文本行跳转

<font color="red">Duration</font> 指定文本跳转下一行需要多长时间，一般选择的是<font color=#66ff66>Until Interaction</font>，即：当用户点击界面时，才会跳转到下一行

也可以在经过一段时间、语音播放结束、动画播放结束时自动切换到下一行

![image-20230824104358172](assets/image-20230824104358172.png)

#### 6.2 文本间跳转

文本之间跳转可以通过<font color="red">Jump</font>，默认是<font color=#4db8ff>Continue</font>，它有三个选项

![image-20230824110347294](assets/image-20230824110347294.png)

在剧情中可以使用<font color="red">jump</font>进行文本间跳转，但是需要知道是跳转到那一行，可以通过给文本添加<font color=#66ff66>Tag</font>进行标记，随后跳转指定。

右键呼出

![image-20230824110917561](assets/image-20230824110917561.png)

![image-20230824111413697](assets/image-20230824111413697.png)

![image-20230824111033852](assets/image-20230824111033852.png)

类似于Art3的跳转功能

![image-20230824111251406](assets/image-20230824111251406.png)

### 七、文本对话

#### 7.1 对话顺序

其中文本对话逻辑为，先父子层级对话，随后是同层级对话，对话顺序为：A  - B - C

![image-20230824112432380](assets/image-20230824112432380.png)

#### 7.2 条件

利用<font color="red">Add Condition</font>可以给对话添加条件，即<font color=#4db8ff>Bool、Int、String</font>对应<font color=#4db8ff>是与否、数字、句子</font>。条件可以钳制，判断是否让对话发生。

如：
<font color=#FFCE70>Bool </font>：Math \ Boolean \ Bool

![image-20230824113418130](assets/image-20230824113418130.png)

<font color=#FFCE70>Int</font>：。。。

<font color=#FFCE70>String</font>：。。。

###八、 随机对话

故事的随机对话选择，点击工具栏上面的骰子图标，或者将某段对话的类型改为随机

![image-20230824114111237](assets/image-20230824114111237.png)

设置一小段随机对话![image-20230824115619291](assets/image-20230824115619291.png)

### 九、选择对话

其中，

<font color=#66ff66>Hide Unavailable</font> 对玩家隐藏返回为 <font color="red">否 和 False</font>的选项

<font color=#66ff66>Skip Choice</font> 跳过选择对话框，确定是否执行所选的对话节点

<font color=#66ff66>Shuffle Choices</font> 采用类似洗牌的方式，使得对话顺序随机化

<font color=#66ff66>Timed Choice</font> 允许玩家有<font color="red">固定时间</font>去做选择

当勾选<font color=#66ff66>Timed Choice</font> 会多出两个选择，一个是固定时间，一个定义了行为，变成伪随机

![image-20230824141816947](assets/image-20230824141816947.png)

伪随机中，它可以固定选择第一个或者最后一个

![image-20230824142115289](assets/image-20230824142115289.png)

### 十、任务模块

可以直接创建任务模块，通过：Creator \ Game Creator \ Quests \ Quest

