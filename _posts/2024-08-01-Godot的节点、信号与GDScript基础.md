---
# author:
title: Godot的节点、信号概念与GDScript基础
description: >-
  虽说Godot支持C#和C++等其它语言，但目前阶段GDScript更能发挥出Godot的特性与优势，其开发社区的资料文档也更为全面完整，此博客记录了Godot中节点与信号的重要概念，以及GDScript的基本编写语法
date: 2024-08-01 15:19:00 +0800
categories: [游戏开发, 游戏引擎]
tags: [Godot]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
# math: true
# mermaid: true
# image:
#   path: /resources/xxxxxx.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

## 一、节点的概念

### 1.1 节点（Node）是什么

#### 1.1.1 节点的特征与优势
- 节点是Godot内最常用最基本的开发组件，开发思想接近人类的思想
- 节点之间可以有父子关系，一个节点可以有多个子节点
- 大部分节点都具有非常具体的功能，往往直接涉及人类的视听感官，如显示图片、播放音乐、显示模型、模拟物理体等；比如一个有贴图、可以检测敌人以便造成伤害的子弹，我们可以为之创建一个`Area2D`节点（可以进行区域的检测），然后为之添加子节点`CollisionShape2D`用于确定区域的形状（类似于Unity中添加xxxCollider2D的Component一样），然后可以添加用于显示子弹贴图的子节点`Sprite2D`，然后再添加相应代码即可

![GodotNodeIntroduce.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotNodeIntroduce.png)

#### 1.1.2 节点在场景树与服务器的控制下运行
- 场景树是游戏的主循环对象，只有在树下的节点才可以正常发挥其功能（节点初始化时并不在树下，需要手动处理）
- 服务器是Godot内置的各类Server（服务器的编码接近底层，保障游戏运行效率），节点本身并不具备实际功能，其主要功能由Server实现
- 节点与Server通过场景树进行协调沟通

#### 1.1.3 节点通过场景组织
- 场景是若干节点的集合，场景文件是记录若干节点集合的文件（`.tscn`文件）
- 游戏实际运行时不存在所谓场景，场景是节点在文件系统中储存和加载的单位
- 游戏从主场景开始运行

### 1.2 节点及其组件的调用

#### 1.2.1 通过路径引用
>通过路径引用的坏处是如果我们更改了节点名，则代码中的路径会失效；最好只在我们想要访问的节点是当前工作节点（脚本所在节点）的子节点时才使用路径

##### 1.2.1.1 直接使用路径
- 符号`$`是`get_node()`函数的简写
- 我们在`Main`节点下创建名为`Label0`的`Label`类型子节点用于显示文字（类似于Unity中的`TextMeshPro3D`等），然后我们可以在`Main`的脚本中用`$Label0`（可以直接把节点拖入代码编辑器）来指向这个节点，并借之调用该节点的一些属性

```gdscript
# 这个是父节点Main的脚本
func _ready()
	# 在节点被创建时，将标签文本内容初始化为字符串
	$Label0.text = "Girls Band Cry"
	# 标签文本的颜色为绿色
	$Label0.modulate = Color.GREEN
```

- 对于子节点的子节点，拖入脚本中会产生一条路径（相对路径，从脚本当前路径开始，此处为Main节点上的脚本，而Player位于Main脚本之下）来定位这个对象，比如

```gdscript
$Player/Weapon
# 等价于get_node("Player/Weapon")
```

##### 1.2.1.2 将路径保存起来
- 我们可以将这些引用保存为一个更方便使用的变量，这只需要我们将节点拖入脚本释放的同时按住`Ctrl`键即可自动生成如下变量定义语句

```gdscript
# 自动产生一个同名的变量
# 因为Godot中节点的创建有着严格的顺序，如果我们打开游戏并试图在Weapon节点存在之前查找他，那么就会报错，所以此处使用@onready来确保所有子节点都被创建完毕后再进行使用
@onready var weapon = $Player/Weapon
```

- 我们也可以通过函数`get_path()`来获取绝对路径

```gdscript
print(weapon.get_path())
# /root/Main/Player/Weapon
```

#### 1.2.2 通过`@export`赋值的方式获取节点
- 我们可以通过定义一个检查器中可赋值的（特定类型的）节点来引用我们所需要的节点

```gdscript
@export var my_node: Sprite2D
```

