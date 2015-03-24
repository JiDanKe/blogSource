title: "Animator.Play的一个bug"
date: 2015-03-24 11:43:57
categories: Unity3D
tags: Animator
---
Animator.Play方法，官方文档解释的很简单：“Plays a state”，但是实际使用中会有个问题。
如下，在同一帧里连续两次调用Play播放不同的动画片段：

```
animator.Play("animation1");  
animator.Play("animation2");
```

你猜最后播放的是哪个动画？
我猜你猜的是animation2，兄弟我们都被坑了:P

结果是animation1被播放，这有点反人类，应该是个bug。我使用的版本是Unity3D 5.0.0p1

照这个结果，很多动画播放逻辑需要注意了。比如我们自己有个状态机，在这个状态机里调用Animator.Play来控制动画播放。如果在一帧内有多个状态切换，则可能会先后面状态的动画没有播放的问题。举个栗子：  

```
Run -> Idle  
```

如果状态切换太快，则可能在Idle状态，却一直播放Run的动画。

因为这个原因，状态切换最好放到下一帧。我们为状态机切换提供一个请求切换状态的接口，状态内部不能直接切换到下一个状态。

```
//设置下一帧想要切换的状态
public void SetWishingState(State state);

//在帧的开始，切换到下一个状态
private void TranslateToWishingState();

```