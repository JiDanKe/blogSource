title: " differenceBetweenCircleCastandOverlapCircle"
date: 2015-03-20 14:58:00
categories:
tags:LinecastAll Physics2D 技能
---
今天重构了下玩家的技能攻击实现类，把之前攻击时直接获取场景中的所有敌人再判读是否在攻击范围内来过滤改为直接获取技能攻击半径内的地方对象。由于检测两点之间是否有障碍物的时候用了：
```csharp
RaycastHit2D[] rayHits = Physics2D.LinecastAll (me.position, target.position, mask);
```
最开始理所当然就认为&nbsp;Physics2D.CircleCastAll 应该就是检测圆形范围内的碰撞器，以至于最开始看Physics2D.CircleCastAll的参数很不理解。事实上我需要的是Physics2D.OverlapCircleAll 。
### 一、获取在一个圆的半径范围内的collider
```csharp

Collider2D [] colliders = Physics2D.OverlapCircleAll(player.position, 1f, enemyLayer);
 if(colliders.Length > 0)
 {
       // enemies within 1m of the player
 }
 
```
### 二、 获取以一个点为中心的圆朝某个方向移动一定距离碰撞到的layer上的collider
```csharp
 RaycastHit2D[] hits=  Physics2D.CircleCastAll(player.position, 1f, player.right, 10f, enemyLayer);
 foreach(RaycastHit2D hit in hits)
 {![alt text][1]
     ApplyDamage(hit.collider.gameObject);
 }
```
### 看图说话
![CircleCastAll and OverlapCircleAll](/images/24166795-E46D-45D3-A5A4-F7A75C266F50.jpg)