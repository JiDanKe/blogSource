title: "Unity 3D Canvas Scaler 解析"
date: 2015-03-30 19:31:39
categories: Unity3D 
tags: UI
---

UGUI(Unity3D 原生 UI 组件) 的屏幕分辨率自适应是通过在 `Canvas` 组件上添加 Canvas Scaler 组件(5.0 之前的这个组件的功能是由 Reference Resolution 和 Physical Resolution 这两个组件来同完成的)来实现的。这个组件的属性将会决定在这个 `Canvas` 下所有 UI 的缩放规则。`Canvas Scaler` 的缩放模式分为三种，接下来就分别来介绍一下这几种缩放模式，以便我们可以去更好地使用它们。

## Constant Pixel Size

在这种模式下 UI 的大小（以像素为单位）将不会随着屏幕尺寸的改变而改变。在这种模式下可以调节的属性有：

1. `Scale Factor`。调节它可以对当前 Canvas 下的所有 UI 进行缩放，这个值变大 UI 也会相应地变大。这个值是需要手动调节的，它并不会跟着屏幕大小的变化而变化。

2. `Reference Pixels Per Unit`。该属性就是一个 Unity 单位所能容纳的像素的个数，这个值越小 UI 将会越大。


## Scale With Screen Size

`Scale With Screen Size` 模式中 UI 的大小会随着屏幕的大小而变化。在这个模式下首先要设置的时 `Reference Resolution` 属性，该属通常与 UI 的设计尺寸一直，比如：有一组 UI 为 1024 X 768 的屏幕设计的 UI，那么 `Reference Resolution` 可以设成这个值，这要就可以在游戏中更好的还原设计。这个模式下 UI 缩放的程度是由它所在的 `Canvas` 的缩放程度所决定的，如果 `Canvas` 缩放的屏幕的分辩率的宽高比是和 `Reference Resoltion` 中设置的分辨的值宽高比一至，那么 `Canvas` 里的 UI 也和 `Canvas` 进行同样比例的缩放即可。

举例来说：`Canvas` 的 `Reference Resolution` 为 600 X 400，在这个 `Canvas`中有一个大小为 100 X 20 的 `Button`，当这个 `Canvas` 适配到 1200 X 800 屏幕时，宽高都要增大一倍，这时我们说 `Canvas` 大小增大了一倍，与此相对应 `Canvas` 中 `Button` 也要增大一倍，即扩大到 200 X 40 大小，这种缩放 `Canvas` 和 `UI` 的相对大小是一样的，我们可以把它叫做等比缩放。

但是如果 `Canvas` 需要缩放的屏幕分辨的长宽比和 `Reference Resolution` 不一样，那么缩放应该怎么进行？首先有一点可以确定那就是 `Canvas` 会铺满屏幕，需要考虑是如何缩放 `Canvas` 中的 UI 了，如何对 UI 进行缩放就是由 `Screen Match Mode` 属性来决定了。`Screen Match Mode` 有三个模式，下面将具体说明一下这几种模式。

### Match Width or Height
首先我们来看 `Match Width or Height`。在这一行为模式下 `Canvas` 中 UI 的大小缩放比例是参考 `Canvas` 中高度、宽度的缩放比例来进行缩放，而缩放比例的计算则由是 
`Match` 参数决定，这个参数一个在 0 与 1 的之间值。还是用举例的方式来说明这个值是怎么作用于 UI 的缩放的。

1. 值为0。在这种情况下，UI 缩放比例完全参考宽度的缩放比例。例如，一个 `Reference Resoltion` 为 600 X 400 的 `Canvas` 适配到 1200 X 500 的分辨率， 宽度增大了一倍，那么 `Canvas` 中的 UI 大小也要增大一倍，原来大小为 100 X 20 的 `Button` 就要变成 200 X 400 了。

2. 值为1. 在这种情况下，UI 缩放比例完全参考高度的缩放比例。例如，一个 `Reference Resoltion` 为 600 X 400 的 `Canvas` 适配到 600 X 800 的分辨率， 高度增大了一倍，那么 `Canvas` 中的 UI 大小也要增大一倍，原来大小为 100 X 20 的 `Button` 就要也变成 200 X 400。

3. 值为0到1之间的值. 在这种情况下，UI 缩放比例完要同时参考高度和宽度的缩放比例。例如， `Match` 值为0.4，一个 `Reference Resoltion` 为 600 X 400 的 `Canvas` 适配到 1800 X 800 的分辨率， 高度变为原来的2倍，而宽度则变为原来的3倍，那么 `Canvas` 中的 UI 大小就要变成原来的 （3 X 0.6 + 2 X 0.4 ）= 2.6 倍，原来大小为 100 X 20 的 `Button` 就要也变成 260 X 52。

### Expand
然后是 `Expand` 模式。在这个模式中 `Canvas` 的大小会横向（Horizontally）或纵向（Vertical）的放大以已适应屏幕的变化，使 `Canvas` 的显示分辨率不会小于设定的 `Reference Resolution` 值，所以在这种模式之下所有 UI 都能显示完全，只不过比例相对 `Canvas` 的比例变小了而已。

例如：一个 `Reference Resoltion` 为 600 X 400 的 `Canvas` 适配到 `600 X 200` 的分辨率，那么 `Canvas` 的显示分辨率将会变成 1800 X 400，在这种那个情况下 UI 大小不变，只不过在视觉上变小了而已。

### Shrink
最后是 `Shrink` 模式。 该模式中 `Canvas` 当屏幕的比例发生变化时会横向或纵向的去裁剪自己的大小，使得自己的大小不会大于之前设置的 `Reference Resolution` 值 。从表现上来看就会发现有的处于屏幕边缘或者尺寸较大的 UI 会变得显示不完全，其原因就是 `Canvas` 的显示分辨率变小了。

例如：一个 `Reference Resoltion` 为 600 X 400 的 `Canvas` 适配到 500 X 200 的分辨率，那么 `Canvas` 的显示分辨率将会变为 500 X 200 ，如之前在 `Canvas` 中有一张大小为 600 X 400 的图片，那么在这种情况下，图片就不能显示完全了。

### 总结
`Match Width or Height` 无论是适配到比 `Reference Resolution` 高还是低的分辨率时，`Canvas` 和它里面的 UI 都会进行一定比例的缩放，而 `Expand` 和 `Shrink` 在适配高于 `Reference Resolution` 的分辨率的屏幕时会同时缩放 `Cavans` 和 它里面的 UI，而在适配高于 `Reference Resolution` 的分辨率的屏幕时只是改变 `Canvas` 的显示大小而不改变 UI 的大小。 

## Constant Physical Size

这种模式下 UI 的物理大小（通常的单位是： millimeters、points、 或者 picas）将会保持不变，而显示的正确与否则取决于设备能否准确获取屏幕的 DPI。
