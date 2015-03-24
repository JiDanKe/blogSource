title: "Unity 5 过渡控制变化"
date: 2015-03-24 10:12:09
categories: Unity3D
tags: Animator
---
# 过渡控制变换
Unity 5默认给状态机连线生成的过渡时间是不一定的，跟动画时间有关，但是似乎都不是在当前状态机结束再切换到下一个状态，自动生成的过度时间都有比较严重的交叠，或者某个状态机重复多次才进入下一个状态。  
可以使用如下图所示的reset 重置到一个Unity5默认的过渡时间设置。   

![Paste_Image.png](/images/6ffc27e54a23f87b096dbce889ff8403ac8ade17.png)  

reset得到的应该是0.9和0.1，也就是在当前状态播放到90%的时候开始同时播放下一个状态机。而这个默认值在Unity 4.x里边是 1.0和0，也就是默认不会在两个状态间留0.1的过渡时间。如果在Unity 5里边改成1.0和0 也会有问题，如果当前状态最后一帧有event，经常会发生这个event不被调用。

![Paste_Image.png](/images/f06c804ce2654d65ac85041ba793e3a89561cc47.png)