- 通过`if`检测节点的种类来判断我们是否进行进一步的操作，此时要注意节点类型间的继承关系，如`Node2D`就是`Sprite2D`的父类

```gdscript
@export var my_node: Sprite2D

func _ready()
	# Sprite2D是Node2D的子类，所以此条判断会被检测为true
	if my_node is Node2D:
		print("Is Node2D")
# Is Node2D
```

#### 1.2.3 使用`%`访问独特的唯一名称节点
- 我们可以将比如GameManager这种我们确定整个游戏只会存在一个的东西设置为一个特殊的唯一节点，这样这个节点右侧就会出现一个%符号，再按一下可以取消该设置

![GodotSetAsUniqueName.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotSetAsUniqueName.png)

- 此时我们就可以通过`%`在任何相同场景中的脚本中访问这个GameManager了

```gdscript
@onready var game_manager = %GameManager
```

- 注意：必须确保引用GameManager的脚本和GameManager必须处在同一个场景树下

### 1.3 节点间的通信

#### 1.3.1 父子节点间的通信
>向下调用，向上传递信号
- 观察以下节点结构（节点间的父子关系并不能代表脚本中类之间有继承关系），父节点`Main`可以调用子节点中的成员变量和函数
- 子节点无法获取父类的成员变量或者调用其函数，所以需要使用信号来向上传递信息，父节点可以选择性地接收这些信号

```
Main
	Player
		Collider
		Sprite2D
	Enemy
		Collider
		Sprite2D
```

#### 1.3.2 兄弟节点间的通信
- 上述节点结构中，`Player`和`Enemy`节点互为兄弟节点，我们若想实现兄弟节点间的信息传递，可以通过共同的父节点，从其中一个兄弟节点获取信号并将其链接到另一个兄弟节点的函数上
- （我有点没搞懂为什么规范我们这样干，为啥不能直接把兄弟节点的信号直接链接给另一个兄弟节点，非要通过父类节点中转呢，可能是方便统一管理？）

## 二、信号的概念

### 2.1 信号（Signal）的基本用法
>信号是节点之间可以相互发送的消息，我们使用信号来告知别的节点某事件已经发生，有点类似Unity引擎中的事件（Event）系统

#### 2.1.1 按钮按下的信号
- 我们以`Button`节点为例，选中这个节点后我们可以看到该类节点上所有信号的列表

![GodotButtonSignal.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotButtonSignal.png)

- `Button`节点是`Main`节点的子节点，我们以`Button`节点上的`pressed()`信号为例，将其链接到此处的`Main`节点的脚本上：双击这个`pressed()`信号，弹出下面的界面

![GodotButtonPressedSignal.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotButtonPressedSignal.png)

- 选择Main后点Connect按钮，这将会在`Main`节点的脚本上创建一个对应的函数

```gdscript
func _on_button_pressed():
	pass
```

- 此函数旁会有一个绿色箭头，这意味着有一个信号链接到了此函数，点击箭头会跳出相关信息

![GodotButtonSignalOnScript.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotButtonSignalOnScript.png)

- 我们将函数中的`pass`替换为按下按钮后（即`pressed`信号被发出）要执行的语句即可

#### 2.1.2 计时器的信号
- 节点`Timer`有一个信号`timeout()`，这个计时器倒数一定时间长度后会发出这个信号，可以用于比如技能冷却等功能的实现

### 2.2 信号的自定义创建

#### 2.2.1 不传参的信号
- 比如我们自己的`Main`节点由子节点`Player`和子节点计时器`Timer`组成，我们想让`Player`脚本中的经验值满了后进行升级，而这个升级需要作为一个信号来触发`Main`中的技能解锁，总之这个时候我们需要在`Player`节点脚本处创建一个信号来完成这个信息的传递

```gdscript
# Player节点脚本中
extends Node

# 创建信号，使得该脚本所属的节点产生新的可选信号，可以在别处调用
signal level_up

# 经验值
var xp := 0

# 计时器的timeout信号链接，实现每隔几秒增加5经验值
func _on_timer_timeout():
	xp += 5
	print(xp)
	# 经验值超过20时升级
	if xp >= 20:
		xp = 0
		print("level up!")
		# 释放升级的信号
		level_up.emit()
```

- 在`Main`脚本处，可以添加这个信号的链接，除了手动在引擎中添加外，还可以通过代码来链接或者断开链接，以下是另一个节点的脚本

