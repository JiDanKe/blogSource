title: "Unity断点调试"
date: 2015-03-19 18:54:39
categories: Unity3D
tags: Debug
---

Unity能使用自带的MonoDevelop进行断点调试，但这个功能做的很不友好，经常Attach进程无响应。有时候断点信息可以很方便定位问题，这里就记录一个方法，可以小心翼翼的“避开雷区”，成功连接Unity和MonoDevelop进行断点调试。

1. 打开Unity、MonoDevelop

2. 在Unity中，Assets -> Sync MonoDevelop Project

3. 在MonoDevelop中，Run -> Attach to Process...

选择Unity Editor (Unity)，点击Attach按钮  

![](/images/201503191901.png)

这个经测试，不是每次都成功:（，如果无响应，那么请cmd+Q退出Unity和MonoDevelop，大侠请回到step1重新来过。