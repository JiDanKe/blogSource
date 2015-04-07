title: "Hero项目Collider使用梳理"
date: 2015-03-25 11:57:51
categories: Unity3D
tags: 规范
---
渐渐发现随着demo的完善，Collider也越来越多，功能各不相同，一时难以弄清。故此写一篇文章梳理目前我们Collider的使用情况，备忘。

###角色Collider

![](/images/d85b01be221e01d51d8aae9ff901ff212dc74433)

Player对象里的Collider用来表示人物占用的体积，包含意义如下：
人物移动时，障碍物碰撞处理，避免人物走到障碍物里面的基本保证。在SimpleCharacterController2D中定义作为障碍物的Collider在哪个Layer。

![](/images/e85a67d8e5411ac7c40f60fc864637daf959e7d6)

Player有个子对象TouchResponse，挂载了一个Collider组件，这个Collider用来作为接受用户点击区域。比如攻击某个敌人，识别点击这个敌人的操作。这里的规则为Tag:Touchable，即获取Tag为Touchable的对象，再从中获取relatedObject。
Player -> Tag:Player/Enemy, Layer:Roles
TouchResponse -> Tag:Touchable Layer:Default

###墙、物体、动态障碍的表示
墙使用Layer:Wall来标识
物体(如宝箱)用Layer:Objects来标识
动态障碍依赖于寻路组件，需要加入DynamicObstacle组件，并且用Layer:Wall/Objects来标识。
Wall、Objects由寻路组件定义，如下图

![](/images/c77bd52ebf3e3a13a2b0352cb7457bd728a8f0a8)

###Tips
1. 在 Unity 3D 中 Collider 分为两大类：一类是普通 Collider，这一类可以在 3D 或者 2D 环境中使用; 另一类为 Collider 2D，一般在 2D 游戏中使用。与此对应的物理系统也分为 Physics 和 Physics 2D 两类。在进行射线（Ray）检测 Collider 碰撞时要注意 Collider 一定要和相应的物理系统匹配，比如：Collider 2D 要用 Physics 2D 发出的射线才能检测到碰撞。如果发现射线检测不到 Collider 可以检查一下 Collider 是否与物理系统相匹配。

如果还有其他Collider的用途，望大家补充。