```gdscript
# Main节点脚本中
extends Node

# 保存Player节点的引用
@onready var player = $Player

# 用于储存player中的升级信号
var level_up_signal: Signal

func _ready():
	# 获取player中的升级信号
	level_up_signal = player.level_up
	# 真正与player中的升级信号建立链接，此时_on_player_level_up函数左侧才会出现绿色箭头
	# 若想要断开链接，则同理调用disconnect函数即可
	level_up_signal.connect(_on_player_level_up)

func _on_player_level_up():
	print("new skill unlocked!")
```

- 此时我们运行场景，得到以下输出

![GodotLevelUpSignal.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotLevelUpSignal.png)

#### 2.2.2 传参的信号
- 同样还是玩家升级的信号，我们新增要求：玩家在特定等级解锁特定等级的技能，这需要我们将玩家等级记录下来并传递在信号中
- 我们需要在三个地方写出需要传递的参数列表
- 第一处：信号定义的参数列表处；第二处：信号`emit()`函数括号内

```gdscript
# Player节点脚本中
extends Node

# 创建可以传参的信号，当然，也可以传多个参数
signal level_up(level: int)

# 经验值
var xp := 0
# 等级
var level := 0

# 计时器的timeout信号链接，实现每隔几秒增加5经验值
func _on_timer_timeout():
	xp += 5
	# 经验值超过20时升级
	if xp >= 20:
		xp = 0
		level += 1
		print("player level: " + str(level))
		# 释放升级的信号，注意这里也要把需要传递的参数传递出去
		level_up.emit(level)
```

- 第三处：信号在被建立连接的实现函数的参数列表处

```gdscript
# Main节点脚本中
extends Node

# 保存Player节点的引用
@onready var player = $Player

# 用于储存player中的升级信号
var level_up_signal: Signal

func _ready():
	# 获取player中的升级信号
	level_up_signal = player.level_up
	# 真正与player中的升级信号建立链接，此时_on_player_level_up函数左侧才会出现绿色箭头
	# 若想要断开链接，则同理调用disconnect函数即可
	level_up_signal.connect(_on_player_level_up)

# 这里注意信号定义时如果有传参，则这里也应当同样格式地写出来
func _on_player_level_up(level: int):
	if level == 3:
		print("skill at Lv.3 unlocked!")
	if level == 6:
		print("skill at Lv.6 unlocked!")
```

- 运行后得到以下输出

![GodotLevelUpSignalWithParameter.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotLevelUpSignalWithParameter.png)

## 三、关于脚本
- GDScript是专门为了Godot进行游戏开发使用的面向对象的编程语言，其语法与Python较为相似
- 在新建项目中创建一个名为`Main`的`Node`类型节点后，为这个节点添加GDS脚本`xxx.gd`，其会被初始化为如下形式
	- `_ready()`函数在该节点被初始化后立刻执行，类似Unity的`Start()`或`Awake()`
	- `_process(delta)`函数类似于Unity中的`Update`

```gdscript
extends Node

# Called when the node enters the scene tree for the first time.
func _ready():
	pass # Replace with function body.

# Called every frame. 'delta' is the elapsed time since the previous frame.
func _process(delta):
	pass
```

## 四、变量与常量

### 4.1 变量的定义与修改
- 通过`var`关键字定义变量

```gdscript
extends Node

# var 变量名称 = 默认值
var health = 100

# 游戏开始时在控制台处打印生命值
func _ready():
	print(health)
	# 打印结果为100
```

- 通过赋值修改变量值；赋值也可以用`+=`，`-=`，`*=`，`/=`等

```gdscript
extends Node

var health = 100

func _ready():
	health = 20 + 20
	health += 10
	print(health)
	# 打印结果为50
```

### 4.2 量的作用域
- 例如`if`语句内或者函数内部声明一个新变量，则只能在该结构内部使用
- 若想在全局都能使用，则将变量定义在外部

### 4.3 数据类型

#### 4.3.1 动态类型
- GDScript中定义变量无需写出数据类型
- 甚至可以像python一样更改同一变量名的数据类型

```gdscript
# bool值
var my_bool = true
# 数字
var my_int = 100
# 更改数据类型
var my_int = false
```

- 数据类型转换，`int`转成`string`

```gdscript
var num = 5
var text = "A band usually has " + str(num) + " members"
print(text)
# A band usually has 5 members
```

