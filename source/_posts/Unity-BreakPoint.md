title: "Unity断点调试"
date: 2015-03-19 18:54:39
categories: Unity3D
tags: Debug
---

有时候调试程序，断点信息可以很方便定位问题，Unity支持使用自带的MonoDevelop进行断点调试，但这个功能做的很不友好，经常Attach进程无响应。这里就记录一个成功率比较高的方法，避免浪费时间。

1. 打开Unity、MonoDevelop

2. 在Unity中，Assets -> Sync MonoDevelop Project

3. 在MonoDevelop中，Run -> Attach to Process...

选择Unity Editor (Unity)，点击Attach按钮  

![](/images/201503191901.png)

这个经测试，不是每次都成功:（，如果无响应，那么请cmd+Q退出Unity和MonoDevelop，大侠请回到step1重新来过。