title: "Change size of the new Unity UI RectTransform"
date: 2015-08-12 17:50:57
categories:
tags:
---

我们在开发过程中发现，要调整Unity UI元素的大小，RectTransform提供了sizeDelta属性可以用来动态修改RectTransform的大小，但同时我们也google到另外一个修改RectTransform大小的方法，方法如下：
``` csharp
    public static void SetRectTransformSize(RectTransform trans, Vector2 newSize)
    {
        Vector2 oldSize = trans.rect.size;
        Vector2 deltaSize = newSize - oldSize;
        trans.offsetMin = trans.offsetMin - new Vector2(deltaSize.x * trans.pivot.x, deltaSize.y * trans.pivot.y);
        trans.offsetMax = trans.offsetMax + new Vector2(deltaSize.x * (1f - trans.pivot.x), deltaSize.y * (1f - trans.pivot.y));
    }
```

那RectTransform.sizeDelta和这个自定义的SetRectTransformSize有没有差别？差别在哪儿？我们做了下实验来测试：
新建一个场景，放入四个按钮
![](/images/1c74b2bca5f5c0036b0378fcd32e344e5c566cfe)

其中deltaSize和resize按钮的Anchors四个点重合，deltaSizeWithCustomAnc和resizeWithCustomAnc按钮的四个点不重合
![](/images/5d9d8a46a44dc8fb4ee9c19834300b129922d7f3)

![](/images/84092fd5bf56e4fa4b9cfbbe454b41180cacf693)

然后挂载一个脚本，在点击change按钮的时候修改上边四个按钮的大小：
``` csharp
using UnityEngine;
using System.Collections;
using UnityEngine.UI;

public class ResizeTestController : MonoBehaviour {

    public RectTransform deltaSizeBtn;
    public RectTransform resizeBtn;
    public RectTransform deltaSizeDiffAncBtn;
    public RectTransform resizeDiffAncBtn;

    Vector2 newSize =new Vector2(300f,50f);

    public void Change(){
        //Anchors在同一点的button
        deltaSizeBtn.sizeDelta = newSize;
        U3DUtils.SetRectTransformSize(resizeBtn, newSize);

        //Anchors不在同一点的button
        deltaSizeDiffAncBtn.sizeDelta = newSize;
        U3DUtils.SetRectTransformSize(resizeDiffAncBtn, newSize);
    }
}
```

点击Change按钮后：
![](/images/9bd9957157cb1c2e129b7d742347e66f3011ca3f)

我们可以看到按钮deltaSizeWithCustomAnc的大小与众不同，说明sizeDelta属性的计算过程跟SetRectTransformSize是不一样的，Unity官方文档对sizeDelta的说明：
[RectTransform sizeDelta](http://docs.unity3d.com/462/Documentation/ScriptReference/RectTransform-sizeDelta.html)
大概意思是Anchors在同一点的时候，sizeDelta相当于设置长宽，但是Anchors不在同一点时，表示的只是比Anchors矩形大或者小多少。

那么sizeDelta具体又是怎么转换到trans.offsetMin和trans.offsetMax的呢？
![](/images/a372fb5139afc590951dc9ff6c6d38496b1cba11)
如上图，left  + right刚好是我们设置的300的-1倍：-300, top + bottom也是刚好-50（负数的意思是比Anchors矩形大；正数就是Anchors矩形内，就是比Anchros矩形小）。
那可以猜测（看源代码没找到sizeDelta最终计算的地方，😓）sizeDelta的计算方法是与anchors有关的，当然还与Pivot有关（可以自己测试下看看），写个模拟方法来验证下：

``` csharp
    public static void SetRectTransformDeltaSize(RectTransform trans, Vector2 newSize)
    {
        var x = -newSize.x - trans.offsetMin.x + trans.offsetMax.x;
        var y = -newSize.y - trans.offsetMin.y + trans.offsetMax.y;

        trans.offsetMin = new Vector2( trans.offsetMin.x + trans.pivot.x * x, trans.offsetMin.y + trans.pivot.y * y);
        trans.offsetMax = new Vector2( -(-trans.offsetMax.x + (1-trans.pivot.x) * x),-(-trans.offsetMax.y + (1-trans.pivot.y) * y));
    }

```
直接看可能不容易明白是什么意思，不过根据Unity的文档我们知道sizeDelta表示的是RectTransformd对象矩形大小比Pivot矩形大或者小多少，那么RectTransform的Left和Right之和就等于sizeDelta.x的-1倍，再考虑pivot，可以得到一个计算公式:
```
(Left + pivot.x *X) + (Right + (1-pivot.x) * X) =  - sizeDelta.x 
```
其中X是要求出的变量，转换下公式就是：
```
X =  - sizeDelta.x - Left - Right
```
由于
```
Left == trans.offsetMin.x
Right == - trans.offsetMax.x
```
带入后
```
X =  - sizeDelta.x - trans.offsetMin.x - (- trans.offsetMax.x)
```
那么新的Left就是
Left + pivot.x *X
新的Right就是
Right + (1-pivot.x) * X
Right转回offsetMax.xx需要*-1