- 数据类型转换，`float`转成`int`，会仅仅去掉小数位（并无四舍五入）

```gdscript
var decimal = 3.94
print(int(decimal))
# 3
```

#### 4.3.2 设置静态类型
- 我们也可以使得一个变量的类型不变

```gdscript
# 设置为整型，该变量名后续无法被改变数据类型
var damage: int = 15
```

- 也可以使用推断类型，让系统自己判断数据类型

```gdscript
# 系统会自动判断为整型，该变量名后续同样无法被改变数据类型
var damage := 15
```

- 通过添加`@export`前缀来使得我们可以在节点的检查器Inspector中设置该变量的值，这样的话引擎只认你在检查器中设置的值，而在脚本中等号后赋予的值将只是一个默认值而已（类似Unity中脚本附着的GameObject上的脚本组件处可以编辑公共变量的值或者`[serializefield]`前缀的私有变量）

```gdscript
@export var damage := 15
# 假设我们在Inspector中设置damage值为100
print(damage)
# 100
```

![GodotInspectorSetValue.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotInspectorSetValue.png)

#### 4.3.3 向量
- 常用二维向量`Vector2`和三维向量`Vector3`

```gdscript
# 二维向量由两个浮点数组成
var vec2 = Vector2(0.0, 0.0)
# 三维向量由三个浮点数组成
var vec3 = Vector3(0.0, 0.0, 0.0)
```

- 修改向量的项

```gdscript
var position = Vector3(1, 2, 3)
# 横坐标加2
position.x += 2
print(position)
# (3, 2, 3)
```

#### 4.3.4 字符串
- 我们可以通过`my_str.length()`获取字符串`my_str`的字符长度整数

### 4.4 常量的定义
- 使用关键字`const`来定义常量

```gdscript
const GRAVITY = -9.81
```

## 五、输入与输出

### 5.1 信息输出
- 运行程序后在底部控制台显示信息

```gdscript
print("It's MyGO!!!!!")
```

### 5.2 按键输入
- 在`Project Settings`面板中的`Input Map`处设置按键映射，我们新建一个例如名为`my_action`的动作，并将空格键`Space`绑定到该动作下

![GodotInputMap.png](/resources/2024-08-01-Godot的节点、信号概念与GDScript基础/GodotInputMap.png)

- 接下来我们在父节点`Main`的脚本下引出`_input(event)`函数用于处理按键输入，这个函数是Godot内置的函数

```gdscript
extends Node

func _ready():
	pass

func _process(delta):
	pass

func _input(event):
	pass
```

- 函数`_input(event)`在接收到任意输入的时候才会运行；其中`event`指代触发输入的事件

```gdscript
func _input(event):
	# 括号内引用我们设置好的按键映射名称，此处是"my_action"，即Space键
	if event.is_action_pressed("my_action")
		# 游戏运行时，按下空格键则会执行此处语句
	if event.is_action_released("my_action")
		# 松开空格键会执行的语句
```

## 六、数据容器

### 6.1 数组

#### 6.1.1 数组的创建
- 创建空数组

```gdscript
var items = []
```

- 在GDScript中可以在数组中混合数据类型，我们也可以限定数组元素为特定数据类型

```gdscript
# 普通数组，不限制元素的数据类型
var items = [3.14, "KON", 114514]
# 限定为字符串类型数组
var girls: Array[String] = ["Nina", "Bocchi"]
```

#### 6.1.2 数组的读写
- 通过索引来对数组元素进行读取或修改

```gdscript
# 读值
print(girls[1])
# Bocchi

# 写值
girls[0] = "Anon"
```

- 删除和添加数组元素

```gdscript
var crychic: Array[String] = ["Tomori", "Sakiko", "Muzimi", "Taki", "Soyo"]

# 删除第二个元素"Sakiko"
crychic.remove_at(1)

# 添加新元素"Anon"到数组末尾
crychic.append("Anon")
```

### 6.2 字典

#### 6.2.1 字典的创建
- 字典中的每个元素为一对`key: value`

```gdscript
var instruments = {
	"Guitar": 2,
	"Bass": 1,
	"Keyboard": 0
}
```

- 在GDScript的字典中，可以有多种类型的键值对
- 你甚至可以在字典中嵌套另一个数组或者字典，此时我们可以通过比如两个键来访问

```gdscript
var players = {
	"Steve": {"Health": 100, "Level": 1}.
	"Alice": {"Health": 80, "Level": 3}
}
print(players["Steve"]["Level"])
# 1
```

