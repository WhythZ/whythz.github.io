---
# author:
title: 详解UGUI中RectTransform的轴心、锚点、尺寸、位置
description: >-
  因为这块内容做UI时比较常用，故研究无限滚动列表时顺便总结了一下，以免忘记
date: 2025-10-18 01:51:00 +0800
categories: [游戏开发, 玩法相关]
tags: [Unity, U3D, UGUI]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
math: true
# mermaid: true
# image:
#   path: /resources/xxxxxx.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

## 一、轴心
- 轴心`RectTransform.pivot`用于定义UI元素**自身旋转缩放**的参照点
    - 默认为`Vector2(0.5f,0.5f)`即表示轴心位于UI元素的正中心
    - 若取值`Vector2(0.6f,0.6f)`则表示轴心位于UI元素的$60\%$宽（从左往右）高（从下往上）点位（靠近右上角）

![RectTransform轴心示意.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/RectTransform轴心示意.png)

## 二、锚点

### 2.1 基本定义
- 锚点用于定义UI元素**相对于父容器**的位置与拉伸关系，`RectTransform.anchorMin`和`RectTransform.anchorMax`分别表示子UI对象相对父UI对象的**左下角和右上角锚点**，其`x`和`y`的取值均在$[0,1]$范围内，**这两点定义了一个锚框**，UI元素会相对于该区域进行定位和大小计算
    - 若两者均为`Vector2(0.5f,0.5f)`而重合，**则UI元素的Pivot会对齐在父容器的中心点**
        - 类似地若两者均为`Vector2(0,0)`而重合，则UI元素的Pivot会对齐在父容器的左下顶点
        - 类似地若两者均为`Vector2(1,1)`而重合，则UI元素的Pivot会对齐在父容器的右上顶点
        - 类似地若两者均为`Vector2(0,1)`而重合，则UI元素的Pivot会对齐在父容器的左上顶点
        - 类似地若两者均为`Vector2(1,0)`而重合，则UI元素的Pivot会对齐在父容器的右上顶点
    - 若`anchorMin`为`Vector2(0,0.5f)`即左中、`anchorMax`为`Vector2(1,0.5)`即右中（即锚点水平方向分开、垂直方向重合），则该UI元素会**水平拉伸适应**父UI而垂直方向保持自由
        - 若此时将二者的`y`改为`1`则该UI元素垂直方向上会向上侧对齐
        - 若此时将二者的`y`改为`0`则该UI元素垂直方向上会向下侧对齐
    - 若`anchorMin`为`Vector2(0.5f,0)`即中下、`anchorMax`为`Vector2(0.5f,1)`即中上（即锚点垂直方向分开、水平方向重合），则该UI元素会**垂直拉伸适应**父UI而水平方向保持自由
        - 若此时将二者的`x`改为`1`则该UI元素水平方向上会向右侧对齐
        - 若此时将二者的`x`改为`0`则该UI元素水平方向上会向左侧对齐
    - 若`anchorMin`为`Vector2(0,0)`即左下、`anchorMax`为`Vector2(1,1)`即右上，则该UI元素会**完全拉伸适应**父UI的宽高达到四边贴合的效果
 - **在调整锚点的同时调整轴心**，可更精准地调整子对象相对父对象的位置和尺寸关系
    - 若子对象两锚点相重合，体现为唯一锚点
        - 此时子对象的尺寸不会随着父对象的尺寸变化而变化，子对象`RectTransform`组件中的属性是`PosX/PosY/Width/Height`
        - 此时无论父对象尺寸如何变化，子对象`pivot`与该唯一锚点间的距离会保持不变，相当于子对象会参照锚点来实时调整自己的位置
    - 若子对象两锚点不重合，体现为锚框区域
        - 此时子对象的大小会随着父对象的大小变化而拉伸变化，子对象`RectTransform`组件中的属性是`Left/Top/Right/Bottom`
        - 此时无论父对象尺寸如何变化，保持不变的不再是子对象`pivot`与谁之间的距离，而是锚框左下角到子对象左下角的向量`(Left,Bottom)`以及子对象右上角到锚框右上角的向量`(Right,Top)`

![RectTransform锚点不重合时的Left等属性含义.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/RectTransform锚点不重合时的Left等属性含义.png)

### 2.2 锚框预设
- 同时Unity也提供了一系列常用的预设（按住`Shift`键后再应用预设会同步将轴心`pivot`也应用到符合直觉的位置，按住`Alt`键后再应用预设会立刻应用锚框引发的位置或尺寸变化以便查看）

