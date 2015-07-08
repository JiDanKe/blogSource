title: "uGUI之AutoLayout详解——minHeight,preferredHeight,flexibleHeight"
date: 2015-07-07 20:39:12
categories: Unity3D
tags: AutoLayout
---

首先看一个例子，新建一个Panel，在下面添加两个Button，分别命名为Button、Button2。
1、给Panel添加一个VerticalLayoutGroup组件，ChildForceExpand属性中勾上Width。
2、给Button、Button2添加LayoutElement组件，其中Button的FlexibleHeight设置为0.3，Button2的FlexibleHeight设置为0.1
3、将Panel的高度设置为100

![1-1](/images/cc7b7acff95f50dacd518d7fc8858ce81361114b.png)


![1-2](/images/fb0e7991d0ed8d5cb82b8f80489162b29ddbb225.png)

这时我们发现，Button的高度是70，Button2的高度是30。奇怪，这个高度是怎么算出来的呢？

网上搜索一番，竟然很少有人讨论uGUI的AutoLayout，尤其是flexibleWidth/Height属性的意义，官方文档也语焉不详。这时只能放大招了，uGUI已经开源，索性把代码拉下来看看到底怎么实现的。下面是托管代码的地址：
[https://bitbucket.org/Unity-Technologies/ui](https://bitbucket.org/Unity-Technologies/ui)

uGUI的AutoLayout有三个核心接口，定义在ILayoutElement.cs文件中：
ILayoutElement
ILayoutController
ILayoutIgnorer

结构很清晰，由ILayoutElement提供布局信息，ILayoutController来控制布局，ILayoutIgnore提供给UI忽略AutoLayout的能力。

例子中使用的VerticalLayoutGroup继承自HorizontalOrVerticalLayoutGroup，这个类实现了布局的核心逻辑，代码量不多，我就直接贴上来了
```
protected void CalcAlongAxis(int axis, bool isVertical)
{
    float combinedPadding = (axis == 0 ? padding.horizontal : padding.vertical);

    float totalMin = combinedPadding;
    float totalPreferred = combinedPadding;
    float totalFlexible = 0;

    bool alongOtherAxis = (isVertical ^ (axis == 1));
    for (int i = 0; i < rectChildren.Count; i++)
    {
        RectTransform child = rectChildren[i];
        float min = LayoutUtility.GetMinSize(child, axis);
        float preferred = LayoutUtility.GetPreferredSize(child, axis);
        float flexible = LayoutUtility.GetFlexibleSize(child, axis);
        if ((axis == 0 ? childForceExpandWidth : childForceExpandHeight))
            flexible = Mathf.Max(flexible, 1);

        if (alongOtherAxis)
        {
            totalMin = Mathf.Max(min + combinedPadding, totalMin);
            totalPreferred = Mathf.Max(preferred + combinedPadding, totalPreferred);
            totalFlexible = Mathf.Max(flexible, totalFlexible);
        }
        else
        {
            totalMin += min + spacing;
            totalPreferred += preferred + spacing;

            // Increment flexible size with element's flexible size.
            totalFlexible += flexible;
        }
    }

    if (!alongOtherAxis && rectChildren.Count > 0)
    {
        totalMin -= spacing;
        totalPreferred -= spacing;
    }
    totalPreferred = Mathf.Max(totalMin, totalPreferred);
    SetLayoutInputForAxis(totalMin, totalPreferred, totalFlexible, axis);
}

protected void SetChildrenAlongAxis(int axis, bool isVertical)
{
    float size = rectTransform.rect.size[axis];

    bool alongOtherAxis = (isVertical ^ (axis == 1));
    if (alongOtherAxis)
    {
        float innerSize = size - (axis == 0 ? padding.horizontal : padding.vertical);
        for (int i = 0; i < rectChildren.Count; i++)
        {
            RectTransform child = rectChildren[i];
            float min = LayoutUtility.GetMinSize(child, axis);
            float preferred = LayoutUtility.GetPreferredSize(child, axis);
            float flexible = LayoutUtility.GetFlexibleSize(child, axis);
            if ((axis == 0 ? childForceExpandWidth : childForceExpandHeight))
                flexible = Mathf.Max(flexible, 1);

            float requiredSpace = Mathf.Clamp(innerSize, min, flexible > 0 ? size : preferred);
            float startOffset = GetStartOffset(axis, requiredSpace);
            SetChildAlongAxis(child, axis, startOffset, requiredSpace);
        }
    }
    else
    {
        float pos = (axis == 0 ? padding.left : padding.top);
        if (GetTotalFlexibleSize(axis) == 0 && GetTotalPreferredSize(axis) < size)
            pos = GetStartOffset(axis, GetTotalPreferredSize(axis) - (axis == 0 ? padding.horizontal : padding.vertical));

        float minMaxLerp = 0;
        if (GetTotalMinSize(axis) != GetTotalPreferredSize(axis))
            minMaxLerp = Mathf.Clamp01((size - GetTotalMinSize(axis)) / (GetTotalPreferredSize(axis) - GetTotalMinSize(axis)));

        float itemFlexibleMultiplier = 0;
        if (size > GetTotalPreferredSize(axis))
        {
            if (GetTotalFlexibleSize(axis) > 0)
                itemFlexibleMultiplier = (size - GetTotalPreferredSize(axis)) / GetTotalFlexibleSize(axis);
        }

        for (int i = 0; i < rectChildren.Count; i++)
        {
            RectTransform child = rectChildren[i];
            float min = LayoutUtility.GetMinSize(child, axis);
            float preferred = LayoutUtility.GetPreferredSize(child, axis);
            float flexible = LayoutUtility.GetFlexibleSize(child, axis);
            if ((axis == 0 ? childForceExpandWidth : childForceExpandHeight))
                flexible = Mathf.Max(flexible, 1);

            float childSize = Mathf.Lerp(min, preferred, minMaxLerp);
            childSize += flexible * itemFlexibleMultiplier;
            SetChildAlongAxis(child, axis, pos, childSize);
            pos += childSize + spacing;
        }
    }
}
```

其中SetChildrenAlongAxis方法清晰地阐释了minHeight,preferredHeight,flexibleHeight的涵义。

为了帮助理解，我们先定义几个概念。我们把当前UI所有同级并参与自动布局的组件的preferredHeight总和称为totalPreferredHeight，minHeight的总和称为totalMinHeight，父UI的真实高度称为realHeight。总结如下：

1、**minHeight**
在自动布局中，此UI最小高度不会小于minHeight。这个参数定义了realHeight < totalMinHeight时，当前子UI的height为minHeight。

2、**preferredHeight**
可以理解为，UI自身希望的高度。
当totalMinHeight < realHeight  < totalPreferredHeight时，realHeight处于totalMinHeight和totalPreferredHeight之间一定百分比，把这个比例应用到每一个接受自动布局的子UI上，即是我们最终得到的效果

![1-3](/images/b63ec5b54e3cbcd3e42717dbcd468d4b701fb090.png)

3、**flexibleHeight**
当realHeight > totalPreferredHeight时，父UI会剩下一部分高度。flexibleHeight就是告诉AutoLayout系统，应该怎么瓜分剩下的高度，使子UI填充满父UI。flexibleHeight默认是-1，不会进行扩充。当flexibleHeight > 0时，flexibleHeight值作为权重来计算当前子UI最终的高度，公式如下：

>height = preferredHeight + (flexibleHeight / totalFlexibleHeight) * (realHeight - totalPreferredHeight)

![flexibleHeight示意图](/images/ccd93ef2ddc7b0e1bc7195b09cf19c115db9c635.png)


弄清楚这些概念后，我们再看一下文章开头的例子。
button1的flexibleHeight=0.3，button2的flexibleHeight=0.1，minHeight和preferredHeight都没有设置，按道理高度应该分别是75、25。为什么会出现70、30？

查一下ILayoutElement的实现类
![ILayoutElement实现类](/images/662905a043156155048c7bc0a31350a03da13331.png)

发现Image和Text实现了ILayoutElement，而我们的按钮中默认是有一个Image组件的，用脚本获取这个Image然后打印它的preferredHeight，发现等于10
再套用flexibleHeight的计算公式：
Button1:10 + 0.3/0.4 * (100 - 20) = 70
Button2:10 + 0.1/0.4 * (100 - 20) = 30

这里有个问题，一个GameObject上挂载两个ILayoutElement组件，是怎么决定用哪个的？这个可以在LayoutUtility.cs中找到答案：

LayoutUtility.cs

```
public static float GetLayoutProperty(RectTransform rect, System.Func<ILayoutElement, float> property, float defaultValue, out ILayoutElement source)
{
    source = null;
    if (rect == null)
        return 0;
    float min = defaultValue;
    int maxPriority = System.Int32.MinValue;
    var components = ComponentListPool.Get();
    rect.GetComponents(typeof(ILayoutElement), components);

    for (int i = 0; i < components.Count; i++)
    {
        var layoutComp = components[i] as ILayoutElement;
        if (layoutComp is Behaviour && (!(layoutComp as Behaviour).enabled || !(layoutComp as Behaviour).isActiveAndEnabled))
            continue;

        int priority = layoutComp.layoutPriority;
        // If this layout components has lower priority than a previously used, ignore it.
        if (priority < maxPriority)
            continue;
        float prop = property(layoutComp);
        // If this layout property is set to a negative value, it means it should be ignored.
        if (prop < 0)
            continue;

        // If this layout component has higher priority than all previous ones,
        // overwrite with this one's value.
        if (priority > maxPriority)
        {
            min = prop;
            maxPriority = priority;
            source = layoutComp;
        }
        // If the layout component has the same priority as a previously used,
        // use the largest of the values with the same priority.
        else if (prop > min)
        {
            min = prop;
            source = layoutComp;
        }
    }

    ComponentListPool.Release(components);
    return min;
}
```

原来LayoutElement有一个layoutPriority属性用来决定优先级，这个属性暂时还没有在编辑器中暴露，也许后续版本会加强这方面的能力。
AutoLayout系统会选用优先级最高的ILayoutElement里相应属性返回。Image和Text的优先级默认是0，LayoutElement默认优先级是1。所以正常情况会使用LayoutElement中的设置，但我们的例子中，LayoutElement没有设置preferredHeight，LayoutElement里布局相关的初始值都是-1，所以还是使用了Image的preferredHeight:10。

【结语】
其实，只要官方文档描述详细一些，根本没必要浪费时间去查这个来龙去脉。这几天在学习swift，苹果人性化的Programming Guide加上iReader的配合，使得学习这门语言真是件轻松愉快的事情。相比之下，Unity简直是在虐待开发者。Unity、Unreal、Cryengine等最近也为争市场弄得头破血流，除了降价开源提供新特性之外，完善文档、降低门槛也是不容忽视的工作。