#### 6.2.2 字典的读写
- 通过字典的`key`索引来读出值的大小或者为某键重定义一个新的值

```gdscript
# 读值
print(instruments["Bass"])
# 1

# 写值
instruments["Bass"] = 0
```

- 为字典添加新条目

```gdscript
instruments["Violin"] = 1
```

#### 6.2.3 数组的循环遍历
- 用`for`与`in`关键字获取字典中的键

```gdscript
# 这里的name获取到的是字典instruments中的键
for name in instruments:
	print(name + ": " + str(instruments[name]))
```

### 6.3 枚举
- 枚举中用于定义一些特殊的标签，其好处之一在于可以在Inspector中进行标签的选取，很方便，这与我们在Unity中的枚举是同样的道理

```gdscript
# 定义了一组阵营标签
enum Alignment {
	ALLY,
	NEUTRAL,
	ENEMY
}

# 比如可以定义某个对象的成员变量中的阵营为敌人标签
@export var slime.alignment = Alignment.ENEMY
```

- 枚举的本质上是为其内元素设置了从0开始的索引值，比如我们`print()`枚举中的某个元素就会打印出来其编号；我们也可以在定义枚举的时候修改索引值，但是一般没必要

## 七、逻辑结构

### 7.1 条件语句的使用

#### 7.1.1 判断结构

```gdscript
if 判断语句1:
	执行语句1
elif 判断语句2:
	执行语句2
else:
	执行语句3
```

#### 7.1.2 比较运算符
- 等于/不等于：`==`/`!=`
- 大于/小于：`>`/`<`
- 大于等于/小于等于：`>=`/`<=`

#### 7.1.3 逻辑运算符
- 且：`and`，例如`x == y and y > z`
- 或：`or`，例如`x == y or y > z`

### 7.2 `match`语句的使用

- 类似C++中的`switch`，`case`和`default`语句，依据值是否匹配来有选择地执行语句
```gdscript
enum Alignment {
	ALLY,
	NEUTRAL,
	ENEMY
}

@export var alignment: Alignment = Alignment.ENEMY

func _ready():
	# 对变量alignment的值进行评估，对其不同值进行执行语句的选取
	match alignment:
		Alignment.ALLY:
			执行语句1
		Alignment.NEUTRAL:
			执行语句2
		Alignment.ENEMY:
			执行语句3
		_:
			默认情况下的执行语句
```

### 7.3 循环语句的使用

#### 7.3.1 `for`循环
- `for`循环执行指定次数

```gdscript
for n in 4
	print(n)
# 0
# 1
# 2
# 3
```

- `for`循环用于遍历数组

```gdscript
var crychic: Array[String] = ["Tomori", "Sakiko", "Muzimi", "Taki", "Soyo"]

# 遍历数组中的元素
for girl in crychic:
	# 将会逐行打印出数组中的字符串元素
	print(girl)
```

#### 7.3.2 `while`循环
- 一致循环运行直到条件达成才结束循环

```gdscript
while 循环条件:
	循环语句
```

#### 7.3.3 `break`与`continue`
- `break`关键字结束循环，执行循环后的语句
- `continue`结束当前一层循环，进入下一层循环，并非结束循环

## 八、函数

### 8.1 定义格式
- 可以不接收参数，可以没有`return`

```gdscript
func 函数名(参数1, 参数2, ...):
	执行语句
	return xxx
```

- 指定接收参数类型，也可以不指定

```gdscript
func 函数名(参数1: 数据类型, 参数2: 数据类型, ...):
	执行语句
	return xxx
```

### 8.2 系统给的可用函数
>对于系统提供的函数，我们可以通过`Ctrl + Click`这个函数名来跳转到函数的使用文档

#### 8.2.1 以下划线开头的函数
- 那些开头带下划线`_`的函数是不由我们激活或者调用的，是由引擎自身所调用

#### 8.2.2 随机数
- 函数`randf()`获取0~1间的随机浮点数

```gdscript
func _ready():
	var random_value = ranf()
	if random_value < 0.2:
		print("rare item")
	else:
		print("general item")
```

- 函数`randi_range(min, max)`来获取范围内的随机整数

```gdscript
var height = randi_range(150, 200)
```