![RectTransform锚点预设.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/RectTransform锚点预设.png)

## 三、尺寸

### 3.1 关系公式
- `RectTransform.sizeDelta`的`x/y`表示UI元素相对于锚框的参考尺寸，存在关系`sizeDelta = offsetMax - offsetMin`
    - `RectTransform.offsetMin`指的是`anchorMin`到子对象左下角的向量
    - `RectTransform.offsetMax`指的是`anchorMax`到子对象右上角的向量

![RectTransform的sizeDelta解释.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/RectTransform的sizeDelta解释.png)

- `RectTransform.rect.width/height`则表示UI元素实际的世界空间尺寸，其与`sizeDelta`间的关系如下（注意两个锚点`x`或`y`间的差值范围同样在`[0,1]`之间，而）
    - 当两锚点相重合时，`rect.width/height`和`sizeDelta.x/y`恰巧相等
    - 当两锚点不重合时，`sizeDelta.x/y`代表**子对象尺寸`rect.width/height`与锚框尺寸间的差值**

```
sizeDelta.x = rect.width - 父对象宽度 * (anchorMax.x - anchorMin.x)
sizeDelta.y = rect.height - 父对象高度 * (anchorMax.y - anchorMin.y)
```

### 3.2 代码验证
- 为了验证上述公式，我编写了以下脚本挂载在如下父对象（一个黑色正方形`Image`）上，该父对象有一个名为`Child`的子对象（一个红色正方形`Image`），我们调整子对象的锚框预设然后观察输出日志

![测试RectTransform的sizeDelta与rect间关系P1.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/测试RectTransform的sizeDelta与rect间关系P1.png)

```cs
using UnityEngine;

public class Printer : MonoBehaviour
{
    void Update()
    {
        //父对象
        Rect _parentRect = GetComponent<RectTransform>().rect;

        //子对象
        Vector2 _childSizeDelta = transform.Find("Child").GetComponent<RectTransform>().sizeDelta;
        Rect _childRect = transform.Find("Child").GetComponent<RectTransform>().rect;
        Vector2 _childAnchorMin = transform.Find("Child").GetComponent<RectTransform>().anchorMin;
        Vector2 _childAnchorMax = transform.Find("Child").GetComponent<RectTransform>().anchorMax;

        //尝试验证关系
        float _proposedWidth = _childSizeDelta.x + (_childAnchorMax.x - _childAnchorMin.x) * _parentRect.width;
        float _proposedHeight = _childSizeDelta.y + (_childAnchorMax.y - _childAnchorMin.y) * _parentRect.height;

        Debug.LogError("子对象的rect宽高=(" + _childRect.width + "," + _childRect.height + ")");
        Debug.LogError("公式计算所得宽高=(" + _proposedWidth + "," + _proposedHeight + ")");
    }
}
```

- 观察如下三种情况（一种锚点重合在右边框中心的情况，两种锚点分开的情况）的输出，公式的确是正确的

![测试RectTransform的sizeDelta与rect间关系P2.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/测试RectTransform的sizeDelta与rect间关系P2.png)

![测试RectTransform的sizeDelta与rect间关系P3.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/测试RectTransform的sizeDelta与rect间关系P3.png)

![测试RectTransform的sizeDelta与rect间关系P4.png](/resources/2025-10-18-详解UGUI中RectTransform的轴心、锚点、尺寸、位置/测试RectTransform的sizeDelta与rect间关系P4.png)

## 四、位置
- `RectTransform.anchoredPosition`用于控制子对象相对父对象的位置
    - 当两锚点相重合时，`anchoredPosition`表示锚点到`pivot`轴心的向量，修改此值可直观调整相对位置
    - 当两锚点不重合时，需分类讨论，例如当`anchorMin=(0,1)`且`anchorMax=(1,1)`时，子对象水平拉伸适应，此时`anchoredPosition.x`表示子对象`pivot`相对锚框上边界中垂线的水平偏移距离，`anchoredPosition.y`表示从锚框上边界到子对象`pivot`的垂直偏移距离（此时当修改`anchoredPosition.x`例如为`testVal`时，由于其不会显示在Hierarchy中，此时观察可发现`Left`值会对应变为`testVal`，而`Right`的值会变为`-testVal`，可发现水平宽度没发生变化；此时若手动修改二者的值为绝对值不相等的两个值，则宽度会被缩放，`anchoredPosition.x`也会相应地变化）