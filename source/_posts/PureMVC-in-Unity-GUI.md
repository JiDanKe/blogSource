title: "在 Unity 的 GUI 中使用 PureMVC"
date: 2015-08-24 14:15:00
categories: Unity3D
tags: MVC
---

#PureMVC

PureMVC 是一个轻量级的应用框架，用于帮助开发者实现 MVC（模型、视图、控制器） 模式。PureMVC 框架最初是为 ActionScript 开发的，现在已经发展出了各种语言的版本，这其中也包括 C# 版本，这就意味可以把这个框架应用到 Unity 的游戏开发中。

##PureMVC 的基本组件

在 PureMVC 中提供了下面4种组件供开发者使用：

1. Proxy。Proxy 对应于 MVC 中的 M，但是它不是真正的 Model，在 PureMVC 中我们是通过 Proxy 对象去使用 Model，这也就保证了 Model 的独立性。
2. Meditor。Meditor 对应于 MVC 中的 V， 同样它也不是真正的 View，而是通过它来使用我们事先创建好的一些关于视图的一些接口。
3. Command。Cammand 就是 MVC 中的 C，在 Command 里我们可以调用 Proxy 去和 Model 进行交互，大多数业务逻辑都应该在 Cammand 里完成。Command 是无状态的，他和其他的 Command 和 Meditor 之间的通信可以通过通知（Notification）的方式来进行。
4. Facade。这可能是 PureMVC 中最重要的一个组件了，我们要通过它来初始化 Proxy、 Meditor、Command 以及绑定他们和对应的通知之间的关系。

#在 Unity 中使用 PureMVC

在网上的并没有找到比较好的 PureMVC C# 的教程，官方的文档也十分有限，所以就参考了一个 Java 版的教程做了下面的例子。在这个例子中主要关注 GUI ，而不是游戏逻辑。例子的内容很简单：点击按钮之后，在 Label 中显示 “Hello World!” 这句话。

<!-- more -->

##创建 View

上面已经说过了 PureMVC 中的 Meditor 并不是真正的 View，所以我们先要创建可以让 Meditor 使用的 API。

```cs
public class TestUIManager : MonoBehaviour {

	private static TestUIManager _instance;

	public Text showText;

	public static TestUIManager GetShareInstance()
	{
		if(_instance == null)
		{
			_instance = FindObjectOfType(typeof(TestUIManager)) as TestUIManager;
		}

		return _instance;
	}

	public void Click()
	{
		TestFacade.GetShareInstance().ClickTest();
	}

	public void ShowClick()
	{
		showText.text = "Hello World!";
	}
	
}
```

在这个例子里我们使用的原生的 Unity UI 构建了一个单例，这样 Meditor 可以很方便地使用这些接口。我们点击按钮时调用的是 `Click()`方法，这个方法并没有去直接让 Label 去显示 “Hello World!”，而是通过调用 `TestFacade` 中的`ClickTest()`方法去触发在 PureMVC 中实现的业务逻辑，这些业务逻辑会决定什么时候调用`ShowClick()`方法把"Hello World!"显示出来。下面就来介绍一下`TestFacade`。

##实现 Facade

`TestFacade`继承了`Facade`类，在其构造方法中调用了父类的构造方法也就是`Facade`的构造方法，这个方法会调用`InitializeModel()`、`InitializeController()`、 `InitializeView()`完成对 Proxy、Command 和 Meditor 的一些初始化工作，同时我们可以在`TestFacade`重写这些方法以实现一些自定义功能。

```cs
public class TestFacade : Facade
{
	public static string TEST_CLICK = "test_click";

	private static TestFacade _insance;

	public TestFacade():base(){}

	public static TestFacade GetShareInstance()
	{
		if(_insance == null)
		{
			_insance = new TestFacade();
		}

		return _insance;
	}

	protected override void InitializeController ()
	{
		base.InitializeController ();

		RegisterCommand(TEST_CLICK, typeof(TestClicKCommand));
	}

	protected override void InitializeView ()
	{
		base.InitializeView ();

		RegisterMediator(new TestUIMediator());
	}
	 
	public void ClickTest()
	{
		GetShareInstance().NotifyObservers(new Notification(TEST_CLICK, null, null));
	}
}
```

在`InitializeController()`中我们通过 RegisterCommand 方法把`TEST_CLICK`通知和`TestClickCommand`绑定起来，这样`TestClickCommand`就会响应由其他组件发出`TEST_CLICK`通知。同时在`InitializeView()`方法中初始化了一个 Meditor。在`ClickTest()`方法中我们就发出了一个`TEST_CLICK`通知希望得到相应的 Command 的响应。

##实现 Command

因为我们绑定了`TEST_CLICK`和`TestClickCommand`，所以`TestClickCommand`就会响应`TEST_CLICK`通知，现在就来看看`TestClickCommand`时怎么响应这个通知的。

```cs
public class TestClicKCommand : SimpleCommand{

	public override void Execute (PureMVC.Interfaces.INotification notification)
	{
		base.Execute (notification);

		TestUIMediator mediator = this.Facade.RetrieveMediator(TestUIMediator.NAME) as TestUIMediator;

		mediator.ClickHandler();
	}

}
```

这类首先继承`SimpleCommand`，然后重写了`Execute`方法，在收到相关的通知后就会调用`Execute`方法对通进行处理。因为在 TestFacade 中我们已经初始化了 `TestUIMediator`这个 Meditor 所以在这里我们可以使用 Facade 的`RetrieveMediator`方法获取到`TestUIMediator`的实例来对 UI 进行操作。最后就来看看`TestUIMediator`时怎么实现的。

##实现 Meditor

`TestUIMediator`继承了`Mediator` 类。

```cs
public class TestUIMediator : Mediator{

	public const string NAME = "TestUIMediator";

	public TestUIMediator():base(NAME){}

	public override IList<string> ListNotificationInterests ()
	{
		return base.ListNotificationInterests ();
	}

	public override void HandleNotification (PureMVC.Interfaces.INotification notification)
	{
		base.HandleNotification (notification);
	}

	public void ClickHandler()
	{
		TestUIManager.GetShareInstance().ShowClick();
	}
}
```

其中`ClickHandler()`方法可以供之前的`TestClicKCommand`直接调用来显示“Hello World!”。同时我们还可以重写`ListNotificationInterests`方法来定义这个`Mediator`所感兴趣的通知，这些通知往往来自于 Proxy，在接受到相关的通知之后我们就可以在`HandleNotification` 中处理这些通知。

#总结

至此我们就使用 PureMVC 完成了一个简单的 Demo，在这个 Demo 中我们并没有去关注 Proxy，但是 Proxy 使用的核心就是和 Model 交互，在完成后通过 Facade 发通知给相关的 Command 或 Meditor。

#参考资料

* [实战 PureMVC](http://www.ibm.com/developerworks/cn/java/j-lo-puremvc/)