#### 8.2.3 `Set()`与`Get()`函数
>这两个函数能够使我们在变量改变或者变量被读取时添加执行代码，比如可以让我们能对被修改的变量被赋的值按我们的需求而不一定完全等于传入的值、对被读取的变量被读到的值不一定等于其真实值，而是读取到一个被加工过后的值，又比如在被修改或读取的同时传递出信号等

##### 8.2.3.1 `Set()`函数
- 对于`Set()`函数的使用，生命值的变化是一个很棒的例子，这基于我们需要将生命值限制在合法的范围区间内的需求

```gdscript
# 创建一个信号表示生命值被改变了，并将新的生命值作为参数传递
signal health_changed(new_health)

# 默认生命值为最大值100
# 在该变量定义完成后添加一个冒号，并设置一个setter
var health := 100:
	# set()函数括号内的值我们来命名，用于指代health被试图更改为的传入值
	set(vallue):
		# health被修改时，即接收到了传入值value后需要执行的语句
		# 将health赋值为对value的clamp，使得生命值最小不低于0，最大不超过100
		health = clamp(value, 0, 100)
		# 传递生命值变化的信号，并将当前生命值作为参数传递出去
		health_changed(health)

# 为了方便我们就将这个信号的链接处放在自身脚本上来监听信号的传递结果
func _on_health_changed(new_health):
	print(new_health)

func _ready():
	# 给health进行赋值，这将会触发setter部分的函数
	health = -10

# 得到的输出结果为0，因为-10超出了clamp限制的范围，则自动取限制范围的最小值
```

##### 8.2.3.2 `Get()`函数
- 对于`Get()`函数的使用，以百分比（如暴击率、闪避率）意义存储的变量的读取是一个很棒的例子，我们可以让存储的xx率为浮点型的值，被读取时乘上100而读取整型类型的值

```gdscript
# 某种几率，小数形式；此语境下chance变量身处幕后，不被直接读写
var chance := 0.2
# 这种几率的百分号前的整数部分；此语境下chance_pct变量才是真正被读写的对象
var chance_pct: int:
	# 该变量被读取时执行的语句，最终需要return一个真正被读取的值
	get:
		return chance*100
	# 该变量被修改时也要同步修改chance
	set(value):
		# 为了保证chance为浮点数，记得类型转换，以及除数也需要是浮点数
		chance = float(value) / 100.0

func _ready():
	# 此处只是一个该变量被读取的其中一种形式，其他的形式还可能为被调用去为别的变量赋值等
	print(chance_pct)
	chance_pct = 60
	print(chance)

# 20
# 0.6
```

## 九、类与对象

### 9.1 节点类的创建
- 我们新建一个名为`Player`的`Node`类型的节点，为之创建一个脚本，然后我们就可以以构筑类的思想来构筑这个脚本，每个脚本所在的节点即为该类实例化的对象（从这个角度看，带脚本的节点就像Unity中带脚本的GameObject，而Unity脚本的编写就是一个类的编写，这里也类似）
- 我们在创建一个所谓`Player`类的时候，本质上是在定义一个新的节点类型

```gdscript
# 类名称
class_name Player
extends Node

# 成员变量，可以通过该节点的引用来调用
@export var profession: String # 玩家职业，在编辑器中手动设置
@export var health: int # 玩家生命值

# 成员函数，可以通过该节点的引用来调用
func die():
	health = 0
	print(profession + " died")
```

### 9.2 内部类的创建
- 我们在节点的内部还可以创建内部类（并不一定是父子关系哦），如玩家的装备类

```gdscript
# 类名称
class_name Player
extends Node

# Player类的成员变量中可以使用new()函数实例化装备类的一个对象
var chest := Equipment.new() # 胸甲
var legs := Equipment.new() # 腿甲

# 我们可以访问这些变量所属类的一些属性
func _ready():
	chest.armor = 20 # 更改胸甲护甲值

# 创建一个玩家装备的内部类
class Equipment:
	var weight := 5 # 装备重量
	var armor := 10 # 装备护甲值
```

### 9.3 继承
- 我们所有节点类都是基类`Node`的子类，我们通过`extends`来声明了这个继承关系，意味着我们在节点类中可以调用`Node`类中的变量和函数

```gdscript
extends Node
```

- 在我们创建一个Godot提供的类型的节点的时候，他会自动继承自其正确的对应父类节点，即在脚本中以`extends base_class`的方式继承；我们撰写脚本的时候需要注意处理好这些继承关系