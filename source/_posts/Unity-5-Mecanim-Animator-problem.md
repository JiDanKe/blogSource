title: "Unity 5动画系统问题"
date: 2015-03-24 10:12:09
categories: Unity3D
tags: Animator
---
### 1.如果只有一帧，第一帧放置event基本上不会触发，event放的第二帧即可
#### 不会触发
![不会触发](/images/c5868628369eb7755c4b133bed49a4b9e6428b03.jpg)

#### 触发
![触发](/images/d068f3cb1017fc8e3ed816b9f57dbf1f3f0bbb69.jpg)

### 2. 如果最后是暂停动画的控制，需要延后至少一帧才有保障，不然动画不一定播放完成

![延后一帧](/images/4729e5bafe4aec8d0563f06e97eb4df2efaebdaa.jpg)
